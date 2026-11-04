---
type: Deployment Phase
title: "Phase 3: Storage"
description: Longhorn for configs and databases, shared NFS media volume from TrueNAS.
tags: [phase, longhorn, nfs]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 3
---

- [x] [repo] Applied 2026-11-01 (HelmRelease Ready, all pods Running; a 1Gi test PVC bound, mounted, and kept data; the volume was healthy with 3 replicas on node-1..3; removed afterwards): Longhorn v1.13.0 `HelmRelease` (default `StorageClass`, 3 replicas, disk-usage limits for the single-NVMe layout; needs Kubernetes >= 1.34)
- [x] [human] TrueNAS NFS export for media with the agreed UID/GID: done 2026-11-03. `192.168.15.24:/mnt/Poolio/media` (a plain folder in the `Poolio` dataset holding `Movies`, `TV Shows` and `downloads`), allowed from `192.168.15.0/24`, Mapall `apps` (568:568), Maproot empty. The Docker host (`dockery`) and the Plex box (`plexy`) already mount it and Plex plays from it. The old `/mnt/Poolio` share still exists and should be removed once nothing uses it.
- [x] [repo] Applied 2026-11-03 (PR #1; `infrastructure-configs` Ready; PV `media-nfs` Bound to `media/media`): NFS `PersistentVolume`/`PersistentVolumeClaim` for the media volume in `infrastructure/configs/storage/` (namespace `media`). A throwaway busybox pod running as 568:568 on node-3 mounted the claim, listed `Movies`, `TV Shows` and `downloads`, created and removed a file, and was deleted. Only node-3 was tested.
- [x] Node prerequisites checked 2026-11-01 on all four nodes: `iscsid` active, `iscsi_tcp` loaded, `open-iscsi` and `nfs-common` installed, `multipathd` inactive, `findmnt`/`blkid`/`lsblk`/`curl` present, root disk 916G with ~865G free. `longhornctl check preflight` was not run.

Decision: [storage](/decisions/storage.md).
