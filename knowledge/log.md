# Update Log

## 2026-11-01 (end of session)
* **Update**: Added `preseed/custom.ipxe` and the `custom_url` netboot hook; rewrote the remaining-steps list in [Phase 0](/phases/phase-0-provisioning.md) with the current state and unverified items.

## 2026-11-01 (later)
* **Update**: Merged the repo skeleton, CI, preseed and Ansible work; ticked the repo-side items in [Phase 0](/phases/phase-0-provisioning.md). Hostnames are `node-1`..`node-4` (the OPNsense hostname), not `kube-node-N`.

## 2026-11-01
* **Update**: Recorded node addressing, VIP (.6) and LB pool (.100-.200) in [cluster topology](/decisions/cluster-topology.md); preseed switched from static to DHCP networking in [Phase 0](/phases/phase-0-provisioning.md).

## 2026-10-27
* **Creation**: Established the bundle from `deployment-plan.md` and `plan.md`; all concepts are `generated` by Claude and unverified until a human confirms them.
