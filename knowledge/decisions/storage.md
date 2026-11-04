---
type: Decision
title: Storage
description: Longhorn for app configs and databases; media on the existing TrueNAS over NFS.
tags: [longhorn, nfs, truenas]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 3
---

* Longhorn holds app configs and databases (1 GbE makes Rook-Ceph overkill).
* One shared NFS media volume on TrueNAS for the *arr stack and download client, with consistent UID/GID.
* Longhorn needs `open-iscsi`, `iscsid`, `nfs-common` and a multipath blacklist on nodes (done in Ansible, [Phase 0](/phases/phase-0-provisioning.md)).

# Media volume (2026-11-03)

* Server and export: `192.168.15.24:/mnt/Poolio/media`. It is a folder, not a dataset, in the `Poolio` root dataset, so moving `Movies` and `TV Shows` into it from the pool root was an instant rename. A child dataset was tried and dropped: moving into it would have been a full copy of about 2.8T.
* Owner: `apps` (568:568), enforced by Mapall on the share, so every client writes as 568 whatever UID it runs as. Directories are 755 and files 644 so Plex (on its own box, `plexy`) can read them.
* The Docker host (`dockery`) and `plexy` mount the same export at `/media`. Its `*arr` containers run as 568 through `MEDIA_PUID`/`MEDIA_PGID` in the Ansible `env.j2`; other containers on that host keep their own IDs.
* In the cluster: a static `PersistentVolume` (`media-nfs`, `Retain`, empty storage class so Longhorn's default class does not claim it) bound to the claim `media/media`. Pods that need to write should run as 568 or rely on Mapall.
* `downloads/` exists on the export but is unused. The Docker stack still downloads to local NVMe and copies to the NAS, and the cluster apps should avoid that: mount one volume containing both library and downloads in each app so imports are renames. Not yet applied on the Docker side.
* Not done: the old `/mnt/Poolio` share is still active, and nothing has mounted the PV from a node yet.

# Disks (2026-11-01)

* Each node has a single 1TB NVMe holding the OS, etcd and Longhorn (`/var/lib/longhorn` on `/`). The earlier plan assumed a second disk; there is none. Media is on NFS, so Longhorn only holds configs and databases and the capacity is ample.
* Cost: Longhorn and etcd share one disk, so heavy volume I/O can add etcd fsync latency, and a full disk would hurt etcd and kubelet. Mitigation in the HelmRelease: 30% reserved, 25% minimal available, no over-provisioning, 3 replicas. A second disk can be added later and set as a Longhorn disk without touching the OS.
* The Longhorn UI has no authentication, so it is not exposed.
