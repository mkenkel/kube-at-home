---
type: Deployment Phase
title: "Phase 5: Applications"
description: One category per PR, lowest risk first, using a shared app-template chart.
tags: [phase, apps]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 5 and section 4
  - id: plan
    resource: /plan.md
    title: App list
---

Use the bjw-s `app-template` chart for the one-container/one-PVC/one-ingress shape. One PR per category, in order:

1. Administration: Glance, then Gitea last (the user's choice, 2026-11-04). Duplicati is deferred and Portainer is dropped.
2. Notetaking: nothing deployed here.
3. Media: Jackett, Sonarr, Radarr, Lidarr, Sabnzbd, Jellyseerr, Wizarr, Notifiarr, Tautulli. Shared TrueNAS media volume; the media server box must be reachable from the cluster. Only Overseerr/Seerr (`requests`) is public ([exposure](/decisions/exposure-access.md)).
4. File management: nothing deployed here.
5. Home automation: nothing deployed here.

# Media (first category)

- [x] [repo] Applied 2026-11-03 (PR #4; all five pods Running, routes Accepted, `media-isolation` tested from the Sonarr pod): Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr as bjw-s `app-template` 5.3.0 HelmReleases in `apps/base/<app>/`, assembled in `apps/home/media/` (namespace `media`, PSA `baseline` with `restricted` warnings; HTTPRoutes at `<app>.home.mkenkel.com`; Longhorn config volumes that are kept on uninstall; Sonarr, Radarr and Sabnzbd mount the `media` claim at `/media`; Cilium network policy `media-isolation`). Rendered with `helm template` and checked with `kustomize build`, `yamllint --strict` and `kubeconform` (HTTPRoutes are skipped by the schema catalog).
- [x] [human] Done 2026-11-03: Prowlarr, Sonarr, Radarr, Sabnzbd and Overseerr moved from the Docker stack with their existing data (Sonarr 25 series / 975 files, Radarr 92 movies, Overseerr 122 requests, all counts matched). The Docker copies of the first four are stopped and not removed; the Docker Overseerr still runs so the public `requests` hostname keeps loading. Method, pitfalls found and rollback are in the [migration runbook](/runbooks/media-migration.md).
- [x] [repo] Applied 2026-11-03 (PR #5; log reports the migration completed, 122 requests and 1 user as before, Sonarr, Radarr and Plex tests pass, Plex login works): Overseerr is archived (last release 1.35.0, 2026-02-15), so the `overseerr` release runs its successor Seerr v3.5.1 (`ghcr.io/seerr-team/seerr`), which migrates the existing config and database in place on first start (one-way). Preparation done first: config backed up and the volume chowned to 1000:1000. Verify after the rollout: 122 requests, 1 user, Sonarr, Radarr and Plex connection tests pass.
- [ ] [repo] In-cluster `cloudflared` (2 replicas, restricted PSA, egress-only policy) staged in `apps/home/cloudflared/`, not yet included in `apps/home/kustomization.yaml` because its pods need the `cloudflared-token` secret first. `media-isolation` already allows it in on port 5055 only.
- [ ] [human] Create the tunnel and its SOPS secret, add Cloudflare Access policies, then move the `requests` hostname: see the public hostnames section of the [migration runbook](/runbooks/media-migration.md).

Decisions for this category (2026-11-03): the app set is what the user runs today (Prowlarr instead of the plan's Jackett, Overseerr instead of Jellyseerr, now running Seerr; Lidarr, Notifiarr, Tautulli and Wizarr are not included); existing config is migrated, not rebuilt; library and downloads share one `/media` mount so imports are renames; Sabnzbd keeps incomplete downloads on the NFS volume to stay off the etcd disk; Only Overseerr stays public, through a new in-cluster tunnel that points straight at its Service instead of the Gateway; Radarr, Sonarr, Sabnzbd and Prowlarr are LAN-only.

Monitoring apps (Prometheus, Grafana) land in [Phase 4](/phases/phase-4-observability-security.md). Game servers are out of scope.

# Administration (second category)

- [x] [repo] Applied 2026-11-04 (PR #11, then #12 for the checks; all 10 checks OK from inside the cluster): Glance v0.8.7 (`apps/base/glance`, ConfigMap `glance-config` and the rest in `apps/home/administration/`), namespace `glance` (PSA `restricted`), at `https://glance.home.mkenkel.com` on the LAN: a home page with live up/down checks for the media apps, Grafana, Hubble UI, ntfy, Plex and TrueNAS, plus two links. Stateless, no login, no icons (Glance loads them from cdn.jsdelivr.net, so the page makes no outside requests). The links use the Gateway names, but each check goes straight to the service's own health URL on its port (a pod calling a Gateway name gets `Access denied` from Envoy, see below), so a green dot means the service answers, not that its Gateway route works; Plex and TrueNAS are checked on the LAN. Config validated by running the same v0.8.7 binary locally: all 10 checks and both links render and the page references no external hosts. First version applied 2026-11-04 (pod, route and page worked) but its 8 Gateway-name checks all showed 403 `Access denied`; fixed by checking the services directly and opening exactly those ports in `glance-isolation`, `media-isolation` and `ntfy-isolation`.
- [ ] [repo] Headlamp 0.46.0 (`apps/base/headlamp`, chart from kubernetes-sigs.github.io/headlamp), read only, at `https://headlamp.home.mkenkel.com` on the LAN, namespace `headlamp`. Login with a short-lived token for `headlamp-viewer` (built-in `view` role plus read-only access to nodes, volumes and the Flux, Cilium, Longhorn, Prometheus, Gateway API and cert-manager resources; no Secrets, no exec, no writes). The chart binds the pod's own ServiceAccount to `cluster-admin` by default, switched off here so the pod has no permissions. Helm plugin, OIDC and the pod-account mode stay off. Runbook: [Headlamp](/runbooks/headlamp.md). Tick when it is applied and the `kubectl auth can-i` checks in the runbook give the expected answers.
- [ ] [repo] Gitea, last: not started. Open choices: SQLite or PostgreSQL, HTTPS only or also SSH, repos included in the Longhorn config backups.
