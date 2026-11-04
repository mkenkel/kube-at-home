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
3. Media: Jackett, Sonarr, Radarr, Lidarr, Sabnzbd, Jellyseerr, Wizarr, Notifiarr, Tautulli. Shared TrueNAS media volume; the media server box must be reachable from the cluster. Public through the tunnel: Overseerr (`requests`), Radarr and Sabnzbd, which already have public hostnames ([exposure](/decisions/exposure-access.md)).
4. File management: nothing deployed here.
5. Home automation: nothing deployed here.

# Media (first category)

- [ ] [repo] Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr as bjw-s `app-template` 5.3.0 HelmReleases in `apps/base/<app>/`, assembled in `apps/home/media/` (namespace `media`, PSA `baseline` with `restricted` warnings; HTTPRoutes at `<app>.home.mkenkel.com`; Longhorn config volumes that are kept on uninstall; Sonarr, Radarr and Sabnzbd mount the `media` claim at `/media`; Cilium network policy `media-isolation`). Rendered with `helm template` and checked with `kustomize build`, `yamllint --strict` and `kubeconform` (HTTPRoutes are skipped by the schema catalog). Not verified on the cluster: tick this when Flux has applied it and the pods are Running.
- [ ] [human] Migrate the existing Docker apps in the order in the [migration runbook](/runbooks/media-migration.md), then stop each Docker copy.
- [ ] [repo] In-cluster `cloudflared` (2 replicas, restricted PSA, egress-only policy) staged in `apps/home/cloudflared/`, not yet included in `apps/home/kustomization.yaml` because its pods need the `cloudflared-token` secret first. `media-isolation` already allows it in on ports 5055, 7878 and 8080.
- [ ] [human] Create the tunnel and its SOPS secret, add Cloudflare Access policies, then move `requests`, `radarr` and `sabnzbd` one at a time: see the public hostnames section of the [migration runbook](/runbooks/media-migration.md).

Decisions for this category (2026-11-03): the app set is what the user runs today (Prowlarr instead of the plan's Jackett, Overseerr instead of Jellyseerr; Lidarr, Notifiarr, Tautulli and Wizarr are not included); existing config is migrated, not rebuilt; library and downloads share one `/media` mount so imports are renames; Sabnzbd keeps incomplete downloads on the NFS volume to stay off the etcd disk; Overseerr, Radarr and Sabnzbd are to stay public through a new in-cluster tunnel, pointing the tunnel straight at the app Services instead of the Gateway.

Monitoring apps (Prometheus, Grafana) land in [Phase 4](/phases/phase-4-observability-security.md). Game servers are out of scope.
