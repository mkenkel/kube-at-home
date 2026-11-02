# Phase 1 runbook (manual steps)

Run these by hand, in order, from your workstation or the nodes as noted. Nothing here is applied by Flux until step 6.
Versions: Kubernetes v1.36.6, Cilium chart 1.20.4, Gateway API v1.6.3, kube-vip v1.3.1 (checked 2026-11-01).
Unverified items are marked.

## 1. kube-vip on node-1

node-1's NIC is `eno1` (from `ip -br link`). On node-1 as root:

```sh
export VIP=192.168.15.6
export INTERFACE=eno1
export KVVERSION=v1.3.1
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
kube-vip manifest pod --interface $INTERFACE --address $VIP --controlplane --arp --leaderElection \
  | tee /etc/kubernetes/manifests/kube-vip.yaml
```

No `--services`: Cilium L2 announcements own LoadBalancer IPs.

Kubernetes 1.29+ workaround (from the kube-vip docs, re-read them before relying on this): on the first node only, kube-vip must use `super-admin.conf` instead of `admin.conf` until init finishes. Edit the `kubeconfig` hostPath in the manifest to `/etc/kubernetes/super-admin.conf`, then switch it back to `admin.conf` after `kubeadm init`.

## 2. kubeadm init (node-1)

```sh
kubeadm init --config kubeadm/kubeadm-config.yaml --skip-phases=addon/kube-proxy --upload-certs
```

Unverified: combining `--config` with `--skip-phases` and `--upload-certs`. Try `--dry-run` first. If kubeadm refuses, move `skipPhases` into `InitConfiguration`.

Save the join commands. Copy `/etc/kubernetes/manifests/kube-vip.yaml` to node-2 and node-3 (with `admin.conf` as the kubeconfig) before they join. Join node-2 and node-3 with the control-plane join command and node-4 with the worker one.

If control planes still carry the `control-plane` taint: `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`.

Nodes stay `NotReady` until Cilium is up.

## 3. Gateway API CRDs (pinned)

```sh
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/standard-install.yaml
```

This is the same ten CRDs (plus the `safe-upgrades` policy) that Flux applies from `config/crd/standard` at tag v1.6.3 in `clusters/home/gateway-api-crds.yaml`. Name match checked; byte equality not.

## 4. Cilium

```sh
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium --version 1.20.4 -n kube-system -f cilium/values.yaml
```

Wait for all nodes to be `Ready`.

## 5. Push the repo

Flux reads from GitHub, so merge this work to `main` and push before bootstrapping.

## 6. Flux

Use a `flux` CLI close to the version you want in the cluster (local is 2.9.5, latest was 2.10.0 on 2026-11-01).

```sh
flux bootstrap github --owner=mkenkel --repository=kube-at-home --branch=main --path=clusters/home --personal
git pull
kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/path/to/age.key
```

The key file is the one from `age-keygen`; never commit it. Flux retries the Kustomizations until `sops-age` exists.

Then check `flux get kustomizations` and `flux get helmreleases -A`. Cilium should show as adopted with no changes. If it wants to upgrade, diff `cilium/values.yaml` against `spec.values`.
