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
