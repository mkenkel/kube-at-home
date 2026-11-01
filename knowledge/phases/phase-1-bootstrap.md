---
type: Deployment Phase
title: "Phase 1: Cluster init and GitOps bootstrap"
description: kubeadm init with kube-vip, Gateway API CRDs and Cilium by hand, then Flux bootstrap and adoption.
tags: [phase, kubeadm, cilium, flux]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 1
---

Order matters: nodes stay `NotReady` until Cilium is up, and Cilium needs the Gateway API CRDs first. Net-new cluster, nothing to reset.

- [ ] [human] kube-vip static pod manifest on the first control plane (VIP from Phase 0)
- [ ] [human] `kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint=<VIP>:6443 --upload-certs`; join 2 control planes and the worker
- [ ] [human] Install pinned Gateway API CRDs by hand
- [ ] [human] `helm install` Cilium from `cilium/values.yaml` (kube-proxy replacement, `k8sServiceHost`/`Port`, L2 announcements, Gateway API)
- [ ] [human] `flux bootstrap github`, then create the SOPS age secret in-cluster
- [ ] [repo] Cilium `HelmRelease` with identical release name, namespace and values so Flux adopts it; Flux-owned Gateway API CRDs
- [ ] [repo] Renovate config (or Dependabot)
- [ ] [repo] CI on PRs: `kustomize build`, `kubeconform`, `flux diff`

Decisions: [GitOps](/decisions/gitops-flux.md), [topology](/decisions/cluster-topology.md), [networking](/decisions/networking.md).
