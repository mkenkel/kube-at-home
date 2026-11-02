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

- [ ] [repo] Prepared 2026-11-01, not yet applied: `kube-prometheus-stack` 92.17.0 in `monitoring` (Prometheus 30Gi/15d and Grafana 5Gi on Longhorn, Grafana at `https://grafana.home.mkenkel.com` through the Gateway, Prometheus and Alertmanager not exposed). Grafana admin password is in the SOPS secret `grafana-admin`; read it with `sops -d --extract '["stringData"]["admin-password"]' infrastructure/controllers/monitoring/grafana-admin.sops.yaml`.
- [ ] [repo/human] Expose etcd, scheduler and controller-manager metrics (kubeadm binds them to 127.0.0.1; their scrapes are disabled for now). Needs `extraArgs` in `kubeadm/kubeadm-config.yaml` and edits to the static pod manifests on the live control planes, one at a time. etcd matters most because it shares its disk with Longhorn.
- [ ] [repo] ServiceMonitors for Cilium/Hubble and Longhorn
- [ ] [repo] Loki + Grafana Alloy (Promtail is deprecated)
- [ ] [repo] Cilium NetworkPolicies; Pod Security Admission labels (`baseline`/`restricted`)
- [ ] [repo] Longhorn recurring backups and Velero. Blocked on the TrueNAS dataset decision in [open questions](/open-questions.md).
- [ ] [human] First test restore, then quarterly
- [ ] Offsite Duplicati: deferred, see [backups](/decisions/backups.md)
