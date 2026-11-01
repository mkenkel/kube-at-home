# kube-at-home

GitOps repo for a home Kubernetes cluster: 4x Lenovo M70q Gen2 (Debian, kubeadm + containerd), Flux, Cilium, Longhorn, TrueNAS NFS for media.

## Layout

| Path | Purpose |
|---|---|
| `clusters/home/` | Flux entrypoint: `flux-system/`, `infrastructure.yaml`, `apps.yaml` |
| `infrastructure/controllers/`, `infrastructure/configs/` | Cluster services and their config |
| `apps/base/`, `apps/home/` | App definitions and per-category overlays |
| `cilium/values.yaml` | Cilium Helm values, installed by hand in Phase 1 |
| `.sops.yaml` | SOPS + age rules for encrypted secrets |

`ansible/` and `preseed/` (node provisioning) are added separately.

## Nodes

192.168.15.0/24: kube-node-1..3 (control planes) at .7-.9, kube-node-4 (worker) at .10, API VIP .6, Cilium LB pool .100-.200.

## Secrets

Secrets are SOPS-encrypted with age. The private key lives in the password manager and is never committed. Before encrypting anything, replace the placeholder recipient in `.sops.yaml`.

## Docs

Plans are in `plan.md` and `deployment-plan.md`; decisions and per-phase checklists are in `knowledge/` (start at `knowledge/index.md`). Changes reach the cluster through Git and Flux.
