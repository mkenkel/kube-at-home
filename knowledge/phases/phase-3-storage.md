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

- [x] [repo] Applied 2026-11-01 (HelmRelease Ready, all pods Running; a 1Gi test PVC bound, mounted, and kept data; the volume was healthy with 3 replicas on node-1..3; removed afterwards): Longhorn v1.13.0 `HelmRelease` (default `StorageClass`, 3 replicas, disk-usage limits for the single-NVMe layout; needs Kubernetes >= 1.34)
- [ ] [human] TrueNAS NFS export for media with the agreed UID/GID
- [ ] [repo] NFS `PersistentVolume`/`PersistentVolumeClaim` for the media volume
- [x] Node prerequisites checked 2026-11-01 on all four nodes: `iscsid` active, `iscsi_tcp` loaded, `open-iscsi` and `nfs-common` installed, `multipathd` inactive, `findmnt`/`blkid`/`lsblk`/`curl` present, root disk 916G with ~865G free. `longhornctl check preflight` was not run.

Decision: [storage](/decisions/storage.md).
