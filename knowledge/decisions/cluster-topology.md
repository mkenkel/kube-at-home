---
type: Decision
title: Cluster topology
description: 3 untainted control-plane nodes plus 1 worker on kubeadm, API endpoint via kube-vip in ARP mode.
tags: [kubeadm, ha, nodes]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, sections 2 and 3
---

* Option A chosen: 3 control planes, untainted so they run workloads, plus 1 worker, for an HA etcd quorum. Confirmed by the user, who is using the 4 spare nodes they have.

# Addressing (192.168.15.0/24, OPNsense "Servers" interface)

| Role | Host | IP |
|---|---|---|
| control plane | node-1 | 192.168.15.7 |
| control plane | node-2 | 192.168.15.8 |
| control plane | node-3 | 192.168.15.9 |
| worker | node-4 | 192.168.15.10 |
| control-plane VIP (kube-vip, ARP) | none | 192.168.15.6 |
| Cilium LoadBalancer pool (L2) | none | 192.168.15.100 - 192.168.15.200 |

* Nodes get their addresses from OPNsense DHCP static mappings, so the preseed uses DHCP and takes the hostname from it.
* The VIP and LB pool are announced over ARP and need no OPNsense entries. They must stay outside the dynamic DHCP range (192.168.15.201-254) and clear of static mappings (currently up to .44).
* Debian nodes, kubeadm + containerd. OS and etcd on NVMe; second M.2 or 2.5" bay for storage disks.
* kube-proxy skipped (`--skip-phases=addon/kube-proxy`); Cilium replaces it.
* Control-plane endpoint is a kube-vip VIP (ARP mode, static pod), reserved outside DHCP.
* Jellyfin/Plex and Frigate run on dedicated boxes outside the cluster; no GPU/Coral passthrough needed.
