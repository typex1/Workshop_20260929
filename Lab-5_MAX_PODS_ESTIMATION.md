# Max Replica Estimation — `deployment/prodcatalog` (namespace `workshop`)

_Analysis date: 2026-10-01_

## Question
How many pod replicas of `deployment/prodcatalog` (namespace `workshop`) can run on the cluster at maximum?

## Answer
**~31 additional replicas → ~33 total** before new pods go `Pending`.

The binding constraint is the **max-pods-per-node (IP/ENI-based) limit**, not CPU or memory.

---

## Cluster Under Test
- **Cluster:** `dev-cluster` (eu-west-2), configured via local kubeconfig
- **Nodes:** 3 × `t3.medium` worker nodes
- **Deployment:** `prodcatalog`, currently 2 replicas, **1 container, no resource requests/limits (`resources: {}`)**

---

## Key Finding — the limit is IP/ENI-based (max-pods)

The AWS VPC CNI assigns each pod a secondary IP from the node's ENIs. The max-pods value is:

```
max-pods = (ENIs × (IPv4 per ENI − 1)) + 2
```

For a **t3.medium**: `3 ENIs × (6 − 1) = 15`, `+ 2` host-networked pods = **17**.
This matches the observed `status.capacity.pods = 17` / `allocatable.pods = 17` on every node.

Because `prodcatalog` requests **zero** CPU and memory, the scheduler only needs a free **pod slot** (IP) on a node — CPU and memory never reject it.

---

## Capacity Snapshot

### Pod slots (the binding constraint)

| Node | Pod capacity | Pods in use | Free slots |
|------|-------------|-------------|------------|
| ip-10-10-10-252 | 17 | 7 | 10 |
| ip-10-10-20-109 | 17 | 7 | 10 |
| ip-10-10-20-90  | 17 | 6 | 11 |
| **Total** | **51** | **20** | **31** |

> In-use slots include system DaemonSets (e.g. `aws-node`, `kube-proxy`) that occupy a slot on every node.

### CPU / Memory (NOT binding)

| Resource | Committed / node | Allocatable / node | Headroom |
|----------|------------------|--------------------|----------|
| CPU | 350–450m | 1930m | ~1.5 cores free |
| Memory | ~140–712Mi | ~3.2Gi | plenty |

`prodcatalog` adds 0 to committed requests, so these never become the limiting factor.

---

## What Limits Replica Count (ranked)
1. **Primary: max-pods = 17/node (IP-based)** → caps at ~31 more replicas.
2. CPU — not binding (requests are 0).
3. Memory — not binding at *scheduling* time (requests are 0).

---

## Caveats
- This is a **scheduling** capacity estimate, not a **performance** one. Running ~33 replicas with **no resource requests** risks real CPU/memory contention and kubelet eviction under pressure. Set realistic CPU/memory requests for a meaningful "run well" number.
- The number shifts as other workloads scale up/down (the 20 in-use slots are not all static).
- If Cluster Autoscaler / Karpenter is enabled, new nodes could raise this ceiling (not yet verified).

---

## Ways to Raise the IP Ceiling
- **Enable VPC CNI prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`): assigns /28 prefixes instead of single IPs; raises t3.medium to ~110 pods/node.
- **Use larger instance types** (more ENIs / IPs per node).
- **Add nodes** (manually or via Cluster Autoscaler / Karpenter).

---

---

## Add-on & Autoscaling State (verified 2026-10-01)

### Prefix delegation: **NOT enabled**
VPC CNI DaemonSet `aws-node` (image `amazon-k8s-cni:v1.22.4-eksbuild.3`):

```
ENABLE_PREFIX_DELEGATION          = false
WARM_PREFIX_TARGET                = 1      # inert while prefix delegation is off
WARM_ENI_TARGET                   = 1
AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG= false
```

Running in default single-IP mode → the **17 pods/node** cap on t3.medium is in full effect.

### Autoscaling: **none**
- Cluster Autoscaler: not found
- Karpenter: not found

No controller adds nodes. However, the managed nodegroup `dev-nodes` has
`scalingConfig = {min:2, desired:3, max:4}` — so the group *could* be manually
scaled to **4 nodes** (one more node ≈ +17 pod slots), but it will not scale on its own.

---

## Permission Check — Can I enable prefix delegation?

**Yes — all required permissions are in place.**

| Layer | Needed for | Result |
|-------|-----------|--------|
| K8s RBAC | `patch/update daemonsets.apps` in kube-system (to set the env var) | ✅ yes (`can-i '*' '*'` → yes) |
| K8s RBAC | `delete nodes` (to recycle so change takes effect) | ✅ yes |
| IAM (VPC CNI IRSA role `eksctl-dev-cluster-addon-vpc-cni-Role1-*`) | `ec2:AssignPrivateIpAddresses` for /28 prefix assignment | ✅ has `AmazonEKS_CNI_Policy` |

Notes:
- The VPC CNI authenticates via **IRSA** (service account `aws-node` → role `eksctl-dev-cluster-addon-vpc-cni-Role1-FNumZhTM20bQ`), **not** the node instance role. That IRSA role carries `AmazonEKS_CNI_Policy`, which grants the prefix-assignment EC2 permissions.
- `iam:SimulatePrincipalPolicy` is blocked by an SCP, so permissions were verified by directly inspecting attached policies rather than via the policy simulator.

---

## Projection: Pods With Prefix Delegation Enabled (t3.medium)

**Hardware facts (t3.medium):** 3 ENIs max, 6 IPv4 per ENI, 2 vCPU.

**Current (single-IP) formula:**
```
max-pods = (ENIs × (IPsPerENI − 1)) + 2 = 3 × (6 − 1) + 2 = 17
```

**With prefix delegation** (each ENI slot = one /28 prefix = 16 IPs):
```
raw = (ENIs × ((IPsPerENI − 1) × 16)) + 2 = 3 × (5 × 16) + 2 = 242
```

**EKS hard cap:** instance types with < 30 vCPUs are capped at **max-pods = 110**
(AWS stability ceiling). t3.medium has 2 vCPU, so:
```
effective max-pods = min(242, 110) = 110
```

### Before vs. after

| Scenario | max-pods / node | 3 nodes total | In use | Free pod slots |
|----------|-----------------|---------------|--------|----------------|
| **Now** (single-IP) | 17 | 51 | 20 | **31** |
| **Prefix delegation** | 110 | 330 | 20 | **~310** |

- Per node: **17 → 110** (+93/node)
- Cluster free slots: **31 → ~310**
- For `prodcatalog` (zero resource requests): ~**310** schedulable replicas vs ~31 today
  — roughly **10× more**, i.e. **~279 additional** beyond today's ceiling.

### Reality checks
1. **You get 110, not 242** — the EKS < 30-vCPU cap applies. You MUST set kubelet
   `--max-pods=110` on the nodegroup; otherwise the extra IPs are unused and it stays at 17.
2. **This is a scheduling number, not performance.** 110 request-less pods on a
   2-vCPU / ~3.9 GB t3.medium is heavily oversubscribed — CPU/memory contention and
   eviction would hit well before 110. The 110 cap exists precisely as a safety ceiling.
3. **Memory becomes the real bottleneck** at that density (~3.2 GiB allocatable/node).
   Set realistic resource requests on `prodcatalog` for a trustworthy "runs *well*"
   number — it would land far below 110/node.

---

### To actually enable it (medium-risk, shared add-on)
```bash
# 1. Turn on prefix delegation
kubectl set env ds aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# 2. Raise kubelet --max-pods on the nodegroup (otherwise still capped at 17).
#    For managed nodegroups this requires a new launch template / nodegroup
#    with --max-pods bootstrap arg (e.g. up to ~110 for t3.medium).

# 3. Recycle nodes so new ENIs use /28 prefixes and new max-pods applies
#    (existing nodes keep their current IP mode until replaced).
```
Caveats:
- Only affects **newly launched nodes / newly assigned ENIs**; existing nodes must be replaced.
- `--max-pods` must be raised in tandem or the IP gain is wasted.
- t3.medium (Nitro) supports prefix assignment.

---

## How This Was Measured
- `kubectl config current-context` → identified `dev-cluster` (eu-west-2).
- `kubectl get deployment prodcatalog -n workshop -o jsonpath='{...resources}'` → confirmed empty `resources: {}` and 2 replicas.
- `kubectl get nodes` custom-columns → instance type `t3.medium`, `capacity.pods`/`allocatable.pods` = 17.
- `kubectl get pods -A --field-selector=status.phase!=Succeeded,status.phase!=Failed` → counted non-terminated pods per node and summed committed CPU/memory requests.
