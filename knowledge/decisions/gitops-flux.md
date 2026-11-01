---
type: Decision
title: GitOps with Flux
description: Flux (not ArgoCD) reconciles the cluster from GitHub, with SOPS + age for secrets.
tags: [gitops, flux, secrets]
status: stable
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, section 1
---

Flux was recommended for small footprint, native `HelmRelease` CRDs and built-in SOPS/age.[^dp] GitHub is the Git source of truth; Gitea in the cluster is just an app. ArgoCD remains a cheap later switch since manifests and Helm values are identical.

# Layout

```
clusters/home/        # flux-system/, infrastructure.yaml, apps.yaml
infrastructure/       # controllers/, configs/
apps/                 # base/<app>/, home/<category>/
```

`apps.yaml` `dependsOn` `infrastructure.yaml` so CNI, storage and cert-manager apply before apps. Renovate keeps chart versions current via PRs.

Manual bits kept outside Flux: `ansible/`, `preseed/`, `cilium/values.yaml`. The SOPS age key lives in the password manager, never in the repo.

[^dp]: deployment-plan.md section 1.
