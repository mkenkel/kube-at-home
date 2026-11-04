---
type: Runbook
title: Move the media apps from Docker to the cluster
description: Order, commands and rollback for moving Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr from the dockery Docker stack to the cluster.
tags: [runbook, media, migration, phase-5]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-03T00:00:00Z }
sources:
  - id: ansible
    resource: /home/matt/repos/ansible/playbooks/templates/docker-compose.yml
    title: Docker stack on dockery (outside this repo)
---

All steps here are **[human]**: they change running apps. Nothing in this runbook has been run yet.

# What the PR deploys

* Namespace `media` (PSA `baseline`), apps from `apps/base/` rendered by the bjw-s `app-template` chart 5.3.0, one HelmRelease each: Sonarr, Radarr, Sabnzbd, Prowlarr, Overseerr.
* Hostnames on the Gateway: `sonarr`, `radarr`, `sabnzbd`, `prowlarr`, `overseerr` + `.home.mkenkel.com`. These need the OPNsense Unbound override for `*.home.mkenkel.com` -> 192.168.15.100 ([open questions](/open-questions.md)).
* Each app has a Longhorn config volume named after the app (`sonarr`, `radarr`, ...; Sonarr and Radarr 10Gi, others 2Gi) that is kept if the release is removed. Sonarr, Radarr and Sabnzbd also mount the `media` claim at `/media`.
* All of them run as 568:568 (LinuxServer `PUID`/`PGID`; Overseerr ignores them). Timezone is `America/New_York`.
* Plan-doc tools replaced by what you run today: Prowlarr instead of Jackett, Overseerr instead of Jellyseerr. Lidarr, Notifiarr, Tautulli and Wizarr are not in this PR.
* Sabnzbd completes into `/media/downloads/complete` and unpacks in `/media/downloads/incomplete`, on the NFS volume, not on a node disk: the nodes have one NVMe that also holds etcd, and heavy unpack I/O there is a risk. Library and downloads sit in one mount, so imports are renames.

# Service names did not change

The Docker stack reaches apps by container name (`sabnzbd`, `sonarr`, `radarr`). The cluster Services have the same names and ports in the same namespace, so download clients, indexer apps and Overseerr's Sonarr/Radarr entries keep working after a restore without editing.

# Before you start

1. PR merged, Flux applied, `kubectl -n media get pods` shows all five Running (they start with empty default config).
2. DNS override in place so the hostnames resolve.
3. Let the Sabnzbd queue finish on Docker; do not move mid-download.
4. Keep the Docker config folders under `/home/whally/` untouched for a few weeks. They are the rollback.
5. **Do not let two instances of the same app manage the library.** Stop the Docker copy once its cluster twin works.
6. Stopping Docker Radarr, Sabnzbd or Overseerr also takes down the public hostnames `radarr`, `sabnzbd` and `requests` that the Docker `cloudflared` serves. Decide whether you still want those before you cut over.

# Order

Prowlarr, Sonarr, Radarr, Sabnzbd, Overseerr. One app at a time; check it works before the next.

## Prowlarr, Sonarr, Radarr (built-in backup)

1. In the Docker app: System > Backup > Backup Now, download the zip (scheduled backups are also in the app's `Backups` folder under `/home/whally/<app>/config/`).
2. In the cluster app: System > Backup > Restore Backup > upload the zip. The app restarts itself.
3. Stop the Docker app: `docker stop sonarr` (and so on).
4. Sonarr and Radarr only, fix paths. The restored database points at `/media/tv/...` and `/media/movies/...`, which do not exist in the cluster.
   * Sonarr: Settings > Media Management > Root Folders > add `/media/TV Shows`. Series > Mass Editor > select all > Root Folder `/media/TV Shows` > Save. Answer **No** when asked to move files.
   * Radarr: same with `/media/Movies` and Movies > Mass Editor.
   * Check one series and one movie in the UI, then run Update Library on both.
5. Check the download client entries (host `sabnzbd`, port 8080) and indexers, and press Test.

## Sabnzbd (copy the ini with the pod stopped)

Sabnzbd rewrites its ini when it exits, so a file copied into a running pod can be overwritten. Copy with the app scaled to zero.

1. On dockery: `docker stop sabnzbd`. Copy `/home/whally/sabnzbd/sabnzbd.ini` to your workstation.
2. Edit the copy: set `complete_dir = /media/downloads/complete`, `download_dir = /media/downloads/incomplete`, and add `sabnzbd.home.mkenkel.com` to `host_whitelist` (otherwise the new hostname is refused). Server and category settings stay as they are.
3. `kubectl -n media scale deploy/sabnzbd --replicas=0`
4. Start a helper pod that mounts the `sabnzbd` volume at `/config` (a busybox pod running `sleep 3600`), `kubectl cp sabnzbd.ini media/<helper>:/config/sabnzbd.ini`, then delete the helper.
5. `kubectl -n media scale deploy/sabnzbd --replicas=1`. LinuxServer's init fixes the ownership of `/config` to 568.
6. Open `sabnzbd.home.mkenkel.com`, check the Usenet server test and the folders. The API key is unchanged, so Sonarr and Radarr need no edit.

## Overseerr (copy settings and database with the pod stopped)

1. `docker stop overseerr`. Copy `settings.json` and the `db` folder from `/home/whally/overseerr/config/` to your workstation.
2. `kubectl -n media scale deploy/overseerr --replicas=0`, then use a helper pod on the `overseerr` volume (mounted at `/app/config`) as above, copy both in, delete the helper.
3. `kubectl -n media scale deploy/overseerr --replicas=1`. Check Settings > Services: Plex, Sonarr and Radarr entries use the same names and should test green. Users and requests come across with the database.

# Rollback

Per app: scale the cluster Deployment to zero (`kubectl -n media scale deploy/<app> --replicas=0`) and `docker start <app>` on dockery. The Docker config was not changed. If Sonarr or Radarr already imported or renamed files in the cluster, that is on the shared library and does not roll back; Sonarr and Radarr note moves in their history.

# After the move

* Remove the migrated apps from the Ansible `docker-compose.yml` template (outside this repo) so the next playbook run does not bring them back.
* Image tags are pinned inside the HelmReleases and Renovate here does not bump them; update by hand or add a Renovate rule later.
* Network policy `media-isolation` allows ingress from the Gateway and the namespace only, and egress to the namespace, DNS and anything outside the cluster. If a page does not load through the Gateway after applying, suspect it first and remove `networkpolicy.yaml` from `apps/home/media/kustomization.yaml` to confirm.

# Public hostnames: `requests`, `radarr`, `sabnzbd`

The in-cluster `cloudflared` is written and staged in `apps/home/cloudflared/` but not switched on, because its pods cannot start without the tunnel token. Nothing here changes Cloudflare records until you do it. Cloudflare dashboard wording may differ from the names below.

1. In Cloudflare Zero Trust, create a new tunnel (a second one, not the Docker `homelab-tunnel`) and copy its token. Do not add hostnames yet.
2. Put the token in an encrypted secret, from the repo root, with your age key available (replace `<TOKEN>`; do not paste it anywhere else):
   `kubectl create secret generic cloudflared-token -n cloudflared --from-literal=token='<TOKEN>' --dry-run=client -o yaml | sops --encrypt --filename-override apps/home/cloudflared/cloudflared-token.sops.yaml /dev/stdin > apps/home/cloudflared/cloudflared-token.sops.yaml`
3. Uncomment `cloudflared-token.sops.yaml` in `apps/home/cloudflared/kustomization.yaml`, add `- cloudflared` to `apps/home/kustomization.yaml`, and send it through Git like everything else. The tunnel shows as Healthy when both pods connect.
4. Create Cloudflare Access applications and policies for `requests`, `radarr` and `sabnzbd` (allow only you, plus whoever uses Overseerr). Radarr and Sabnzbd are admin interfaces, so do not skip this for them. Mobile and API clients cannot pass the Access login without a service token.
5. Cut over one hostname at a time, only after the cluster app works over the LAN name. In the old tunnel remove the public hostname, then in the new tunnel add it with the service URL: `http://overseerr.media.svc.cluster.local:5055`, `http://radarr.media.svc.cluster.local:7878` or `http://sabnzbd.media.svc.cluster.local:8080`. Cloudflare repoints the DNS record; expect a short gap.
6. Sabnzbd refuses unknown hostnames: add `sabnzbd.mkenkel.com` to `host_whitelist` in the ini if it is not already there.
7. Rollback is the reverse: put the hostname back on the Docker tunnel and start the Docker app.

See [open questions](/open-questions.md) and [exposure](/decisions/exposure-access.md).
