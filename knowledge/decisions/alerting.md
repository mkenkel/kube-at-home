---
type: Decision
title: Alerting
description: Alerts go from Prometheus through Alertmanager to a self-hosted ntfy on the LAN, read on an iPhone; what is and is not alerted on.
tags: [alerting, ntfy, alertmanager, monitoring]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 4
---

* Channel (chosen 2026-11-04): ntfy, self-hosted in the cluster, LAN only, auth on with default deny. The user wants nothing new exposed publicly; alerts reach the iPhone while it is on the LAN or the router's WireGuard VPN. Alternatives considered: public ntfy.sh with a secret topic (works anywhere but sends alert text through a third party), and self-hosted behind the tunnel (too many moving parts).
* iPhone push needs ntfy.sh as a wake-up relay (`upstream-base-url`); only the message ID and a hash of the topic URL are sent there.
* Rules: kube-prometheus-stack's defaults plus Longhorn rules (stale config backups, degraded or faulted volumes, node not ready, disk above 75% and 85%). Flux failures and certificate expiry are planned next.
* Not covered: a dead cluster or LAN cannot send alerts. A heartbeat to an external service would be needed for that and is not planned.
* Details and operations: [Alerts and notifications runbook](/runbooks/alerting.md).
