---
type: Deployment Phase
title: "Phase 5: Applications"
description: One category per PR, lowest risk first, using a shared app-template chart.
tags: [phase, apps]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 5 and section 4
  - id: plan
    resource: /plan.md
    title: App list
---

Use the bjw-s `app-template` chart for the one-container/one-PVC/one-ingress shape. One PR per category, in order:

1. Administration: Glance, Gitea, then Duplicati last (needs its backup plan). Portainer is dropped (2026-11-03).
2. Notetaking: nothing deployed here.
3. Media: Jackett, Sonarr, Radarr, Lidarr, Sabnzbd, Jellyseerr, Wizarr, Notifiarr, Tautulli. Shared TrueNAS media volume; the media server box must be reachable from the cluster. Only Overseerr/Seerr (`requests`) is public ([exposure](/decisions/exposure-access.md)).
4. File management: nothing deployed here.
5. Home automation: nothing deployed here.

# Media (first category)

- [x] [repo] Applied 2026-11-03 (PR #4; all five pods Running, routes Accepted, `media-isolation` tested from the Sonarr pod): Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr as bjw-s `app-template` 5.3.0 HelmReleases in `apps/base/<app>/`, assembled in `apps/home/media/` (namespace `media`, PSA `baseline` with `restricted` warnings; HTTPRoutes at `<app>.home.mkenkel.com`; Longhorn config volumes that are kept on uninstall; Sonarr, Radarr and Sabnzbd mount the `media` claim at `/media`; Cilium network policy `media-isolation`). Rendered with `helm template` and checked with `kustomize build`, `yamllint --strict` and `kubeconform` (HTTPRoutes are skipped by the schema catalog).
- [x] [human] Done 2026-11-03: Prowlarr, Sonarr, Radarr, Sabnzbd and Overseerr moved from the Docker stack with their existing data (Sonarr 25 series / 975 files, Radarr 92 movies, Overseerr 122 requests, all counts matched). The Docker copies of the first four are stopped and not removed; the Docker Overseerr still runs so the public `requests` hostname keeps loading. Method, pitfalls found and rollback are in the [migration runbook](/runbooks/media-migration.md).
- [ ] [repo] Overseerr is archived (last release 1.35.0, 2026-02-15), so the `overseerr` release runs its successor Seerr v3.5.1 (`ghcr.io/seerr-team/seerr`), which migrates the existing config and database in place on first start (one-way). Preparation done first: config backed up and the volume chowned to 1000:1000. Verify after the rollout: 122 requests, 1 user, Sonarr, Radarr and Plex connection tests pass.
- [ ] [repo] In-cluster `cloudflared` (2 replicas, restricted PSA, egress-only policy) staged in `apps/home/cloudflared/`, not yet included in `apps/home/kustomization.yaml` because its pods need the `cloudflared-token` secret first. `media-isolation` already allows it in on port 5055 only.
- [ ] [human] Create the tunnel and its SOPS secret, add Cloudflare Access policies, then move the `requests` hostname: see the public hostnames section of the [migration runbook](/runbooks/media-migration.md).

Decisions for this category (2026-11-03): the app set is what the user runs today (Prowlarr instead of the plan's Jackett, Overseerr instead of Jellyseerr, now running Seerr; Lidarr, Notifiarr, Tautulli and Wizarr are not included); existing config is migrated, not rebuilt; library and downloads share one `/media` mount so imports are renames; Sabnzbd keeps incomplete downloads on the NFS volume to stay off the etcd disk; Only Overseerr stays public, through a new in-cluster tunnel that points straight at its Service instead of the Gateway; Radarr, Sonarr, Sabnzbd and Prowlarr are LAN-only.

Monitoring apps (Prometheus, Grafana) land in [Phase 4](/phases/phase-4-observability-security.md). Game servers are out of scope.
