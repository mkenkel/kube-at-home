# Update Log

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
