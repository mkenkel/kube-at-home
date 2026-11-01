---
type: Deployment Phase
title: "Phase 0: Node provisioning"
description: Preseed install, Ansible node prep, network reservations, age key and repo skeleton.
tags: [phase, ansible, preseed]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 0
---

Legend: **[repo]** an agent can do it by editing files; **[human]** needs hardware, secrets or the network.

- [x] [repo] Repo skeleton: `clusters/home/`, `infrastructure/{controllers,configs}/`, `apps/{base,home}/`, `ansible/`, `preseed/`, `cilium/values.yaml`
- [x] [repo] `preseed/` file: disk layout, user, SSH key, DHCP networking with the hostname taken from DHCP (no hardcoded hostname)
- [ ] [human] Custom netboot.xyz menu entry pointing at the preseed URL (netboot.xyz is hosted outside the cluster)
- [x] [repo] `ansible/` playbook: `open-iscsi`/`iscsid`, `nfs-common`, multipath blacklist, containerd, kubeadm/kubelet/kubectl, kernel modules, sysctls
- [x] DHCP static mappings for node-1..4 exist (.7-.10; the OPNsense hostname is node-N, the description is kube-node-N); VIP .6 and LB pool .100-.200 chosen, see [topology](/decisions/cluster-topology.md)
- [ ] [human] Click "Apply changes" on the OPNsense DHCP page so the reservations are live before the first netboot
- [ ] [human] Generate the age key and store it in the password manager. Never commit it.

## Current state (2026-11-01)

Repo side is merged and committed: skeleton, preseed (real SSH key in, timezone America/Indianapolis, DHCP, hostname from DHCP), Ansible roles, CI, `preseed/custom.ipxe`. Hostnames are `node-1`..`node-4` (OPNsense hostname), not `kube-node-N`.

netboot.xyz runs in Docker on `dockery` (192.168.15.25), version 2.0.88, assets/nginx on port 8081, web app on 3000. The stock `menu.ipxe` has no menu item for a Menus-tab `custom.ipxe`; the only hook is `custom_url`, so `preseed/README.md` uses that.

Remaining, in order:

- [ ] [human] OPNsense DHCP page: click "Apply changes" so the `node-N` reservations are live
- [ ] [human] Upload `preseed/preseed.cfg` to the netboot.xyz assets dir as `assets/preseed/preseed.cfg`
- [ ] [human] Paste `preseed/custom.ipxe` into the web app Menus as `custom.ipxe`; add `set custom_url http://192.168.15.25:8081` to `boot.cfg` under `:global_vars`
- [ ] [human] Verify: `curl -sI http://192.168.15.25:8081/custom.ipxe` and `.../preseed/preseed.cfg` both return 200 (assumes nginx serves menus and assets at the web root; unverified)
- [ ] [human] Netboot node-1: "Custom URL Menu" appears, installer fetches the preseed, hostname is `node-1` (not `unassigned-hostname`), boots with no monitor. If it hangs headless, add `video=HDMI-A-1:e` to GRUB (check connector name in `/sys/class/drm/`). Also confirm the OS disk is `/dev/nvme0n1`; the install wipes it.
- [x] [human] Install node-2..4
- [x] [human] Add host keys, then `ansible-playbook site.yml` from `ansible/` (run twice; second run should change nothing). Verified 2026-11-01: second run `changed=0` on all four nodes.
- [ ] [human] `age-keygen`, private key into the password manager, public recipient into `.sops.yaml`
- [ ] [repo/human] Pin Cilium chart and Gateway API CRD versions in `cilium/values.yaml` (can wait until Phase 1)

Found during the first Ansible run: node RTCs can be wrong (node-2 months behind, node-3 4h behind), which breaks TLS and apt. The `common` role now runs chrony and steps the clock first; a rerun after a failed run also refreshes apt lists when `kubelet` has no candidate. The multipathd restart handler tolerates a missing unit.

Unverified: `initrd=initrd.magic` on the kernel line, the "Return to Main Menu" `exit`, whether Debian's `containerd` package ships a default config (the role handles both), and the CI workflow (has not run yet).

Notes: Kubernetes pin is 1.36.6-1.1 (checked 2026-11-01; latest minor then was 1.37). Ansible roles are our own and kept small. The preseed installs `python3-debian` for the `deb822_repository` module; `nfs-common` comes from Ansible, not the preseed.

Next: [Phase 1](/phases/phase-1-bootstrap.md).
