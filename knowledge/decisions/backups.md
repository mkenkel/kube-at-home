---
type: Decision
title: Backups
description: Two-point backup plan; local Longhorn and Velero to TrueNAS, offsite Duplicati leg deferred.
tags: [backup, velero, longhorn, duplicati]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 4
---

* Local (decided 2026-11-03): Longhorn recurring backups of the app config volumes (Sonarr, Radarr, Sabnzbd, Prowlarr, Overseerr/Seerr, Grafana) to an NFS dataset `k8s/longhorn-backups` on the TrueNAS `k8s` pool, exported to the four node IPs only. Nightly (keep 14) and weekly (keep 8). Not backed up: Prometheus, Loki, media. Details and steps: [backup and restore runbook](/runbooks/backup-restore.md).
* Velero is dropped: manifests and SOPS secrets are in Git, and the cluster is rebuilt from Ansible and Flux. There are no etcd snapshots by choice.
* Offsite: Duplicati to a cloud provider, scope and target not planned. It does not point at TrueNAS. Deployed last in Phase 5 administration.
* Git backs up manifests; the age key and Duplicati passphrase live in the password manager.
* Media is replaceable and covered by TrueNAS. Test restore at least quarterly.
