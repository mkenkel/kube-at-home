# kube-at-home

GitOps repo for a home Kubernetes cluster: 4x Lenovo M70q Gen2, Debian, kubeadm + containerd, Flux, Cilium, Longhorn, TrueNAS NFS for media.

## Where things live

- `plan.md`: short scope checklist (what runs).
- `deployment-plan.md`: how and in what order (phases 0-5, rationale, testing).
- `knowledge/`: OKF v0.2 bundle with the current decisions and per-phase checklists. Start at `knowledge/index.md`.
- Planned, not yet created: `clusters/home/`, `infrastructure/`, `apps/`, `ansible/`, `preseed/`, `cilium/values.yaml` (layout in `knowledge/decisions/gitops-flux.md`).

## Working rules

- Follow the phase order in `deployment-plan.md`. Phase 1 has strict ordering: Gateway API CRDs, then Cilium by hand, then Flux.
- Cluster changes go through Git and Flux. Never `kubectl apply` or `helm upgrade` against the real cluster unless a phase step says it is manual, and confirm with the user first.
- Steps tagged **[human]** in the phase docs need hardware, secrets or the network. Prepare the files and commands, but do not run them against nodes.
- Never commit secrets or the age private key. Secrets are SOPS-encrypted with age; the key lives in the user's password manager.
- Validate before committing: `kustomize build`, `kubeconform`, `flux diff` where available.
- Pin chart and CRD versions. One PR per app category in Phase 5.
- Don't plan or build items marked dropped or deferred (Authentik, BGP, offsite Duplicati, game servers, notetaking/file/home-automation apps).

## Knowledge bundle (OKF)

Keep `knowledge/` in sync with reality as work lands:

- Every concept `.md` needs YAML frontmatter with a non-empty `type`. `index.md` and `log.md` are reserved; root `index.md` carries only `okf_version`.
- Use `title`, `description`, `tags`, `status` (`draft | stable | deprecated`) and `sources` for provenance. Links are bundle-relative (`/decisions/storage.md`).
- Agent-written content gets `generated: { by: claude/<model>, at: <ISO time> }`. Only the user adds `verified: { by: human:<id>, at: ... }`; never add it yourself.
- When a step is done or a decision changes: tick the checkbox in the phase doc, update the decision doc, resolve the item in `open-questions.md`, and add a dated entry at the top of `knowledge/log.md` (YYYY-MM-DD).
- Spec: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md

## Parallel work with workmux

`.workmux.yaml` is set up for `workmux add <name>` (one git worktree + tmux window + Claude per task). Parallelise only independent repo-side work, such as the Ansible playbook, preseed file, repo skeleton and CI config. Merge with `workmux merge`. Cluster-touching steps stay serial in the main checkout.

## Conventions

- Git: small commits, imperative subject lines, branch off `main`.
