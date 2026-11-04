---
type: Runbook
title: Alerts and notifications (ntfy)
description: How alerts reach the phone, what is alerted on, how to set up the iPhone, how to test, and how to add rules.
tags: [runbook, alerting, ntfy, alertmanager, monitoring]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: ntfy-config
    resource: https://docs.ntfy.sh/config/
    title: ntfy server configuration
  - id: ntfy-publish
    resource: https://docs.ntfy.sh/publish/
    title: ntfy publishing, including message templating
---

# How it fits together

Prometheus evaluates rules, sends firing alerts to Alertmanager, and Alertmanager posts them to **ntfy** (a small self-hosted notification server) which pushes them to the iPhone app.

* ntfy runs in the `ntfy` namespace (`apps/base/ntfy`, `apps/home/administration/`) at `https://ntfy.home.mkenkel.com`, LAN only, through the Gateway. Auth is on and everything is denied by default.
* Two users, declared in config (the password hashes and the token are in the SOPS secret `ntfy-auth`): `alertmanager`, which may only publish to the topic `alerts` and authenticates with a token, and `phone`, which may only read `alerts`.
* Alertmanager (in `monitoring`) uses the built-in ntfy template `alertmanager` for firing and resolved alerts, and the token from the SOPS secret `alertmanager-ntfy`. The always-firing `Watchdog` and `InfoInhibitor` are routed to a null receiver.
* iPhone: Apple only allows pushes through its own service, so ntfy has `upstream-base-url: https://ntfy.sh`. For every message ntfy.sh receives only the message ID and a hash of the topic URL, never the text. The phone then fetches the message from `ntfy.home.mkenkel.com`, so **the phone must be able to reach that name** (home Wi-Fi or the router's WireGuard VPN with the router as DNS). Away from home without the VPN you get the wake-up but not the text.

# iPhone setup **[human]**

1. Install the ntfy app from the App Store.
2. Add a subscription: topic `alerts`, tick "Use another server", server `https://ntfy.home.mkenkel.com`, and log in with the user and password in `~/ntfy-phone-credentials.txt` (user `phone`).
3. Move that password into the password manager and delete the file.
4. Allow notifications for the app when iOS asks.

# Test the whole path

From a pod that can reach Alertmanager (port-forward works too):

`kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093`

`curl -s -XPOST localhost:9093/api/v2/alerts -H 'Content-Type: application/json' -d '[{"labels":{"alertname":"TestAlert","severity":"warning","namespace":"test"},"annotations":{"summary":"Test from the runbook","description":"If you can read this on your phone, alerts work."}}]'`

The phone should buzz within about 30 seconds (Alertmanager's `group_wait`) and show a "resolved" message about five minutes later when the alert expires.

# What is alerted on

* The default kube-prometheus-stack rules: nodes, kubelet, API server, etcd, pods crash-looping, node filesystem filling up, target down, and so on.
* Longhorn (`infrastructure/configs/monitoring/alerts-longhorn.yaml`): a config volume without a backup for 36 hours (or ever), a degraded volume for 20 minutes, a faulted volume, a Longhorn node not ready, and the Longhorn disk above 75% (warning) or 85% (critical). Each node has one NVMe that also holds etcd, so disk space matters.

* Flux (`alerts-flux.yaml`): a Kustomization, HelmRelease, GitRepository or HelmRepository not ready for 15 minutes (a sync normally goes unready for a minute or two while it waits for a dependency, so only a lasting state alerts), a resource suspended for an hour, a Flux controller with no available pod, and the Flux state metrics going missing (so the other Flux alerts cannot silently stop working). Flux's controllers expose no per-resource ready state, so `gotk_resource_info` comes from kube-state-metrics' custom resource state config (copied from fluxcd/flux2-monitoring-example for the four kinds used here, with the standard collectors left on, which that example turns off).
* Certificates (`alerts-cert-manager.yaml`, from the cert-manager ServiceMonitor): a certificate expiring within 21 days (cert-manager renews at 30, so this means renewal has been failing for over a week), a certificate not ready for 15 minutes, and cert-manager's metrics missing.

**Known gaps:** When the whole cluster or the LAN is down, nothing can notify you: Alertmanager lives inside the cluster. Critical alerts are not sent at a higher priority yet; ntfy's built-in template does not map severity to priority.

# Add or change a rule

Add a `PrometheusRule` with the label `release: kube-prometheus-stack` (the Prometheus only loads rules with that label), for example next to `alerts-longhorn.yaml`, and check it with `promtool check rules` before pushing. Severity is a label: `warning` or `critical`.

# Silence or mute

Port-forward Alertmanager as above and use its UI, or `amtool silence add alertname=Something`. Alertmanager repeats an unresolved alert every 12 hours.

# Rotate the ntfy credentials

Generate a new phone password and bcrypt hash and a new token (`tk_` plus 29 lowercase letters or digits), put them in `apps/home/administration/ntfy-auth.sops.yaml` (`NTFY_AUTH_USERS`, `NTFY_AUTH_TOKENS`) and the token in `infrastructure/controllers/monitoring/alertmanager-ntfy.sops.yaml`, re-encrypt with `sops`, push, then log in on the phone with the new password. Users removed from the config are deleted on the next ntfy restart.
