---
type: Open Questions
title: Open questions
description: Unresolved decisions that block or shape later phases.
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, section 6
---

# Open

* NFS media volume: TrueNAS address, export path, UID/GID for the *arr stack. Blocks [Phase 3](/phases/phase-3-storage.md).
* Security hardening: small version (PSA labels now, network policies per app namespace in Phase 5) or skip until Phase 5; see [Phase 4](/phases/phase-4-observability-security.md).
* OPNsense Unbound override for `*.home.mkenkel.com` -> 192.168.15.100: done or not?
* Which TrueNAS dataset or share holds the local Longhorn/Velero backups? Blocks [Phase 4](/phases/phase-4-observability-security.md).
* Duplicati: what to back up and where to upload. Deferred until everything else is settled; see [Backups](/decisions/backups.md).

# Resolved

* Renovate is used for chart updates (`renovate.json`).
* Node IPs, control-plane VIP and LoadBalancer range: see [Cluster topology](/decisions/cluster-topology.md).
* Other decisions are in the `decisions/` directory.
