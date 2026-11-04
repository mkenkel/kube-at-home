---
type: Status
title: Current status and handoff
description: Where the cluster and repo stand, what is blocked on the user, and how to resume from another host.
tags: [status, handoff]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: log
    resource: /log.md
    title: Update log
---

Snapshot taken 2026-11-02 from the live cluster: node-1..4 Ready (v1.36.6); Flux Kustomizations `flux-system`, `gateway-api-crds`, `infrastructure-controllers`, `infrastructure-configs`, `apps` all Ready; HelmReleases `cilium` 1.20.4, `cert-manager` v1.21.3, `longhorn` 1.13.0, `kube-prometheus-stack` 92.17.0, `loki` 20.2.1, `alloy` 1.15.0 all Ready; no pods outside Running/Completed; `home-wildcard` certificate Ready (production Let's Encrypt).

# Phase status

| Phase | State |
|---|---|
| 0 Provisioning | Done. Netboot items about headless boot and the NVMe check are unticked in [Phase 0](/phases/phase-0-provisioning.md) because the user never confirmed them. |
| 1 Bootstrap | Done. `ansible/cluster.yml` built the cluster; Flux bootstrapped; Cilium adopted by Flux. |
| 2 Networking | Done. L2 pool `.100-.200`, Gateway `gateway/home` at 192.168.15.100, wildcard `*.home.mkenkel.com`. |
| 3 Storage | Longhorn done and tested with a PVC. NFS media export is live on TrueNAS (568:568); PV `media-nfs` and claim `media/media` are applied by Flux and a test pod on node-3 read and wrote it. Other nodes untested. |
| 4 Observability and security | Prometheus, Grafana, control-plane metrics, Loki and Alloy done; the Longhorn ServiceMonitor was found to be scraping nothing and is being fixed with the alerting work. PSA `restricted` enforced on `cert-manager`, `gateway` and `flux-system` (applied 2026-11-03). **Network policies (Phase 5) and backups not applied.** |
| 5 Applications | Media category deployed and migrated 2026-11-03: Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr run in the `media` namespace with their old data (see [Phase 5](/phases/phase-5-applications.md) and the [migration runbook](/runbooks/media-migration.md)). The Overseerr release runs Seerr v3.5.1 since the in-place upgrade. The in-cluster `cloudflared` for the one public hostname (`requests`) is staged in `apps/home/cloudflared/` and waits on a tunnel token. Docker Prowlarr, Sonarr, Radarr and Sabnzbd are stopped, the Docker Overseerr still runs. |

# Blocked on the user

* Check the first scheduled backups of all six config volumes (first run 02:30 UTC); a manual Prowlarr backup and restore already worked. See the [backup runbook](/runbooks/backup-restore.md). Velero is dropped.
* Set up the ntfy app on the iPhone and send a test alert: [alerting runbook](/runbooks/alerting.md). The credentials are in `~/ntfy-phone-credentials.txt` on the workstation until moved to the password manager.
* A new Cloudflare tunnel token (as a SOPS secret) and an Access policy for `requests`, then moving that hostname off the Docker tunnel; and removing the old `radarr` and `sabnzbd` tunnel hostnames in Cloudflare. See the [runbook](/runbooks/media-migration.md).
* `dockery` (the Docker host) is meant to join the cluster later. The Ansible repo for it is outside this repo and was left alone, so do not run its playbook: it still defines the migrated apps and would restart the Docker copies.
* Optional TrueNAS tidy: remove the old `/mnt/Poolio` NFS share once nothing mounts it, and the unused `k8s` iSCSI/NFS leftovers.
* Gitea first use: create the admin account and add an SSH key (the `git.mkenkel.com` DNS override is done): [runbook](/runbooks/gitea.md).
* OPNsense Unbound host override `*` / `home.mkenkel.com` -> 192.168.15.100 (was being set up; not confirmed done). Without it `grafana.home.mkenkel.com` does not resolve on the LAN.

# Next steps, in order

1. The public cutover (tunnel token, Access, move `requests`), then retire the Docker Overseerr. Then the other Phase 5 categories, one per PR.
2. Gitea is deployed (last Phase 5 app); run the first-use steps in the [Gitea runbook](/runbooks/gitea.md).
4. Optional: ServiceMonitors for Loki and Alloy, Cilium/Hubble metrics (changes `cilium/values.yaml` and the HelmRelease together, restarts agents).

# Resuming from another host

* Clone the repo. Tools used: `ansible` plus `ansible-galaxy collection install -r ansible/requirements.yml`, `ansible-lint`, `kubectl`, `flux`, `kustomize`, `yamllint`, `sops`. Helm is not needed locally.
* Kubeconfig: `cluster.yml` only fetches it during init, so on a new host run `ssh ops@192.168.15.7 sudo cat /etc/kubernetes/admin.conf > ~/.kube/kube-at-home.conf` (the server is the VIP 192.168.15.6).
* SOPS: the age private key is in the user's password manager. Put it at `~/.config/sops/age/keys.txt` or point `SOPS_AGE_KEY_FILE` at it. Never commit it. When piping into sops use `--filename-override <path>.sops.yaml`, or no creation rule matches.
* SSH: nodes accept only the public key in `preseed/preseed.cfg` (user `ops`). The new host needs that key's private half or a second key added to `~ops/.ssh/authorized_keys`, and `ansible.cfg` has `host_key_checking=True`, so add the node host keys first.
* Working rules are in `/CLAUDE.md`: Git and Flux for cluster changes, confirm before touching the live cluster, no secrets in Git. Read-only checks with `kubectl` and Ansible ad-hoc commands were used freely; anything that changes the cluster outside Flux was confirmed first.

# Lessons from this build (keep in mind)

* Node RTCs were wrong on fresh installs (months off); chrony now runs first in the `common` role.
* containerd's default CNI path is `/usr/lib/cni`, Cilium installs to `/opt/cni/bin`; the `containerd` role sets `bin_dir`.
* Hand installs use Helm 3 (Helm 4 applies server-side by default and may clash with Flux adoption).
* The OSS Loki chart lives in `grafana-community/helm-charts`; `grafana/helm-charts` is Enterprise only. Render charts locally (`helm template`) before pushing; it caught a replica validation error.
* `*.sops.yaml` is excluded from yamllint (sops indents with 4 spaces).
* Existing Cloudflare records for mkenkel.com must not be touched; see [Exposure and access](/decisions/exposure-access.md).
* The user's workstation shell is fish; the user says "push it" to authorise a push and expects each change tested or verified before it is called done.

# Unverified

* Grafana Explore with the Loki datasource, and the Grafana login through the Gateway in a browser.
* kube-apiserver logs about 20 `grpc: addrConn.createTransport failed ... 127.0.0.1:2379: operation was canceled` warnings per 10 minutes per node; probably benign, unconfirmed, and not known whether it predates the etcd metrics change. Etcd shows 0 restarts.
* Longhorn behaviour on node failure; its replicas landed on node-1..3 only in the one test. `longhornctl check preflight` was not run.
* `flux diff` is not set up. (The GitHub Actions `validate` workflow has run and passed on every PR so far.)
* The Sabnzbd and Sonarr API keys that were printed into a working session were rotated on 2026-11-03; old copies remain in the stopped Docker config folders, an old Sonarr backup zip, `~/sabnzbd.ini` and `~/overseerr-config-backup-20261103.tgz` on the workstation.
