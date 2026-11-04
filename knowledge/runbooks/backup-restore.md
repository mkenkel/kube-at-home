---
type: Runbook
title: Back up and restore the app config volumes
description: What Longhorn backs up, how to set up the TrueNAS target, how to check it, how to test a restore, and what is deliberately not covered.
tags: [runbook, backup, longhorn, truenas]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: longhorn
    resource: https://longhorn.io/docs/1.10.0/snapshots-and-backups/scheduling-backups-and-snapshots/
    title: Longhorn scheduling backups and snapshots (1.10 docs; the cluster runs 1.13.0)
---

# What is backed up

* The config volumes of Sonarr, Radarr, Sabnzbd, Prowlarr, Overseerr/Seerr and Grafana. They hold the databases, settings and cover art that would be tedious to rebuild.
* Not backed up on purpose: Prometheus and Loki data (replaceable metrics and logs, 30Gi each), and the media itself (it lives on TrueNAS).
* Cluster state needs no backup: every manifest is in Git, secrets are SOPS-encrypted in Git, and the age key is in the password manager. There are no etcd snapshots; a lost control plane is rebuilt with the Ansible playbooks and Flux. Velero is not used for that reason.
* No offsite copy yet (deferred). The backups sit on the same TrueNAS box as the media, on the separate `k8s` pool.

Backups are incremental and crash-consistent (like a power cut). SQLite in WAL mode recovers from that, and Sonarr, Radarr and Prowlarr also write their own scheduled backups into their config volume.

# Schedule

Two Longhorn recurring jobs in the group `config` (`infrastructure/configs/longhorn/recurringjobs.yaml`):

* `config-nightly`: every day 02:30, keep 14.
* `config-weekly`: Sundays 03:30, keep 8.

A volume is included when its PVC carries both labels `recurring-job.longhorn.io/source: enabled` and `recurring-job-group.longhorn.io/config: enabled`. Longhorn copies PVC labels to the volume periodically, and the `source` label is what makes the PVC win over Longhorn's automatic `default` group. The five app volumes get them through `persistence.config.labels` in their HelmReleases, Grafana through `grafana.persistence.extraPvcLabels`. **A new app's volume must be labelled the same way or it is not backed up.**

# TrueNAS setup **[human]**

Nothing below touches the existing `k8s/nfs` and `k8s/iscsi` leftovers or the media export.

1. Datasets: select the `k8s` pool, Add Dataset, name `longhorn-backups`, default options. A quota of a few hundred GiB bounds growth if you want one.
2. Permissions on that dataset: owner and group `apps` (568). Longhorn writes as root, and Mapall maps that to 568.
3. Shares > Unix (NFS) Shares > Add: path `/mnt/k8s/longhorn-backups`, Advanced Options: Mapall User and Group `apps`, Maproot empty, and under **Hosts** add the four node addresses `192.168.15.7`, `192.168.15.8`, `192.168.15.9`, `192.168.15.10` (not the whole network).
4. System > Services > NFS: Longhorn mounts the target with NFSv4, so make sure v4 is enabled.
5. Recommended: a periodic ZFS snapshot task on `k8s/longhorn-backups` (daily, keep 14 days), as a safety net against a bad prune or an accidental delete.

# Turning it on and checking it

**An available target is not a working one.** `AVAILABLE true` only proves Longhorn can list the share. The first real backup is the test: when the TrueNAS dataset was owned by root while Mapall maps writes to 568, the target looked healthy but the backup failed with `mkdir .../backupstore: permission denied`. Make `apps` (568) the owner of the dataset.

The target URL in `infrastructure/configs/longhorn/backuptarget.yaml` is `nfs://192.168.15.24:/mnt/k8s/longhorn-backups`. It is set on the existing `default` BackupTarget directly, not through the Helm chart value: the chart's default-backup-store setting did not look like it would update a target that already exists, and the object is annotated so Flux never deletes it.

After the TrueNAS steps and once Flux has applied the change:

* `kubectl -n longhorn-system get backuptargets.longhorn.io default` should show `AVAILABLE true`. If it stays false, read the `status.conditions` message: mount errors usually mean the host list, NFSv4 or the dataset owner.
* `kubectl -n longhorn-system get recurringjobs.longhorn.io` lists both jobs.
* `kubectl -n longhorn-system get volumes.longhorn.io --show-labels | grep recurring-job-group.longhorn.io/config` should list the six config volumes within a few minutes (the Prometheus and Loki volumes stay on the old `default` group, which has no jobs).
* After the first 02:30 run, `kubectl -n longhorn-system get backups.longhorn.io` has an entry per volume and the share has a `backupstore` folder.

# Test a restore **[human]**, then quarterly

A backup that has never been restored is a hope. Done on 2026-11-03 with the smallest volume (Prowlarr), without the Longhorn UI:

1. Make a backup by hand if none exists yet: a `Snapshot` object (`spec.volume`, `spec.createSnapshot: true`) and a `Backup` object (`spec.snapshotName`, `spec.backupMode: incremental`, label `backup-volume: <volume>`) in `longhorn-system`; wait for `status.state: Completed`.
2. Create a StorageClass with provisioner `driver.longhorn.io`, `reclaimPolicy: Delete`, and parameters `numberOfReplicas: "1"` and `fromBackup: <the backup's status.url>`, a scratch namespace, and a PVC using that class.
3. Mount the PVC read-only in a throwaway pod. Check that the files are there, that `config.xml` has the same checksum as the live volume, and run `PRAGMA integrity_check` on a copy of the SQLite database.
4. Delete the namespace and the StorageClass; the restored volume goes with its claim. Keep the real backup.

The Longhorn UI works too (`kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`; it has no authentication, so keep it on localhost).

# Restoring for real

* A single app: scale its Deployment to zero, restore the newest good backup over a new volume as above, then point the PVC at it (or delete the damaged PVC and recreate it from the restored volume with the same name and namespace, which is what the HelmRelease expects), scale up.
* A rebuilt cluster: bring up the cluster, Flux and Longhorn from Git, apply the backup target, restore each volume from its backup with Create PV/PVC using the original names (`sonarr`, `radarr`, `sabnzbd`, `prowlarr`, `overseerr` in `media`, Grafana's PVC in `monitoring`) before the apps start, then let Flux converge.

# Why the firewall logged blocked traffic from TrueNAS

While the share did not exist, OPNsense logged "default deny / state violation" for TrueNAS connecting to a Longhorn manager pod address (`10.244.x.x`). Longhorn mounts the NFS target inside its own pod, so the NFS client advertises the pod IP, and the NFSv4.0 callback channel (tried as a fallback after 4.2 and 4.1 failed) points at an address nothing outside the cluster can route. It stopped once the share worked and the mount succeeded at NFSv4.2. If it returns, look at the backup target's status first.
