---
type: Open Questions
title: Open questions
description: Unresolved decisions that block or shape later phases.
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, section 6
---

# Open

* Security hardening: small version (PSA labels now, network policies per app namespace in Phase 5) or skip until Phase 5; see [Phase 4](/phases/phase-4-observability-security.md).
* OPNsense Unbound override for `*.home.mkenkel.com` -> 192.168.15.100: done or not?
* Which TrueNAS dataset or share holds the local Longhorn/Velero backups? Deferred until apps are hosted; the `k8s` pool is the candidate, see [Backups](/decisions/backups.md). Blocks [Phase 4](/phases/phase-4-observability-security.md).
* Duplicati: what to back up and where to upload. Deferred until everything else is settled; see [Backups](/decisions/backups.md).

# Resolved

* NFS media volume (2026-11-03): `192.168.15.24:/mnt/Poolio/media`, owner 568:568 through Mapall; see [Storage](/decisions/storage.md).
* Renovate is used for chart updates (`renovate.json`).
* Node IPs, control-plane VIP and LoadBalancer range: see [Cluster topology](/decisions/cluster-topology.md).
* Other decisions are in the `decisions/` directory.
