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

- [ ] [repo] `kube-prometheus-stack`
- [ ] [repo] Loki + Grafana Alloy (Promtail is deprecated)
- [ ] [repo] Cilium NetworkPolicies; Pod Security Admission labels (`baseline`/`restricted`)
- [ ] [repo] Longhorn recurring backups and Velero. Blocked on the TrueNAS dataset decision in [open questions](/open-questions.md).
- [ ] [human] First test restore, then quarterly
- [ ] Offsite Duplicati: deferred, see [backups](/decisions/backups.md)
