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

Items tagged **[ansible]** are run by `ansible/cluster.yml`. Order matters: nodes stay `NotReady` until Cilium is up, and Cilium needs the Gateway API CRDs first. Net-new cluster, nothing to reset.

- [x] [ansible] kube-vip static pod manifest on the first control plane (VIP from Phase 0)
- [x] [ansible] `kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint=<VIP>:6443 --upload-certs`; join 2 control planes and the worker
- [x] [ansible] Install pinned Gateway API CRDs by hand
- [x] [ansible] `helm install` Cilium from `cilium/values.yaml` (kube-proxy replacement, `k8sServiceHost`/`Port`, L2 announcements, Gateway API)
- [ ] [human] `flux bootstrap github`, then create the SOPS age secret in-cluster
- [x] [repo] Cilium `HelmRelease` with identical release name, namespace and values so Flux adopts it; Flux-owned Gateway API CRDs
- [x] [repo] Renovate config (`renovate.json`)
- [ ] [repo] CI on PRs: `kustomize build` and `kubeconform` exist in `.github/workflows/validate.yaml`; `flux diff` needs cluster access and is not set up

# Prepared files (2026-11-01)

The first four steps above are automated by `ansible/cluster.yml` (run 2026-11-01: all four nodes Ready on v1.36.6); Flux bootstrap and the SOPS secret stay manual. The runbook is `kubeadm/README.md`; the config is `kubeadm/kubeadm-config.yaml`. Pins checked 2026-11-01: Kubernetes v1.36.6, Cilium chart 1.20.4, Gateway API v1.6.3 (the version Cilium's docs list; v1.7.0 is newer and not used), kube-vip v1.3.1. kube-vip runs without `--services` because Cilium L2 announcements own LoadBalancer IPs. Pod CIDR 10.244.0.0/16 is set in both the kubeadm config and `cilium/values.yaml`.

Flux-side files: `clusters/home/{gateway-api-crds,infrastructure,apps}.yaml` and `infrastructure/controllers/cilium/`. `spec.values` in the HelmRelease duplicates `cilium/values.yaml` and must stay identical (kustomize cannot load files outside its directory).

Confirmed by the first run: kubeadm accepted `--skip-phases` and `--upload-certs` with `--config`, the certificate key and joins worked, and nodes joined through the VIP. Still unverified: that the HelmRelease adopts the hand-installed release with no diff; the flux-system `GitRepository`/bootstrap flow, since nothing has run against the cluster.

Decisions: [GitOps](/decisions/gitops-flux.md), [topology](/decisions/cluster-topology.md), [networking](/decisions/networking.md).
