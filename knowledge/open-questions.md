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
* Public Seerr (`requests`): wanted, to be served by an in-cluster `cloudflared`. Needs a new Cloudflare tunnel token as a SOPS secret and a Cloudflare Access policy, then moving the hostname off the Docker tunnel. Radarr and Sabnzbd are not public: the user removes their old hostnames in Cloudflare. See [Exposure](/decisions/exposure-access.md) and the [migration runbook](/runbooks/media-migration.md).
* Duplicati: what to back up and where to upload. Deferred until everything else is settled; see [Backups](/decisions/backups.md).

# Resolved

* Backups (2026-11-03): Longhorn recurring backups of the config volumes to NFS dataset `k8s/longhorn-backups` on TrueNAS; Velero dropped. See [Backups](/decisions/backups.md) and the [runbook](/runbooks/backup-restore.md).
* Media apps (2026-11-03): the app set is what the user runs (Sonarr, Radarr, Sabnzbd, Prowlarr, Overseerr, now Seerr), existing config migrated, one `/media` mount, only Seerr public. See [Phase 5](/phases/phase-5-applications.md).
* Portainer (2026-11-03): dropped; removed from the plan and the Phase 5 list.
* Security hardening (2026-11-03): the small version. PSA `restricted` enforced on `cert-manager`, `gateway` and `flux-system`; network policies per app namespace in Phase 5; no cluster-wide default-deny. See [Phase 4](/phases/phase-4-observability-security.md).
* NFS media volume (2026-11-03): `192.168.15.24:/mnt/Poolio/media`, owner 568:568 through Mapall; see [Storage](/decisions/storage.md).
* Renovate is used for chart updates (`renovate.json`).
* Node IPs, control-plane VIP and LoadBalancer range: see [Cluster topology](/decisions/cluster-topology.md).
* Other decisions are in the `decisions/` directory.
