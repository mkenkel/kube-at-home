---
type: Runbook
title: Headlamp, the read-only cluster UI
description: What Headlamp can and cannot do here, how to log in with a short-lived token, how to check it is read only, and how to cut off a token early.
tags: [runbook, headlamp, dashboard, rbac]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: headlamp-chart
    resource: https://kubernetes-sigs.github.io/headlamp/
    title: Headlamp Helm chart (version 0.46.0)
---

# What it is

Headlamp is a web UI for Kubernetes at `https://headlamp.home.mkenkel.com`, on the LAN and reachable over WireGuard with the router as DNS. It is **read only**: you can browse workloads, logs, events, nodes, volumes, and the custom resources of Flux, Cilium, Longhorn, Prometheus, Gateway API and cert-manager. You cannot change anything and you cannot read Secrets.

# Log in

Headlamp asks for a token. Mint a short-lived one for the read-only account (this needs your own admin kubeconfig):

`kubectl -n headlamp create token headlamp-viewer --duration=8h`

Paste it into the login page. The Headlamp session lasts 4 hours (`sessionTTL`). Treat the token like a password: do not save it in a note or a password manager, mint a new one next time. If the API server refuses the requested duration it says so; use a shorter one.

# What protects the cluster

* **Two accounts, one is empty.** The pod's own ServiceAccount (`headlamp`) has **no** permissions. The chart binds it to `cluster-admin` by default; that is switched off with `clusterRoleBinding.create: false`. Your login token belongs to `headlamp-viewer`, and Headlamp sends each request to the API server with that token, so the server decides what you can do.
* **`headlamp-viewer` can only read.** It is bound to the built-in `view` role plus `headlamp-viewer-extra` (get, list and watch on nodes, volumes, CRDs, and the custom resources above). Neither grants Secrets, `pods/exec`, `nodes/proxy`, or any write. `view` does show pod specs and ConfigMaps, so do not put secrets in plain environment values or ConfigMaps.
* **Off on purpose:** the Helm plugin (`enableHelm`), OIDC, and the "use the pod's service account for every user" mode.
* **Network:** only the Gateway reaches the pod, and the pod can only reach DNS and the API server (`headlamp-isolation`).
* **Pod:** non-root, read-only root filesystem, all capabilities dropped, strict pod-security level.

# Check that it is read only

These run as you and ask the API server what the viewer account may do (all should match the right-hand column):

* `kubectl auth can-i list pods -A --as=system:serviceaccount:headlamp:headlamp-viewer` yes
* `kubectl auth can-i get helmreleases.helm.toolkit.fluxcd.io -A --as=system:serviceaccount:headlamp:headlamp-viewer` yes
* `kubectl auth can-i get secrets -A --as=system:serviceaccount:headlamp:headlamp-viewer` no
* `kubectl auth can-i create pods/exec -A --as=system:serviceaccount:headlamp:headlamp-viewer` no
* `kubectl auth can-i delete deployments -A --as=system:serviceaccount:headlamp:headlamp-viewer` no
* `kubectl auth can-i '*' '*' --as=system:serviceaccount:headlamp:headlamp` no (the pod's own account)

# Cut off a token early

Tokens expire on their own. To invalidate every token already issued for the viewer, delete the ServiceAccount (`kubectl -n headlamp delete serviceaccount headlamp-viewer`); Flux recreates it within about ten minutes, or run `flux reconcile kustomization apps`. Tokens are tied to the account's identity, so the old ones stop working.

# See more or less

Edit `apps/home/administration/headlamp-rbac.yaml`: add an API group to `headlamp-viewer-extra` to show another kind of custom resource. Do not add `secrets`, and do not bind the account to a role that can write. For changes use `kubectl` or Git and Flux, not this UI.
