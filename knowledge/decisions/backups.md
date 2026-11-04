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

* Local: Longhorn recurring backups + Velero to a dedicated TrueNAS dataset (NFS, or S3 via MinIO). The dataset is still undecided; see [open questions](/open-questions.md).
* Decision 2026-11-03: wait until the apps are hosted before setting up backups. Back up app configs and databases (Longhorn volumes), not Movies or TV. The TrueNAS pool `k8s` (separate from `Poolio`, about 8.95 TiB free, holding unused leftovers from an earlier attempt) is the likely target; nothing has been created there.
* Offsite: Duplicati to a cloud provider, scope and target not planned. It does not point at TrueNAS. Deployed last in Phase 5 administration.
* Git backs up manifests; the age key and Duplicati passphrase live in the password manager.
* Media is replaceable and covered by TrueNAS. Test restore at least quarterly.
