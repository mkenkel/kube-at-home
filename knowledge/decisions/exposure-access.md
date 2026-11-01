---
type: Decision
title: Exposure and access
description: Private by default (LAN + WireGuard); only Jellyseerr and Wizarr go public via Cloudflare Tunnel with Cloudflare Access.
tags: [cloudflare, access, wireguard, security]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, section 4
---

* Cilium Gateway API is the in-cluster reverse proxy; `cloudflared` in-cluster points at the Gateway Service.
* Authentik is dropped. Private apps keep local logins.
* Cloudflare Access fronts public apps. Caveats: it only covers tunnel traffic, mobile/API clients need WireGuard or service tokens, Wizarr invite paths need a bypass, and never proxy media streaming through the tunnel.
* Cilium NetworkPolicies keep exposed apps from reaching the rest of the cluster.
