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
- [ ] Prepared, not yet applied: Gateway `gateway/home` at 192.168.15.100 (GatewayClass `cilium` comes from the Cilium chart) with HTTP and HTTPS (`*.home.mkenkel.com`) listeners
- [ ] Prepared, not yet applied: cert-manager v1.21.3, `letsencrypt-staging` and `letsencrypt-prod` ClusterIssuers (Cloudflare DNS-01, SOPS-encrypted token), wildcard Certificate issued from staging. DNS-01 self-checks use 1.1.1.1/8.8.8.8 because of the planned Unbound split-horizon override. After staging issues cleanly, switch `networking/certificate.yaml` to `letsencrypt-prod`.
- [ ] [human] OPNsense: Unbound host override `*.home.mkenkel.com` to 192.168.15.100; allow nodes outbound port 53 to 1.1.1.1/8.8.8.8 if blocked
- [x] Hubble enabled (in `cilium/values.yaml`; pods Running)
- [ ] external-dns (optional)
- [ ] BGP: deferred, see [networking](/decisions/networking.md)
