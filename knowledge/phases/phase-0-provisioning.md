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

Still to do by hand before Phase 1:

- [ ] [human] Put your real public SSH key in `preseed/preseed.cfg` (the install fails on purpose while the TODO placeholder is there)
- [ ] [human] Replace the placeholder age recipient in `.sops.yaml` once the age key exists
- [ ] [human] Pin the Cilium chart and Gateway API CRD versions (`cilium/values.yaml` TODOs)
- [ ] [human] Run the Ansible playbook against the nodes after install: add host keys first (`host_key_checking` is on)

Notes: the Kubernetes pin is 1.36.6-1.1 (checked 2026-11-01; latest minor then was 1.37). Ansible roles are our own, kept small; see `ansible/`. The preseed installs `python3-debian` because the `kubernetes` role uses `deb822_repository`.

Next: [Phase 1](/phases/phase-1-bootstrap.md).
