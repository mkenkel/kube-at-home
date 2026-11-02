# Update Log

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
