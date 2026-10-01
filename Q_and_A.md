## Question

Regarding the vertical pod autoscaler, is that capable to modify the value for "request cpu" of a specific container inside a pod? For example, to squeeze a pod onto a node which otherwise would have been too large?

Yes, with caveats. VPA works per container, not per pod, and it sets resources.requests (CPU and/or memory) on individual containers. In the VPA object you scope that with containerPolicies:

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: Auto          # Off | Initial | Recreate | Auto | InPlaceOrRecreate
  resourcePolicy:
    containerPolicies:
      - containerName: app
        controlledResources: ["cpu"]      # only touch CPU
        controlledValues: RequestsOnly    # or RequestsAndLimits
        minAllowed:
          cpu: 100m
        maxAllowed:
          cpu: "2"
      - containerName: sidecar
        mode: "Off"                       # leave this container alone

```

So the mechanics you asked about exist: it can lower requests.cpu of one specific container and leave the others untouched, and if a smaller request is the result, the pod will then fit onto a node it did not fit on before.

The caveat is the why. VPA does not look at nodes or at scheduling failures. The recommender looks at the container's historical usage (from metrics-server, roughly an 8-day decaying histogram) and derives target/lower/upper bounds. It lowers the request only because the container has been using less than requested, never because "there is a node with 1.5 vCPU free". If the container genuinely needs 2 vCPU, VPA will keep (or raise) the request and the pod stays unschedulable; the only cap it applies is the largest node's allocatable, so it won't recommend something no node could ever hold.

---

## Question

When a node group of an EKS cluster gets a notification from the cluster autoscaler to scale in, which rule exists to pick a specific node from the EC2 autoscaling group (ASG)?

**Short answer:** there is no ASG rule involved. The Cluster Autoscaler picks the node itself and tells the ASG exactly which instance to terminate.

How it actually works:

CA evaluates every node in the node group. A node becomes a scale-down candidate when its requested CPU/memory stays below --scale-down-utilization-threshold (default 50%) for --scale-down-unneeded-time (default 10 min), and all its pods could be rescheduled elsewhere. Nodes are excluded if they run pods that block eviction: non-replicated pods (not backed by a controller), pods with local storage, kube-system pods without a PDB, pods annotated cluster-autoscaler.kubernetes.io/safe-to-evict: "false", or the node itself carries cluster-autoscaler.kubernetes.io/scale-down-disabled: "true".

From the candidates, CA removes empty nodes in bulk (up to --max-empty-bulk-delete, default 10) and non-empty nodes one at a time: cordon, evict pods respecting PDBs, wait for pods to go.

Then it calls the EC2 Auto Scaling API TerminateInstanceInAutoScalingGroup with that specific instance ID and ShouldDecrementDesiredCapacity=true. Because the instance is named explicitly, the ASG's termination policy never runs.

---

# EKS Q&A — Scheduling Conflicts

## Question

A pod needs to be scheduled, and its **node affinity** is enforcing it to be scheduled on a
specific EC2 instance type. But it happens that the **namespace** the pod is in has a matching
**Fargate profile**, telling that the pod needs to run on Fargate.

**So where will the pod finally be placed?**

---

## Short Answer

> **The pod will NOT be scheduled at all. It stays `Pending`.**

It is neither forced onto EC2 nor onto Fargate — the two requirements contradict each other,
so no node can satisfy them.

---

## Why? Step by Step

### 1. Fargate profile selection happens *first*, at admission time

On EKS, a **Fargate profile** matches pods by **namespace** (and optionally by labels/selectors).
This decision is made *before* the scheduler ever runs.

When a pod is created in a namespace that matches a Fargate profile, the EKS Fargate
**mutating admission webhook** intercepts the pod and does two things:

1. Sets the pod's `schedulerName` to `fargate-scheduler` (instead of `default-scheduler`).
2. Injects a node affinity that binds the pod to a Fargate virtual node, labeled:
   ```
   eks.amazonaws.com/compute-type: fargate
   ```

> **Key point:** The "this pod belongs to Fargate" decision is based purely on the
> **namespace match** — not on a scheduler-time comparison against your affinity rules.

### 2. Your node affinity is *not* removed — it is *added to*

Your pod already carries a node affinity like:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node.kubernetes.io/instance-type
              operator: In
              values:
                - m5.large        # a specific EC2 instance type
```

The Fargate webhook does **not** delete this. It adds its own affinity on top.

Multiple `requiredDuringSchedulingIgnoredDuringExecution` requirements are **ANDed together**.

### 3. The contradiction

The pod now effectively requires **both** of the following at the same time:

| Requirement (injected by Fargate) | Requirement (your affinity) |
| --------------------------------- | --------------------------- |
| `compute-type = fargate`          | `instance-type = m5.large`  |

No single node can satisfy both:

- A **Fargate** virtual node has `compute-type: fargate`, but has **no** EC2 `instance-type`
  label (it is not an EC2 instance).
- An **EC2** node has the `instance-type` label, but does **not** have `compute-type: fargate`.

On top of that, the pod is now owned by `fargate-scheduler`, so:

- The **default scheduler** ignores it (not its pod).
- The **Fargate scheduler** can only place it on Fargate — which your EC2 affinity forbids.

### 4. Result

```
$ kubectl get pod my-pod
NAME      READY   STATUS    RESTARTS   AGE
my-pod    0/1     Pending   0          3m
```

You will see a `FailedScheduling` event, and **no Fargate capacity gets provisioned**.
The pod stays `Pending` indefinitely.

---

## Key Takeaways

- **Fargate matching is by namespace** (and optional labels), decided at **admission time** —
  it is *not* a scheduler tie-break against your affinity.
- **Your node affinity is not overridden.** It is combined (ANDed) with the injected Fargate
  affinity, producing an **unsatisfiable** requirement.
- The conflict does not "pick a winner" — it makes the pod **unschedulable**.

---

## How to Avoid This Conflict

Pick **one** compute type per namespace, then choose one of these fixes:

1. **Don't add EC2-targeting node affinity** to pods in a Fargate-matched namespace.
2. **Use a different namespace** (or labels) so the pod does *not* match the Fargate profile —
   this lets the default scheduler honor your EC2 affinity.
3. **Adjust the Fargate profile's selectors** so this pod no longer matches, if you genuinely
   want it on EC2.

---

## Quick Reference

```
Fargate profile match (by namespace/labels)
        │
        ▼
Admission webhook rewrites the pod:
   • schedulerName        → fargate-scheduler
   • + nodeAffinity        compute-type=fargate
        │
        ▼
Pod now requires:  compute-type=fargate  AND  instance-type=<EC2 type>
        │
        ▼
   No node matches both  ──►  Pod stays Pending
```
