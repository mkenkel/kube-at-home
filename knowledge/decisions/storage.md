---
type: Decision
title: Storage
description: Longhorn for app configs and databases; media on the existing TrueNAS over NFS.
tags: [longhorn, nfs, truenas]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 3
---

* Longhorn holds app configs and databases (1 GbE makes Rook-Ceph overkill).
* One shared NFS media volume on TrueNAS for the *arr stack and download client, with consistent UID/GID.
* Longhorn needs `open-iscsi`, `iscsid`, `nfs-common` and a multipath blacklist on nodes (done in Ansible, [Phase 0](/phases/phase-0-provisioning.md)).

# Disks (2026-11-01)

* Each node has a single 1TB NVMe holding the OS, etcd and Longhorn (`/var/lib/longhorn` on `/`). The earlier plan assumed a second disk; there is none. Media is on NFS, so Longhorn only holds configs and databases and the capacity is ample.
* Cost: Longhorn and etcd share one disk, so heavy volume I/O can add etcd fsync latency, and a full disk would hurt etcd and kubelet. Mitigation in the HelmRelease: 30% reserved, 25% minimal available, no over-provisioning, 3 replicas. A second disk can be added later and set as a Longhorn disk without touching the OS.
* The Longhorn UI has no authentication, so it is not exposed.
