# Phase 1 runbook

Steps 1-4 are automated by `ansible/cluster.yml`. Step 5 onward is manual (GitHub token, age key).
Versions: Kubernetes v1.36.6, Cilium chart 1.20.4, Gateway API v1.6.3, kube-vip v1.3.1, Helm v3.22.1 (checked 2026-11-01); they live in `ansible/group_vars/all.yml`. `kubeadm/kubeadm-config.yaml` is copied to node-1 by the playbook.

## Steps 1-4: `ansible-playbook cluster.yml` (from `ansible/`)

1. kube-vip static pod on node-1 (`eno1`, VIP 192.168.15.6, ARP, no `--services` since Cilium L2 owns LoadBalancer IPs), using `super-admin.conf` until init finishes, then switched back to `admin.conf`.
2. `kubeadm init --config ... --skip-phases=addon/kube-proxy --upload-certs`, then node-2 and node-3 join as control planes (one at a time, with the kube-vip manifest copied over) and node-4 joins as a worker. `admin.conf` is fetched to `~/.kube/kube-at-home.conf`.
3. Control-plane taints removed, Gateway API CRDs applied from the v1.6.3 `standard-install.yaml` release asset.
4. Helm installed on node-1 and Cilium installed from `cilium/values.yaml`, then waits for all nodes to be Ready.

Re-running skips steps already done (node-1 and the joins are detected by `/etc/kubernetes/kubelet.conf`).

Unverified: `--skip-phases`/`--upload-certs` together with `--config` (the playbook is not tested; if kubeadm refuses, move `skipPhases` into `InitConfiguration`); the kube-vip `super-admin.conf` workaround on this version; that the last line of `kubeadm init phase upload-certs` is the certificate key. Use `ansible-playbook cluster.yml --check` only for syntax-level sanity, since most tasks are `command`.

## 5. Push the repo

Flux reads from GitHub, so `main` must be pushed before bootstrapping.

## 6. Flux (manual)

Use a `flux` CLI close to the version you want in the cluster (local is 2.9.5, latest was 2.10.0 on 2026-11-01).

```sh
export KUBECONFIG=~/.kube/kube-at-home.conf
flux bootstrap github --owner=mkenkel --repository=kube-at-home --branch=main --path=clusters/home --personal
git pull
kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/path/to/age.key
```

The key file is the one from `age-keygen`; never commit it. Flux retries the Kustomizations until `sops-age` exists.

Then check `flux get kustomizations` and `flux get helmreleases -A`. Cilium should show as adopted with no changes. If it wants to upgrade, diff `cilium/values.yaml` against `spec.values` in the HelmRelease.
