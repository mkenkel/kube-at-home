---
type: Runbook
title: Gitea, self-hosted Git
description: How Gitea is exposed (web through the Gateway, git over SSH on its own IP), the one-time setup, how it is backed up, and what is deliberately blocked.
tags: [runbook, gitea, ssh, git, backup]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-11-04T00:00:00Z }
sources:
  - id: gitea-release
    resource: https://github.com/go-gitea/gitea/releases/tag/v28.3.0
    title: Gitea 28.3.0 (the 28.x line replaced 1.27; its one breaking change is the internal proxy for outgoing Git network operations)
  - id: gitea-rootless
    resource: https://github.com/go-gitea/gitea/blob/v28.3.0/Dockerfile.rootless
    title: Rootless image (ports 3000 and 2222, config rebuilt from GITEA__section__KEY variables at each start)
---

# What it is

Gitea 28.3.0 (rootless image) in the `gitea` namespace. GitHub stays the source of truth for this cluster's own repo ([GitOps](/decisions/gitops-flux.md)); Gitea is an app for your other repositories.

* **Web:** `https://gitea.home.mkenkel.com` through the Gateway, LAN and WireGuard only. Registration is off and nothing, public repos included, can be viewed without signing in.
* **Git over SSH:** `git@git.mkenkel.com:<user>/<repo>.git`, port 22. The Gateway only carries HTTP(S), so SSH has its own `LoadBalancer` Service, `gitea-ssh`, with the fixed address **192.168.15.101** from the Cilium pool (announced by ARP on the LAN like the Gateway's .100). Inside the pod Gitea's built-in SSH server listens on 2222. Its host keys live on the data volume, so they survive restarts and your `known_hosts` entry stays valid.
* **Storage:** SQLite and the repositories on one 10Gi Longhorn volume, `retain: true`, in the nightly and weekly config backups ([backups](/runbooks/backup-restore.md)). Backups are crash-consistent; Git and SQLite both recover from that.
* **Config:** there is no `app.ini` to edit. The image rebuilds it from the `GITEA__section__KEY` variables in `apps/base/gitea/helmrelease.yaml` at every start. The two secrets (`SECRET_KEY`, which protects stored 2FA secrets and tokens, and the OAuth2 JWT secret) are in the SOPS secret `gitea-secrets`. **Keep a copy of `SECRET_KEY` in the password manager: a restored database is useless for those without it.**
* **Network policy:** `gitea-isolation` admits the Gateway and Glance on 3000 and the LAN on 2222, and lets Gitea reach DNS only.

# One-time setup **[human]**

1. **DNS** (done 2026-11-04): in OPNsense, Services > Unbound DNS > Overrides, host `git`, domain `mkenkel.com`, A, 192.168.15.101. **Do not put it under `home.mkenkel.com`.** The `*.home.mkenkel.com` wildcard is an Unbound `redirect` zone that allows no other name inside it: a `git.home.mkenkel.com` override makes Unbound fail to start (`local-data in redirect zone must reside at top of zone`) and DNS goes down for the whole LAN until the row is deleted. Without any override you can still use the IP in `~/.ssh/config`.
2. **Admin account** (after Flux has applied and the pod is Ready). The password is generated and printed once, so redirect it to a file and move it to the password manager:

   `kubectl -n gitea exec deploy/gitea -- gitea admin user create --admin --username mkenkel --email mkenkel@home.mkenkel.com --random-password --must-change-password=false > ~/gitea-admin-credentials.txt`

   (`admin` itself is a reserved name in Gitea.) Change the e-mail in the UI if you want notifications; mail is not configured.
3. Sign in at the web address, then Settings > SSH / GPG Keys > Add Key and paste your public key (the same one you use for GitHub is fine).
4. Optional `~/.ssh/config` entry:

   `Host git.mkenkel.com` / `  User git` / `  IdentityFile ~/.ssh/<your key>`

5. Test: `ssh -T git@git.mkenkel.com` should greet you by username. Then `git remote add gitea git@git.mkenkel.com:mkenkel/<repo>.git`.

# If SSH does not connect

* `kubectl -n gitea get svc gitea-ssh` must show `EXTERNAL-IP 192.168.15.101`. `<pending>` means the address annotation was not accepted or the pool is exhausted.
* Connection refused or times out from the LAN but works with `kubectl -n gitea port-forward deploy/gitea 2222:2222`: look at Hubble (`hubble.home.mkenkel.com`, namespace `gitea`) for a drop. Which Cilium identity the client is seen as depends on the node that answers ARP and where the pod runs, and `gitea-isolation` admits `world`, `remote-node` and `host` on 2222 for that reason.
* "Host key verification failed" after a restore from backup means the data volume (and the host keys) were replaced; remove the old line from `known_hosts`.

# Deliberately blocked: outgoing connections

The policy gives Gitea DNS and nothing else, so **mirroring or migrating a repository from GitHub, push mirrors and webhooks do not work yet.** Gitea 28 also sends its outgoing Git operations through an internal proxy, so it is worth reading the 28.x notes before enabling any of them. To allow one, add an egress rule to `apps/home/administration/gitea-networkpolicy.yaml` for what you need (`toFQDNs` for `github.com`, port 443, is the narrowest). Pushing from your workstation over SSH needs none of that.

# Upgrades

Renovate does not bump the image tag inside HelmRelease values, so change `tag` in `apps/base/gitea/helmrelease.yaml` by hand. Read the release notes first and take a manual Longhorn backup of the volume, since a database migration cannot be undone. The startup probe allows up to three minutes for it.

# Restore

Same as any other config volume: see [Restoring for real](/runbooks/backup-restore.md). The PVC is named `gitea` in namespace `gitea`.
