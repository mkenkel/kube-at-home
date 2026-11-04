---
type: Runbook
title: Move the media apps from Docker to the cluster
description: What was deployed, the order and method used on 2026-11-03 to move Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr off the dockery Docker stack, the pitfalls found, and how to roll back.
tags: [runbook, media, migration, phase-5]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: ansible
    resource: /home/matt/repos/ansible/playbooks/templates/docker-compose.yml
    title: Docker stack on dockery (outside this repo)
  - id: seerr
    resource: https://docs.seerr.dev/migration-guide/
    title: Seerr migration guide
---

This was carried out on 2026-11-03 for all five apps. The Docker copies of Prowlarr, Sonarr, Radarr and Sabnzbd are stopped (not removed) and their config folders under `/home/whally/` are untouched; they are the rollback. The Docker Overseerr is still running so the public `requests` hostname keeps loading until the tunnel cutover below.

# What is deployed

* Namespace `media` (PSA `baseline`), apps in `apps/base/` rendered by the bjw-s `app-template` chart 5.3.0, one HelmRelease each: Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr (now running Seerr, see the last section).
* Gateway hostnames `<app>.home.mkenkel.com`, which need the OPNsense Unbound override for `*.home.mkenkel.com` -> 192.168.15.100.
* A Longhorn config volume per app, named after it (Sonarr and Radarr 10Gi, others 2Gi), kept if the release is removed. Sonarr, Radarr and Sabnzbd also mount the `media` claim at `/media`, so library and downloads are one mount and imports are renames.
* LinuxServer apps run as 568:568 through `PUID`/`PGID`. Timezone `America/New_York`.
* Sabnzbd completes into `/media/downloads/complete` and unpacks in `/media/downloads/incomplete` on NFS, not on a node disk: each node has one NVMe that also holds etcd.
* Tools differ from the plan doc: Prowlarr instead of Jackett, Overseerr/Seerr instead of Jellyseerr. Lidarr, Notifiarr, Tautulli and Wizarr are not deployed.

# Order and method used

Prowlarr, Sonarr, Radarr, Sabnzbd, Overseerr. Each was checked before the next. The apps' own backup and restore (Sonarr, Radarr, Prowlarr) and their HTTP APIs were used; everything can also be done in the web UI.

## Prowlarr, Sonarr, Radarr

1. Check both sides run the same version (a backup restores only into the same or a newer version) and that the Docker queue is empty.
2. Back up the Docker app (API `Backup` command or System > Backup > Backup Now). The zip appears in `/home/whally/<app>/config/Backups/manual/` and holds only `<app>.db`, `config.xml` and `INFO`.
3. Stop the Docker app, so two instances never manage the library.
4. Restore into the cluster app: System > Backup > Restore Backup, or `POST /api/v3/system/backup/restore/upload`. **Restart the pod afterwards** (`kubectl -n media rollout restart deploy/<app>`). Restoring replaces `config.xml`, so the running app still holds the old API key and its own restart call fails with 401.
5. **Copy `MediaCover`.** Posters and banners are not in the backup. Stream it over SSH: `ssh whally@<dockery> 'sudo tar czf - -C /home/whally/<app>/config MediaCover' | kubectl -n media exec -i deploy/<app> -- sh -c 'tar xzf - -C /config && chown -R 568:568 /config/MediaCover'` (Sonarr 28 MB, Radarr 153 MB).
6. Fix paths. The restored database points at `/media/tv/...` (Sonarr) and `/media/movies/...` (Radarr), which do not exist in the cluster, so the apps report a missing root folder and skip disk cleanup (nothing is deleted). Add `/media/TV Shows` or `/media/Movies` as the root folder and set each series or movie path explicitly with `moveFiles=false`, after checking every folder with files exists on disk. Do not rely on the Mass Editor's root folder change: it can derive the folder name from the naming format instead of keeping the existing one. Then remove the old root folder entry. **Radarr only:** its movie collections have their own root folder and must be updated too (30 of them here) or health reports "Missing root folder for movie collection".
7. **Download client.** The restored Sonarr and Radarr download client pointed at the Docker host by IP (`192.168.15.25:8080`), not at the name `sabnzbd`. Change the host to `sabnzbd` (same port and API key) and press Test.
8. Test the indexers and check System > Health.

Prowlarr here had one indexer (NZBgeek) and no applications, so only steps 1 to 4 and a restart applied.

## Sabnzbd

Sabnzbd rewrites its ini when it exits, so never edit the file inside a running pod.

1. Copy `/home/whally/sabnzbd/sabnzbd.ini` (owned by uid 568, mode 600: read it with `sudo cat` over SSH). The Docker copy can keep running meanwhile.
2. In the copy set `download_dir = /media/downloads/incomplete` and `complete_dir = /media/downloads/complete`, and append `sabnzbd.home.mkenkel.com,` to `host_whitelist`. The Docker config had the two directory settings swapped relative to their names.
3. Scale the cluster Sabnzbd to zero, mount its volume in a throwaway busybox pod, copy the file to `/config/sabnzbd.ini`, delete the pod, scale back to one. LinuxServer's init fixes ownership.
4. **Whitelist the service names.** Sonarr and Radarr reach Sabnzbd as `sabnzbd:8080`, and Sabnzbd answers 403 "Hostname verification failed" to any host not in `host_whitelist`. Add `sabnzbd`, `sabnzbd.media`, `sabnzbd.media.svc` and `sabnzbd.media.svc.cluster.local`, for example through its API (`mode=set_config&section=misc&keyword=host_whitelist`), which saves the ini itself.
5. Press Test on the Usenet server in the UI.
6. Stop the Docker Sabnzbd only after the Docker Sonarr and Radarr are stopped, and with its queue empty: they used it as their download client.

## Overseerr

1. Briefly stop the Docker container for a consistent SQLite copy (`db/` holds `db.sqlite3` plus its `-wal` and `-shm` files) and copy `settings.json` and `db`. Start it again if the public hostname must stay up.
2. In the copied `settings.json` replace the Sonarr and Radarr entries' host and port (they were `192.168.15.25:8989` and `:7878`) with `sonarr:8989` and `radarr:7878`.
3. Scale the cluster Overseerr to zero, replace `db/` and `settings.json` with a helper pod, scale up.
4. Fix the folders new requests are sent to, which were `/media/tv` and `/media/movies`: set them to `/media/TV Shows` and `/media/Movies` through Overseerr's API (`PUT /settings/sonarr/<id>` with the entry minus its read-only `id`), not by editing the file while it runs.
5. Compare against Docker: request count (122) and users (1) matched. Test the Sonarr, Radarr and Plex connections from inside the pod.

# Rollback

Scale the cluster app to zero (`kubectl -n media scale deploy/<app> --replicas=0`) and `docker start <app>` on dockery. If Sonarr or Radarr already imported or renamed files in the cluster, that is on the shared library and does not roll back. After a restore the Docker app's download client and Overseerr entries still point where they did before.

# After the move

* The Ansible `docker-compose.yml` template for dockery is outside this repo and was left alone on purpose, because `dockery` is expected to join the cluster later. Until then **do not run that playbook**: it still defines the migrated apps and would start the Docker copies again.
* The old public hostnames `radarr.mkenkel.com` and `sabnzbd.mkenkel.com` on the Docker tunnel should be removed in Cloudflare by the user; Radarr and Sabnzbd are not meant to be public. The repo does not change existing records.
* Image tags are pinned inside the HelmReleases and Renovate here does not bump them.
* Network policy `media-isolation` allows ingress from the Gateway, the namespace and the in-cluster `cloudflared` (port 5055 only), and egress to the namespace, DNS and anything outside the cluster. Verified from the Sonarr pod: other apps and the internet reachable, Prometheus, Grafana and the Longhorn UI blocked.
* Two API keys (Sabnzbd's and Sonarr's) were printed into the working session while migrating. Both are LAN-only; rotate them when convenient (regenerate, then update the entries that use them).

# Overseerr is now Seerr

Overseerr's repository is archived (last release 1.35.0, 2026-02-15). Its successor Seerr migrates an existing Overseerr config and database in place on first start; treat that as one-way. Per the [migration guide](https://docs.seerr.dev/migration-guide/), the image is `ghcr.io/seerr-team/seerr` (the HelmRelease pins `v3.5.1`), it runs as uid 1000 (`node`) so `/app/config` must be owned by 1000:1000, and the Kubernetes settings are `fsGroup: 1000`, `fsGroupChangePolicy: OnRootMismatch` and a non-root, no-escalation, all-capabilities-dropped, `RuntimeDefault` security context. The release, Service, route and volume keep the name `overseerr`.

Preparation done before the switch: the cluster Overseerr config (`settings.json` and `db`) was backed up and the volume chowned to 1000:1000 while Overseerr was scaled down. Rollback is to restore that backup into the volume and revert the image change. The Docker Overseerr config is a second, older fallback.

# Public hostname: `requests`

Only Seerr is public. Radarr, Sonarr, Sabnzbd and Prowlarr are LAN and WireGuard only. The in-cluster `cloudflared` is written and staged in `apps/home/cloudflared/` but not switched on, because its pods cannot start without the tunnel token. Nothing here changes Cloudflare records until you do it. Cloudflare dashboard wording may differ from the names below.

1. In Cloudflare Zero Trust, create a new tunnel (a second one, not the Docker `homelab-tunnel`) and copy its token. Do not add the hostname yet.
2. Put the token in an encrypted secret, from the repo root, with your age key available (replace `<TOKEN>`; do not paste it anywhere else):
   `kubectl create secret generic cloudflared-token -n cloudflared --from-literal=token='<TOKEN>' --dry-run=client -o yaml | sops --encrypt --filename-override apps/home/cloudflared/cloudflared-token.sops.yaml /dev/stdin > apps/home/cloudflared/cloudflared-token.sops.yaml`
3. Uncomment `cloudflared-token.sops.yaml` in `apps/home/cloudflared/kustomization.yaml`, add `- cloudflared` to `apps/home/kustomization.yaml`, and send it through Git like everything else. The tunnel shows as Healthy when both pods connect.
4. Create a Cloudflare Access application and policy for `requests` (allow only the people who use it). Mobile and API clients cannot pass the Access login without a service token.
5. Once the cluster Seerr works over its LAN name: stop the Docker Overseerr (re-copy its database first if it took new requests you want to keep), then in the old tunnel remove the `requests` public hostname, and in the new tunnel add it with the service URL `http://overseerr.media.svc.cluster.local:5055`. Cloudflare repoints the DNS record; expect a short gap.
6. Rollback: put the hostname back on the Docker tunnel and start the Docker Overseerr.

See [open questions](/open-questions.md) and [exposure](/decisions/exposure-access.md).
