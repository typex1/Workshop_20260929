# Argo CD vs. Flux

Both are **GitOps continuous delivery (CD)** tools for Kubernetes, they watch a Git repo (or OCI registry / Helm repo) that holds the desired cluster state and reconcile the cluster to match it. CI (GitHub Actions, GitLab CI, CodeBuild, Jenkins, ...) builds the image and updates the manifest in Git; Argo CD or Flux then deploys it. Both are CNCF graduated projects.

## At a glance

| | Argo CD | Flux (v2) |
|---|---|---|
| Project origin | Intuit, 2018 | Weaveworks, 2016 (v2 rewrite 2020) |
| Core unit | `Application` CRD (source + destination) | Set of CRDs: `GitRepository`, `OCIRepository`, `HelmRepository`, `Kustomization`, `HelmRelease` |
| Architecture | Central controller + API server + repo server, usually one install per cluster or hub-and-spoke | Modular controllers (source, kustomize, helm, notification, image-automation); install only what you need |
| Web UI | Built in, full featured (sync status, diff, resource tree, rollback, SSO) | None upstream; use Weave GitOps (archived), Headlamp Flux plugin, or Capacitor |
| CLI | `argocd` (talks to the API server, needs login) | `flux` (talks to the Kubernetes API, uses kubeconfig) |
| Helm handling | Renders `helm template` and applies plain manifests; no Helm release history in cluster | Native `helm install/upgrade` via HelmRelease; releases visible to `helm list`, supports rollback, tests, drift detection |
| Kustomize | Yes | Yes (first-class, with post-build variable substitution) |
| Multi-tenancy / RBAC | Own RBAC model (Projects, roles, SSO groups) layered on top of Kubernetes RBAC | Pure Kubernetes RBAC and namespaces, impersonation via `serviceAccountName` |
| Multi-cluster | Hub-and-spoke from one control plane, ApplicationSet generators for fleets | One Flux install per cluster (or remote `kubeConfig` ref); fleets managed via Git structure |
| Image update automation | Separate project: Argo CD Image Updater | Built in: image-reflector + image-automation controllers commit new tags back to Git |
| Progressive delivery | Argo Rollouts (canary, blue/green, analysis) | Flagger (canary, A/B, blue/green; works with Argo CD too) |
| Sync model | Pull, with optional webhook trigger; sync waves and hooks (PreSync/PostSync) | Pull, with optional webhook receiver; `dependsOn` between Kustomizations, health checks |
| Secrets | External tools (Sealed Secrets, ESO, Vault plugin) | Native SOPS decryption, plus the same external tools |
| Learning curve | Lower for day-1 thanks to the UI | Lower for people who already think in Kubernetes CRDs and `kubectl` |

## Choosing

Pick **Argo CD** when:
- A visual UI for developers and operators is a hard requirement (sync status, diffs, click-to-rollback).
- You want one central control plane managing many clusters, with its own SSO-backed RBAC for teams that are not Kubernetes experts.
- You plan to use Argo Rollouts and want the full Argo stack (Workflows, Events, Rollouts) in one ecosystem.

Pick **Flux** when:
- You prefer a lightweight, composable, Kubernetes-native toolkit with no extra API server or RBAC model to operate.
- Native Helm release semantics (`helm list`, rollback, hooks) matter.
- You want built-in image tag automation and SOPS secret decryption without extra projects.
- You use EKS Auto Mode / minimal clusters and want the smallest footprint.

Both are also available on EKS via `eksctl` add-ons, Helm charts, and the AWS Marketplace; neither is an EKS-managed add-on.

## One sentence each

- **Argo CD**: an opinionated GitOps platform with a UI at its center.
- **Flux**: a GitOps toolkit of controllers that you assemble.
