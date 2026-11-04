---
type: Decision
title: Exposure and access
description: Private by default (LAN + WireGuard); only Jellyseerr and Wizarr go public via Cloudflare Tunnel with Cloudflare Access.
tags: [cloudflare, access, wireguard, security]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, section 4
---

* Cilium Gateway API is the in-cluster reverse proxy; `cloudflared` in-cluster points at the Gateway Service.
* Authentik is dropped. Private apps keep local logins.
* Cloudflare Access fronts public apps. Caveats: it only covers tunnel traffic, mobile/API clients need WireGuard or service tokens, Wizarr invite paths need a bypass, and never proxy media streaming through the tunnel.
* Cilium NetworkPolicies keep exposed apps from reaching the rest of the cluster.
* Hubble UI is on the LAN at `hubble.home.mkenkel.com` (2026-11-04). It has no login and shows every flow, pod name and DNS lookup, read only, and the Gateway cannot filter by source IP, so reaching it is as open as the LAN and WireGuard. The Longhorn UI stays unexposed because it can change volumes. Neither is for the tunnel.
* Glance (home page with service checks) is on the LAN at `glance.home.mkenkel.com`, no login; it only shows which services are up. Glance supports a login if that is ever wanted.
* 2026-11-03: the public set is Seerr (`requests`, the successor of the archived Overseerr) only, instead of Jellyseerr and Wizarr (neither is deployed). Radarr, Sonarr, Sabnzbd and Prowlarr stay private: LAN and WireGuard only, as little exposure as possible. The in-cluster tunnel points straight at the Overseerr Service (`http://overseerr.media.svc.cluster.local:5055`) rather than at the Gateway, because the Gateway only serves `*.home.mkenkel.com`. The old `radarr` and `sabnzbd` hostnames on the Docker tunnel are not to be moved; the user removes them in Cloudflare when those Docker apps are stopped (the repo does not touch existing records).

# Domain and DNS

* Domain `mkenkel.com`, DNS hosted on Cloudflare. Private apps live under `*.home.mkenkel.com` (wildcard cert via Cloudflare DNS-01); the Gateway is at 192.168.15.100. Internal names get no public DNS records, only an OPNsense Unbound override for `*.home.mkenkel.com` pointing at the Gateway.
* Existing records (screenshot 2026-11-01), none of which this repo may change: `mkenkel.com` A, `tesira` A 192.168.15.35, tunnel CNAMEs `deebo` (Deebo tunnel), `ha` (homeassistant tunnel), `docs`, `radarr`, `requests`, `sabnzbd`, `tabs`, `test` (`homelab-tunnel`), and the `_minecraft._tcp` SRV record.
* The cert-manager Cloudflare token needs Zone:DNS:Edit and Zone:Zone:Read and cannot be limited to a subdomain, so it is SOPS-encrypted and scoped to the one zone. Moving a tunnel hostname to in-cluster `cloudflared` is a deliberate per-name cutover. If external-dns is added, restrict it to `home.mkenkel.com` with no sync policy.
