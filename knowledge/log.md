# Update Log

## 2026-11-04 (Headlamp)
* **Update**: Added Headlamp, a read-only cluster UI, at `headlamp.home.mkenkel.com` (LAN). Login is a short-lived token for `headlamp-viewer` (built-in `view` role plus read-only access to nodes, volumes and the custom resources in use; no Secrets, no exec, no writes). The chart binds the pod's own ServiceAccount to `cluster-admin` by default; that is turned off so the pod has no permissions. Also ticks Glance as applied: all 10 checks are OK from inside the cluster, and a ConfigMap change reaches the pod after about a minute and Glance then reloads by itself. See the [Headlamp runbook](/runbooks/headlamp.md).

## 2026-11-04 (Glance checks)
* **Update**: Glance was applied and served its page, but all 8 checks that used Gateway names showed 403 `Access denied` (Plex and TrueNAS, checked on the LAN, were fine). Cause: Envoy applies the calling pod's egress policy to the backend it forwards to. Glance now checks each service's own health URL on its own port, the links keep the Gateway names, and `glance-isolation`, `media-isolation` and `ntfy-isolation` open exactly those ports. A green dot now means the service answers, not that its Gateway route works. Noted in [Networking](/decisions/networking.md).

## 2026-11-04 (Glance)
* **Update**: Added Glance, the home page with live up/down checks, at `glance.home.mkenkel.com` (LAN only, namespace `glance`, stateless, config in a ConfigMap). Icons are left out because Glance loads them from a public CDN. Config checked by running the same v0.8.7 binary locally. Gitea moves to last, at the user's request. See [Phase 5](/phases/phase-5-applications.md).

## 2026-11-04 (Hubble UI)
* **Update**: Added an HTTPRoute for the already-running Hubble UI at `hubble.home.mkenkel.com` (LAN and WireGuard only, no login). No Cilium change, so no agent restart. See [Exposure and access](/decisions/exposure-access.md).

## 2026-11-04 (alert priority)
* **Update**: Critical alerts now arrive urgent. ntfy's built-in alertmanager template sets no priority, so a custom template (ConfigMap `ntfy-templates`, used as `?template=cluster`) sets it: critical firing = 5, warning = 3, resolved = 2. Tested locally with sample payloads first, including an alert with no severity label. The template is a plain ConfigMap because app-template runs Helm's templating over `{{ }}` in values, which broke the first attempt. Also ticks the alerting items as applied and verified. See the [alerting runbook](/runbooks/alerting.md).

## 2026-11-04 (Flux and certificate alerts)
* **Update**: Added alerts for Flux resources that stay unready, are suspended or lose a controller, and for certificates that expire within 21 days or are not ready. Flux state comes from kube-state-metrics' custom resource state config (copied from fluxcd/flux2-monitoring-example for the four kinds in use; the standard collectors stay on, which the example disables); cert-manager's controller metrics are scraped through a ServiceMonitor defined with the other monitoring configs. Each set has a rule that fires if its own metrics disappear. See the [alerting runbook](/runbooks/alerting.md).

## 2026-11-04 (alerting)
* **Update**: Added alerting: self-hosted ntfy (LAN only, auth with default deny, iPhone wake-up via ntfy.sh), Alertmanager delivery to it, and Longhorn alert rules. Found that Prometheus could never scrape Longhorn because the chart's manager network policy admits only Longhorn pods; fixed with `networkPolicies.metricsScrapeSources` for Prometheus only. Rotated the leaked Sabnzbd and Sonarr API keys on 2026-11-03 (also updated Sonarr, Radarr and Seerr to the new keys). Folded in: the first backup and restore test of the Prowlarr volume, the dataset-permission pitfall, the NFSv4.0 callback explanation of the firewall noise, and CI having passed on every PR. See [Alerting](/decisions/alerting.md) and the [runbook](/runbooks/alerting.md).

## 2026-11-03 (backups)
* **Update**: Designed backups of the app config volumes. Longhorn recurring jobs (nightly keep 14, weekly keep 8, group `config`) and the NFS backup target `nfs://192.168.15.24:/mnt/k8s/longhorn-backups` are written under `infrastructure/configs/longhorn/`; the five media config volumes and Grafana opt in through PVC labels, Prometheus and Loki do not. The `default` BackupTarget is set directly because the chart value would not clearly update an existing target. Velero is dropped. The TrueNAS dataset and share are still to be created by the user. Also ticks the Seerr upgrade as applied and verified. See [Backups](/decisions/backups.md) and the [runbook](/runbooks/backup-restore.md).

## 2026-11-03 (media migrated, Seerr)
* **Update**: Moved Prowlarr, Sonarr, Radarr, Sabnzbd and Overseerr from the Docker stack into the `media` namespace with their existing data; counts matched (25 series / 975 files, 92 movies, 122 requests, 1 user). Docker copies of the first four stopped, Docker Overseerr left running for the public hostname. Rewrote the [migration runbook](/runbooks/media-migration.md) from plan to record with the pitfalls found: backups omit cover art, restore needs a pod restart, download clients used the Docker IP, Sabnzbd's host whitelist needs the service names, Radarr collections carry their own root folder, Overseerr's server entries and request folders. Overseerr's repository is archived, so its release now runs Seerr v3.5.1 (config backed up and volume chowned first). The Ansible repo for dockery is left alone on purpose. See [Phase 5](/phases/phase-5-applications.md).

## 2026-11-03 (media apps)
* **Update**: Added the first Phase 5 category. Sonarr, Radarr, Sabnzbd, Prowlarr and Overseerr as bjw-s `app-template` 5.3.0 HelmReleases under `apps/`, with Gateway routes, kept Longhorn config volumes, the shared `media` claim mounted at `/media`, a Cilium network policy for the `media` namespace, and PSA `baseline` on that namespace. Written [Move the media apps from Docker](/runbooks/media-migration.md). Apps are the user's current Docker set rather than the plan's list (Prowlarr for Jackett, Overseerr for Jellyseerr). Also folds in the earlier record that the PSA labels were applied. Also staged an in-cluster `cloudflared` (`apps/home/cloudflared/`, not enabled until the tunnel token exists) for the one public hostname `requests` (Overseerr; Radarr, Sonarr, Sabnzbd and Prowlarr stay LAN-only), and dropped Portainer from the plan. See [Phase 5](/phases/phase-5-applications.md) and [Exposure](/decisions/exposure-access.md).

## 2026-11-03 (PSA applied)
* **Update**: Merged PR #3 and Flux applied the PSA `enforce: restricted` labels on `cert-manager`, `gateway` and `flux-system`. Read-only check afterwards: all Kustomizations Ready on the new revision, every HelmRelease Ready, no pod outside Running/Completed. `apps` briefly showed "dependency not ready" while `infrastructure-configs` reconciled, as it did after the earlier merges. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-03 (security decision)
* **Update**: The user chose the small hardening plan. Wrote PSA `enforce: restricted` labels for `cert-manager`, `gateway` and `flux-system` (the last through a patch in `clusters/home/flux-system/kustomization.yaml`); `kustomize build` and `yamllint` pass and a server-side dry-run showed no warnings. Not pushed or applied. `default` is unmanaged in Git and stays unlabelled. Network policies come per app namespace in Phase 5. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-03 (NFS applied)
* **Update**: Merged PR #1 and Flux applied the `media` namespace, PV `media-nfs` and claim `media/media` (Bound). A throwaway pod running as 568:568 mounted it from node-3, listed the library, created and deleted a file, and was removed. Only node-3 was tested. See [Phase 3](/phases/phase-3-storage.md).

## 2026-11-03
* **Update**: TrueNAS media export is live: `192.168.15.24:/mnt/Poolio/media`, owner 568:568 (Mapall), mounted by the Docker host and the Plex box. Added `infrastructure/configs/storage/` (namespace `media`, NFS `PersistentVolume` `media-nfs`, claim `media`); `kustomize build` and `yamllint` pass; not pushed or applied. Backups deferred until apps exist, `k8s` pool noted as the candidate. See [Storage](/decisions/storage.md) and [Phase 3](/phases/phase-3-storage.md).

## 2026-11-02
* **Update**: Handoff for resuming on another host: added [Current status and handoff](/status.md), recorded the Pod Security dry-run results and the pending hardening decision in [Phase 4](/phases/phase-4-observability-security.md), and listed the blocked items in [Open questions](/open-questions.md). Nothing was changed on the cluster.

## 2026-11-01 (logging live)
* **Update**: Loki and Alloy are running and pod logs from five namespaces are queryable in Loki. First finding: steady kube-apiserver gRPC dial-cancel warnings to local etcd, probably benign. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (logging)
* **Update**: Added Loki and Grafana Alloy HelmReleases plus a Grafana Loki datasource. Used the `grafana-community` Loki chart because the OSS chart moved there. Rendering the chart locally caught that the simple-scalable replicas must be zeroed. Not yet pushed or applied. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (Longhorn metrics)
* **Update**: Added a Longhorn `ServiceMonitor` in `infrastructure/configs/monitoring/`. It is defined there instead of through the chart value so it applies after the monitoring CRDs exist. Not yet pushed. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (control-plane metrics)
* **Update**: Wrote `ansible/control-plane-metrics.yml` to expose etcd, scheduler and controller-manager metrics, added the matching `extraArgs` to `kubeadm/kubeadm-config.yaml` and enabled their scrapes in the Prometheus HelmRelease. Run on node-1..3 without failures and verified reachable over the node IPs; pushed, and Prometheus shows all control-plane targets up. The stored `kubeadm-config` ConfigMap has the new args. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (monitoring live)
* **Update**: Flux applied `kube-prometheus-stack`; Prometheus and Grafana run on Longhorn and Grafana answers at `grafana.home.mkenkel.com` through the Gateway with a verified production Let's Encrypt wildcard cert. All five Flux Kustomizations are Ready. Ticked the applied items in [Phase 2](/phases/phase-2-networking.md) and [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (Phase 4 start)
* **Update**: Added `kube-prometheus-stack` 92.17.0 (HelmRelease, SOPS-encrypted Grafana admin secret, Grafana HTTPRoute). Not yet pushed or applied. etcd, scheduler and controller-manager scrapes are disabled until their metrics are exposed. See [Phase 4](/phases/phase-4-observability-security.md).

## 2026-11-01 (Longhorn live)
* **Update**: Flux installed Longhorn v1.13.0; the HelmRelease is Ready and all pods in `longhorn-system` are Running. Default StorageClass is `longhorn`. A 1Gi test PVC and pod worked (volume healthy, 3 replicas) and were deleted. The Cilium release stayed at revision 4, so there is no repeated-upgrade loop. See [Phase 3](/phases/phase-3-storage.md).

## 2026-11-01 (Phase 3 start)
* **Update**: Nodes have one 1TB NVMe each (no second disk), so Longhorn uses the root filesystem with reserve limits; recorded in [Storage](/decisions/storage.md). Added the Longhorn v1.13.0 `HelmRelease`; not yet pushed or applied. NFS media volume waits for the TrueNAS details.

## 2026-11-01 (Phase 2 live)
* **Update**: Flux applied the L2 pool, Gateway (Programmed at 192.168.15.100), cert-manager and both issuers; the staging wildcard certificate issued cleanly, so the Certificate now uses `letsencrypt-prod`. See [Phase 2](/phases/phase-2-networking.md).

## 2026-11-01 (issuers)
* **Update**: Added the Let's Encrypt staging and prod ClusterIssuers, the `*.home.mkenkel.com` Certificate (staging) and the HTTPS Gateway listener. Not yet pushed or applied. See [Phase 2](/phases/phase-2-networking.md).

## 2026-11-01 (domain and cert-manager)
* **Update**: Domain is `mkenkel.com` on Cloudflare; private apps go under `*.home.mkenkel.com`. Recorded the existing DNS records to avoid clobbering them in [Exposure and access](/decisions/exposure-access.md). Added a cert-manager `HelmRelease` (chart v1.21.3). Issuers wait for the ACME email and Cloudflare token.

## 2026-11-01 (Phase 2 start)
* **Update**: Added `infrastructure/configs/networking/` with the L2 pool (.100-.200), L2 announcement policy and an HTTP-only Gateway at .100. Not yet pushed or applied. DNS provider is Cloudflare; domain still to be chosen. See [Phase 2](/phases/phase-2-networking.md).

## 2026-11-01 (Flux is live)
* **Update**: Bootstrapped Flux and created the `sops-age` secret. Flux owns the Gateway API CRDs and adopted the hand-installed Cilium (HelmRelease Ready at 1.20.4, release revision 2). See [Phase 1](/phases/phase-1-bootstrap.md).

## 2026-11-01 (CNI path fix)
* **Update**: coredns and Hubble were stuck because containerd looked for CNI plugins in `/usr/lib/cni` and Cilium installs to `/opt/cni/bin`. Fixed in the `containerd` role; `kube-system` is all Running. See [Phase 1](/phases/phase-1-bootstrap.md).

## 2026-11-01 (cluster is up)
* **Update**: Ran `ansible/cluster.yml`; node-1..4 are Ready on v1.36.6 with Cilium. Ticked the kubeadm, CRD and Cilium items in [Phase 1](/phases/phase-1-bootstrap.md). Flux bootstrap and the SOPS secret remain.

## 2026-11-01 (Phase 1 automation)
* **Update**: Added `ansible/cluster.yml` and the `control_plane`, `worker` and `cni` roles to automate kube-vip, kubeadm init/join, Gateway API CRDs and Cilium (Helm pinned to 3.x). Written and linted only; not run against the cluster. Flux bootstrap stays manual.

## 2026-11-01 (Phase 1 prep)
* **Update**: Prepared the Phase 1 repo files: `kubeadm/` config and runbook, pinned Cilium (1.20.4) and Gateway API (v1.6.3) versions, Flux Kustomizations and the Cilium `HelmRelease`. Nothing applied to the cluster. Details in [Phase 1](/phases/phase-1-bootstrap.md).

## 2026-11-01 (nodes prepped)
* **Update**: Ran `site.yml` on node-1..4; the second run changed nothing. Fixed bad node clocks (chrony in the `common` role and preseed), the multipathd handler, and a stale apt-lists case in the `kubernetes` role. Ticked node install and Ansible in [Phase 0](/phases/phase-0-provisioning.md).

## 2026-11-01 (end of session)
* **Update**: Added `preseed/custom.ipxe` and the `custom_url` netboot hook; rewrote the remaining-steps list in [Phase 0](/phases/phase-0-provisioning.md) with the current state and unverified items.

## 2026-11-01 (later)
* **Update**: Merged the repo skeleton, CI, preseed and Ansible work; ticked the repo-side items in [Phase 0](/phases/phase-0-provisioning.md). Hostnames are `node-1`..`node-4` (the OPNsense hostname), not `kube-node-N`.

## 2026-11-01
* **Update**: Recorded node addressing, VIP (.6) and LB pool (.100-.200) in [cluster topology](/decisions/cluster-topology.md); preseed switched from static to DHCP networking in [Phase 0](/phases/phase-0-provisioning.md).

## 2026-10-27
* **Creation**: Established the bundle from `deployment-plan.md` and `plan.md`; all concepts are `generated` by Claude and unverified until a human confirms them.
