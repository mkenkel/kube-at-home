---
type: Deployment Phase
title: "Phase 5: Applications"
description: One category per PR, lowest risk first, using a shared app-template chart.
tags: [phase, apps]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 5 and section 4
  - id: plan
    resource: /plan.md
    title: App list
---

Use the bjw-s `app-template` chart for the one-container/one-PVC/one-ingress shape. One PR per category, in order:

1. Administration: Glance, Gitea, Portainer, then Duplicati last (needs its backup plan).
2. Notetaking: nothing deployed here.
3. Media: Jackett, Sonarr, Radarr, Lidarr, Sabnzbd, Jellyseerr, Wizarr, Notifiarr, Tautulli. Shared TrueNAS media volume; the media server box must be reachable from the cluster. Only Jellyseerr and Wizarr are public ([exposure](/decisions/exposure-access.md)).
4. File management: nothing deployed here.
5. Home automation: nothing deployed here.

Monitoring apps (Prometheus, Grafana) land in [Phase 4](/phases/phase-4-observability-security.md). Game servers are out of scope.
