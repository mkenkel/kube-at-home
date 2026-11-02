---
type: Deployment Phase
title: "Phase 4: Observability and security"
description: kube-prometheus-stack, Loki + Alloy, network policies, pod security, local backups.
tags: [phase, monitoring, logging, security, backup]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 4
---

- [x] [repo] Applied 2026-11-01 (all pods Running, PVCs Bound, HTTPRoute Accepted, Grafana returns 200 through the Gateway with a verified production Let's Encrypt cert; tested with `curl --resolve`, since the OPNsense wildcard override was not yet in place): `kube-prometheus-stack` 92.17.0 in `monitoring` (Prometheus 30Gi/15d and Grafana 5Gi on Longhorn, Grafana at `https://grafana.home.mkenkel.com` through the Gateway, Prometheus and Alertmanager not exposed). Grafana admin password is in the SOPS secret `grafana-admin`; read it with `sops -d --extract '["stringData"]["admin-password"]' infrastructure/controllers/monitoring/grafana-admin.sops.yaml`.
- [x] [human] Expose etcd, scheduler and controller-manager metrics: `ansible-playbook control-plane-metrics.yml`, run 2026-11-01 on node-1..3 with no failures; listeners on `*:2381`, `*:10257`, `*:10259`, and from node-1 etcd answers 200 and the other two 403 (token required) on all three node IPs. Edits the live static pod manifests one control plane at a time (`serial: 1`) and re-uploads `kubeadm/kubeadm-config.yaml` to the `kubeadm-config` ConfigMap so a later `kubeadm upgrade` keeps it. The HelmRelease already scrapes all three (Prometheus targets are down until the playbook runs). etcd metrics are plain HTTP on :2381 and unauthenticated on the LAN. Not yet checked: the `kubeadm-config` ConfigMap contents after the upload, and the Prometheus targets (they turn on once the HelmRelease change is pushed).
- [ ] [repo] ServiceMonitors for Cilium/Hubble and Longhorn
- [ ] [repo] Loki + Grafana Alloy (Promtail is deprecated)
- [ ] [repo] Cilium NetworkPolicies; Pod Security Admission labels (`baseline`/`restricted`)
- [ ] [repo] Longhorn recurring backups and Velero. Blocked on the TrueNAS dataset decision in [open questions](/open-questions.md).
- [ ] [human] First test restore, then quarterly
- [ ] Offsite Duplicati: deferred, see [backups](/decisions/backups.md)
