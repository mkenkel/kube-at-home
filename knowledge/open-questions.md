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

* OPNsense Unbound override for `*.home.mkenkel.com` -> 192.168.15.100: done or not?
* Which TrueNAS dataset or share holds the local Longhorn/Velero backups? Deferred until apps are hosted; the `k8s` pool is the candidate, see [Backups](/decisions/backups.md). Blocks [Phase 4](/phases/phase-4-observability-security.md).
* Public Overseerr: needs a Cloudflare tunnel token for an in-cluster `cloudflared`, a Cloudflare Access policy, and a decision to move the existing `requests` hostname off the Docker tunnel. See [Exposure](/decisions/exposure-access.md) and the [migration runbook](/runbooks/media-migration.md).
* Docker `radarr` and `sabnzbd` hostnames on the existing tunnel: still wanted? Stopping those Docker apps takes them down.
* Is Portainer still planned for the cluster? The user removed it from the Docker stack on 2026-11-03; it is still in the Phase 5 administration list.
* Duplicati: what to back up and where to upload. Deferred until everything else is settled; see [Backups](/decisions/backups.md).

# Resolved

* Security hardening (2026-11-03): the small version. PSA `restricted` enforced on `cert-manager`, `gateway` and `flux-system`; network policies per app namespace in Phase 5; no cluster-wide default-deny. See [Phase 4](/phases/phase-4-observability-security.md).
* NFS media volume (2026-11-03): `192.168.15.24:/mnt/Poolio/media`, owner 568:568 through Mapall; see [Storage](/decisions/storage.md).
* Renovate is used for chart updates (`renovate.json`).
* Node IPs, control-plane VIP and LoadBalancer range: see [Cluster topology](/decisions/cluster-topology.md).
* Other decisions are in the `decisions/` directory.
