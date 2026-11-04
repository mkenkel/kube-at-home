---
type: Decision
title: Networking
description: Cilium CNI with kube-proxy replacement, L2 announcements now, BGP deferred, Gateway API ingress.
tags: [cilium, l2, bgp, gateway-api]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phases 1 and 2
---

* Cilium with `kubeProxyReplacement=true`, Gateway API (`gatewayAPI.enabled=true`) and L2 announcements. Installed by hand once, then adopted by a Flux `HelmRelease` with the same release name, namespace and values.
* Gateway API CRDs are installed by hand before Cilium (pinned version), then owned by Flux.
* L2 now: `CiliumLoadBalancerIPPool` for service IPs. BGP deferred (OPNsense `os-frr`); when added, use a separate routed pool and never announce the same IPs both ways.
* cert-manager with a DNS-01 Let's Encrypt ClusterIssuer; Hubble for flow visibility.
* In-cluster callers get `Access denied` (HTTP 403 from Envoy) when they call a name served by the Gateway and their own egress policy does not allow the backend the Gateway forwards to. Found 2026-11-04 with Glance: Sonarr to Sonarr through the Gateway worked, Sonarr to Grafana did not, and a client outside the cluster worked. A pod that must reach another service should call its Service address directly, with both policies open for exactly that port, and not rely on the Gateway name from inside the cluster.
