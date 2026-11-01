---
okf_version: "0.2"
---

# kube-at-home knowledge bundle

OKF v0.2 bundle for the home Kubernetes cluster (4x M70q Gen2, kubeadm). Narrative source docs are `/deployment-plan.md` and `/plan.md` at the repo root; this bundle holds the decisions and phase state in a form agents can query and update.

# Decisions

* [GitOps with Flux](decisions/gitops-flux.md) - Flux over ArgoCD, SOPS + age secrets, repo layout
* [Cluster topology](decisions/cluster-topology.md) - 3 untainted control planes + 1 worker, kube-vip VIP
* [Networking](decisions/networking.md) - Cilium, L2 announcements now, BGP deferred, Gateway API
* [Storage](decisions/storage.md) - Longhorn for configs/DBs, TrueNAS NFS for media
* [Exposure and access](decisions/exposure-access.md) - LAN/WireGuard default, Cloudflare Tunnel for two apps
* [Backups](decisions/backups.md) - Local Longhorn/Velero to TrueNAS, offsite Duplicati deferred

# Phases

* [Phase 0: Node provisioning](phases/phase-0-provisioning.md) - preseed, Ansible, IPs, age key, repo skeleton
* [Phase 1: Cluster init and GitOps bootstrap](phases/phase-1-bootstrap.md) - kubeadm, Cilium by hand, Flux
* [Phase 2: Networking](phases/phase-2-networking.md) - L2 pool, Gateway API, cert-manager
* [Phase 3: Storage](phases/phase-3-storage.md) - Longhorn, NFS media volume
* [Phase 4: Observability and security](phases/phase-4-observability-security.md) - Prometheus, Loki, policies, backups
* [Phase 5: Applications](phases/phase-5-applications.md) - category-by-category rollout

# Other

* [Open questions](open-questions.md) - unresolved items blocking or shaping later phases
* [Update log](log.md)
