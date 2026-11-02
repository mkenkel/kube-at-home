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

- [ ] Prepared 2026-11-01, not yet applied: `CiliumL2AnnouncementPolicy` and `CiliumLoadBalancerIPPool` (range from Phase 0)
- [ ] Prepared, HTTP only: Gateway `gateway/home` at 192.168.15.100 (GatewayClass `cilium` comes from the Cilium chart); HTTPS listener waits for the domain and cert-manager
- [ ] cert-manager and a DNS-01 `ClusterIssuer` for Cloudflare; needs the domain and a Cloudflare API token (SOPS-encrypted) from the user
- [x] Hubble enabled (in `cilium/values.yaml`; pods Running)
- [ ] external-dns (optional)
- [ ] BGP: deferred, see [networking](/decisions/networking.md)
