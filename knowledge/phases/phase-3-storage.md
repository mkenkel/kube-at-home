---
type: Deployment Phase
title: "Phase 3: Storage"
description: Longhorn for configs and databases, shared NFS media volume from TrueNAS.
tags: [phase, longhorn, nfs]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 3
---

- [ ] [repo] Longhorn `HelmRelease`, default `StorageClass`, replica count suited to 4 nodes
- [ ] [human] TrueNAS NFS export for media with the agreed UID/GID
- [ ] [repo] NFS `PersistentVolume`/`PersistentVolumeClaim` for the media volume
- [ ] [human] Verify node prerequisites from Ansible (iscsid, multipath blacklist) before installing Longhorn

Decision: [storage](/decisions/storage.md).
