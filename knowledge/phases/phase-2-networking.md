---
type: Deployment Phase
title: "Phase 2: Networking"
description: L2 announcements and IP pool, Gateway API ingress, cert-manager, Hubble.
tags: [phase, cilium, cert-manager]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 2
---

All of this is [repo] work applied by Flux after [Phase 1](/phases/phase-1-bootstrap.md).

- [ ] `CiliumL2AnnouncementPolicy` and `CiliumLoadBalancerIPPool` (range from Phase 0)
- [ ] Gateway + GatewayClass (Cilium Envoy)
- [ ] cert-manager and a DNS-01 `ClusterIssuer`; needs a DNS provider API credential (SOPS-encrypted) from the user
- [ ] Hubble enabled
- [ ] external-dns (optional)
- [ ] BGP: deferred, see [networking](/decisions/networking.md)
