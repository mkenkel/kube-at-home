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

- [ ] [repo] Repo skeleton: `clusters/home/`, `infrastructure/{controllers,configs}/`, `apps/{base,home}/`, `ansible/`, `preseed/`, `cilium/values.yaml`
- [ ] [repo] `preseed/` file: disk layout, user, SSH key, DHCP networking with the hostname taken from DHCP (no hardcoded hostname)
- [ ] [human] Custom netboot.xyz menu entry pointing at the preseed URL (netboot.xyz is hosted outside the cluster)
- [ ] [repo] `ansible/` playbook: `open-iscsi`/`iscsid`, `nfs-common`, multipath blacklist, containerd, kubeadm/kubelet/kubectl, kernel modules, sysctls
- [x] DHCP static mappings for kube-node-1..4 exist (.7-.10); VIP .6 and LB pool .100-.200 chosen, see [topology](/decisions/cluster-topology.md)
- [ ] [human] Click "Apply changes" on the OPNsense DHCP page so the reservations are live before the first netboot
- [ ] [human] Generate the age key and store it in the password manager. Never commit it.

Next: [Phase 1](/phases/phase-1-bootstrap.md).
