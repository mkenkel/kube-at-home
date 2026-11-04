# Deployment Plan

Companion to `plan.md`. Covers *how* things get onto the cluster, in what order, and how to test them before they touch the 4x M70q Gen2 nodes.

## 1. GitOps: Flux vs ArgoCD

| | Flux | ArgoCD |
|---|---|---|
| Footprint | Small (a few controllers, a few hundred MB) | Heavier (server, repo-server, redis, app-controller; ~1 GB+) |
| UI | None built in (Headlamp has a Flux plugin) | Good built-in UI, useful for "why is this out of sync" |
| Helm | `HelmRelease` CRD, native, values live in Git | `Application` renders Helm; fine, slightly less Git-native |
| "List all deploys in a file" | `Kustomization` tree (`apps/` → one dir per app) | App-of-apps or `ApplicationSet` generators |
| Secrets | SOPS + age built in | Needs a plugin (ksops, argocd-vault-plugin) or External Secrets |
| Multi-cluster / RBAC / SSO | Fine, less polished | Stronger, and SSO works with Authentik |

**Recommendation: Flux.** Four small nodes running ~40 self-hosted apps is the use case Flux fits best: low overhead, everything is a CRD in Git, and SOPS/age gives encrypted secrets in the repo with no extra tooling. The `HelmRelease` + Kustomize layout also answers the "single place listing all deploys" note in `plan.md`.

Pick ArgoCD instead if you want the UI, or want it on your résumé (it's very common in industry). Migrating later is not painful, because the manifests and Helm values are the same either way.

### Suggested repo layout (Flux)

```
clusters/home/            # Flux entrypoint (flux bootstrap writes flux-system/ here)
  flux-system/
  infrastructure.yaml     # Kustomization -> infrastructure/
  apps.yaml               # Kustomization -> apps/ (dependsOn: infrastructure)
infrastructure/
  controllers/            # cilium, cert-manager, storage, external-secrets, ...
  configs/                # IP pools, ClusterIssuers, StorageClasses
apps/
  base/<app>/             # HelmRelease + values
  home/<category>/        # games, media, home-automation, ...
```

Use `dependsOn` so CNI → storage → cert-manager → apps always apply in order.

## 2. Cluster layout (4x M70q Gen2)

- Existing install: kubeadm + containerd (per `plan.md`).
- **Option A (recommended):** 3 control-plane nodes (untainted, so they also run workloads) + 1 worker. Gives an HA etcd quorum.
- **Option B:** 1 control plane + 3 workers. Simpler, but the control plane is a single point of failure.
- etcd wants fast disk: put the OS and etcd on the NVMe. Use the second M.2 slot or the 2.5" bay for storage-backend disks.
- RAM is 32 GB per node, which is plenty for the app list (Prometheus included).
- Jellyfin/Plex and Frigate run on dedicated boxes outside the cluster, so nothing here needs GPU or Coral passthrough.

## 3. Phased rollout

### Phase 0: Node provisioning
Nodes run Debian. Two layers:
- [ ] **netboot.xyz + preseed (minimal install):** disk layout, user, SSH key, static network and nothing else. Add a custom netboot.xyz menu entry that passes the preseed URL (keep the preseed file in this repo). netboot.xyz is already hosted outside the cluster, so it is not deployed here.
- [ ] **Ansible (everything else, re-runnable):** `open-iscsi` + `iscsid`, `nfs-common`, multipath blacklist for Longhorn, containerd, kubeadm/kubelet/kubectl, kernel modules and sysctls. Keep the playbook in this repo.
- [ ] DHCP reservations for all nodes; pick a LoadBalancer IP range and a control-plane VIP outside DHCP.
- [ ] Generate an age key; back it up somewhere off-cluster (password manager).
- [ ] Create the repo layout from section 1 (`clusters/`, `infrastructure/`, `apps/`) in the GitHub repo, plus `ansible/`, `preseed/` and `cilium/values.yaml` directories for the manual pieces.

### Phase 1: Cluster init and GitOps bootstrap
This is a net-new cluster: nothing is installed yet, so there is nothing to reset.
- [ ] **Control-plane endpoint: kube-vip in ARP mode** (static pod on the control-plane nodes, VIP reserved in Phase 0). A DNS name round-robining the three nodes is the fallback, but it fails over poorly.
- [ ] `kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint=<VIP>:6443 --upload-certs` on the first control-plane node. Join the other two control-plane nodes and the worker.
- [ ] **Install the Gateway API CRDs by hand** (manual, once), before Cilium: Cilium's Gateway API support needs them present when it starts. Pin the version in the repo, and have Flux own them afterwards.
- [ ] **Install Cilium by hand with Helm** (manual, once): `kubeProxyReplacement=true`, `k8sServiceHost`/`k8sServicePort` set to the control-plane endpoint, L2 announcements and Gateway API enabled (`gatewayAPI.enabled=true`). Keep the values file in the repo. Nodes stay `NotReady` until this is done, and Flux cannot run pods before it.
- [ ] `flux bootstrap github` against the repo, then create the SOPS age secret in the cluster (manual).
- [ ] Commit a Cilium `HelmRelease` with the same release name, namespace and values, so Flux adopts the manual install.
- [ ] Add Renovate (or Dependabot) to keep chart versions current via PRs. This covers most of the CI/CD section.

Everything after this goes through Git in dependency order: cert-manager, Longhorn, NFS storage for media, `cloudflared`, Prometheus/Grafana, then apps.

### Phase 2: Networking (Cilium)
- [ ] kube-proxy is replaced by Cilium (done in Phase 1).
- [ ] L2 announcements + `CiliumLoadBalancerIPPool` for LoadBalancer IPs. Start here; it needs no router support.
- [ ] Gateway API (Cilium's Envoy-based implementation) as ingress. This is probably the "L7... thing" in `plan.md`.
- [ ] **Decision: L2 now, BGP later.** Service IPs use L2 announcements and the control plane uses kube-vip in ARP mode. Traffic is a handful of light services, so L2's single-node-per-IP limit does not matter.
- [ ] BGP (deferred): OPNsense can do it via `os-frr` (OPNsense config is backed up to GDrive). To add later: install `os-frr`, use private ASNs, add each node as a neighbor, allow TCP 179 from the node subnet, and give Cilium a BGP peering config plus a separate routed IP pool (not the L2 pool, and never announce the same IPs both ways). Verify ECMP actually spreads traffic on your OPNsense version before relying on it.
- [ ] Hubble for flow visibility.
- [ ] cert-manager with a DNS-01 ClusterIssuer (Let's Encrypt) so internal services get real certs.
- [ ] external-dns (optional) if your DNS server supports it.

### Phase 3: Storage
Options for 4 nodes with 1 GbE:
- **Longhorn:** replicated block storage, easy on small clusters, built-in S3/NFS backups. Good default.
- **Rook-Ceph:** more capable but wants dedicated disks and fast networking. Likely overkill on 1 GbE.
- **local-path / OpenEBS LocalPV:** fastest and simplest, but data is pinned to one node. Fine for apps you can rebuild.
- **NFS (from a NAS):** best fit for bulk media (the *arr libraries).

**Decision:** media lives on the existing TrueNAS box, mounted over NFS (the *arr stack and download client share one media volume with consistent UID/GID). Longhorn only needs to hold app configs and databases. Longhorn is chosen, partly for flexibility if more stateful apps are added later.

### Phase 4: Observability and security
- [ ] `kube-prometheus-stack` (covers Prometheus + Grafana from the app list).
- [ ] Loki + Grafana Alloy for logs (Promtail is deprecated).
- [ ] Cilium NetworkPolicies and Pod Security Admission (`baseline`/`restricted` namespaces). Authentik is dropped for now (see section 4); revisit if local accounts get unwieldy.
- [ ] **Backups: two points (offsite leg deferred).**
  - Local: Longhorn recurring backups + Velero (cluster state) to a dedicated TrueNAS dataset over NFS (or S3 via MinIO). Fast restore path.
  - Offsite: Duplicati to a cloud provider. What it backs up and where it uploads is not planned yet; revisit after everything else is settled. It does not point at TrueNAS.
  - Git (GitHub) backs up manifests. The SOPS age key lives in the password manager. Duplicati's passphrase goes there too once it is set up.
  - Media is treated as replaceable and covered by TrueNAS itself.
  - Do a test restore of the local backups at least quarterly.

### Phase 5: Applications
Roll out one category at a time, one PR each, in this order (lowest risk first):
1. Administration (Glance, Gitea, Duplicati, last, once its backup plan exists; Netboot.xyz is hosted separately and Authentik is dropped)
2. Notetaking: nothing deployed here (Bookstack, Quartz and Perlite are dropped).
3. Media (the *arr stack, Jellyseerr, Wizarr, Notifiarr, Tautulli if you use Plex). They share the TrueNAS media volume and need consistent UID/GID. The media server itself (Jellyfin/Plex) is on a dedicated box; Jellyseerr, Wizarr and Tautulli just point at it, so it needs to be reachable from the cluster.
4. File management: nothing deployed here (Nextcloud, Immich, Syncthing and 0x0.st are dropped).
5. Home automation: nothing deployed here. Home Assistant (HAOS) and Frigate run on their own machines, and Homebridge and Grocy are dropped.

Game servers are out of scope.

The app list on the media side is mostly the same shape (one container, one config PVC, one ingress). A shared `app-template` Helm chart (e.g. bjw-s common library) avoids repeating yourself.

## 4. Exposure and access

Cilium (Gateway API) is the in-cluster reverse proxy. Cloudflare Tunnel is only the path from the internet to it.

- **Private (default):** LAN + the existing WireGuard on the router. Covers Gitea, Grafana and so on; Jellyfin/Plex streaming is handled by its own box. These apps keep their own local logins.
- **Public via Cloudflare Tunnel:** only a handful, Overseerr (`requests`), Radarr and Sabnzbd, which already have public hostnames, plus Wizarr if it is deployed. `cloudflared` runs in the cluster and points at the Gateway Service.
- **Auth for public apps:** Cloudflare Access (free up to 50 users) in front, instead of Authentik. Caveats:
  - It only protects tunnel traffic; LAN/WireGuard access hits the apps directly.
  - Mobile/API clients (mobile apps) can't do the Access login without service tokens or bypass rules, so use WireGuard for those.
  - Wizarr invite links need a bypass for the invite path, or invitees get an extra email-PIN step.
  - Don't proxy media streaming through the tunnel (Cloudflare's terms). Jellyseerr only handles requests, not streams.
- **Certificates:** cert-manager with DNS-01 for internal TLS.
- **Segmentation:** Cilium NetworkPolicies so exposed apps cannot reach the rest of the cluster.

## 5. Testing: can you do it without more hardware?

**Short answer: mostly yes. The 4 M70qs are enough to run it, and you don't need to buy anything to test most of it.** Hardware you may want later is optional (below).

### Local test options (on your workstation)

| Tool | Good for | Limits |
|---|---|---|
| **kind / k3d** | Flux bootstrap against a local Git repo, HelmRelease validity, Kustomize layout, dependency ordering, SOPS, app manifests | Not kubeadm, no real L2/ARP, no real disks, so storage and L2 announcements can't be tested faithfully |
| **KVM/libvirt VMs (4 VMs, kubeadm)** | Closest to the real thing: the same kubeadm install, Cilium, Longhorn on virtual disks, node-failure drills | Uses a lot of RAM; a bridged network is needed for L2 tests |
| **containerlab + FRR** | Practising Cilium BGP peering against a virtual router | Only relevant if you go the BGP route |

Suggested workflow:
1. Develop manifests and the repo layout in **kind** (fast to create/destroy).
2. Rehearse the **kubeadm + Cilium + Longhorn** parts in 3-4 VMs. Skip this if you're comfortable testing those directly on the real cluster.
3. Use `flux diff` / `kustomize build` / `kubeconform` in CI on every PR so mistakes are caught before they reach the cluster.
4. On the real cluster, change things via PR only. Revert with `git revert`.

Because the hardware is spare-able, another option is to **wipe one M70q to be a staging node** (single-node kubeadm), but that costs you the HA layout, so the VMs are likely the better choice.

### What you'd actually need to buy (all optional)

- **Smart/VLAN-capable switch** if you want IoT/camera isolation for the Frigate and Home Assistant boxes.
- **Off-cluster backup target:** TrueNAS already covers media; point Longhorn/Velero backups at it too (NFS or S3 via MinIO), ideally on a separate dataset.
- **UPS** to protect etcd and the disks from hard power loss.
- **2.5 GbE USB NICs** are not needed unless Longhorn replication saturates 1 GbE.

## 6. Open questions

Decided: GitHub is the Git source of truth (Gitea in the cluster is just an app), L2 now with BGP deferred, Longhorn plus TrueNAS NFS for storage, Plex on the media box (so Tautulli stays), and only Jellyseerr and Wizarr go through the tunnel.

- Which dataset or share on TrueNAS will hold the local Longhorn/Velero backups?
- Duplicati: what to back up and where to (deferred until the rest is settled).
