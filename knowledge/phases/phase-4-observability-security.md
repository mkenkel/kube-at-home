---
type: Deployment Phase
title: "Phase 4: Observability and security"
description: kube-prometheus-stack, Loki + Alloy, network policies, pod security, local backups.
tags: [phase, monitoring, logging, security, backup]
status: draft
generated: { by: claude/claude-sonnet-5-5, at: 2026-10-27T00:00:00Z }
sources:
  - id: dp
    resource: /deployment-plan.md
    title: Deployment plan, Phase 4
---

- [x] [repo] Applied 2026-11-01 (all pods Running, PVCs Bound, HTTPRoute Accepted, Grafana returns 200 through the Gateway with a verified production Let's Encrypt cert; tested with `curl --resolve`, since the OPNsense wildcard override was not yet in place): `kube-prometheus-stack` 92.17.0 in `monitoring` (Prometheus 30Gi/15d and Grafana 5Gi on Longhorn, Grafana at `https://grafana.home.mkenkel.com` through the Gateway, Prometheus and Alertmanager not exposed). Grafana admin password is in the SOPS secret `grafana-admin`; read it with `sops -d --extract '["stringData"]["admin-password"]' infrastructure/controllers/monitoring/grafana-admin.sops.yaml`.
- [x] [human] Expose etcd, scheduler and controller-manager metrics: `ansible-playbook control-plane-metrics.yml`, run 2026-11-01 on node-1..3 with no failures; listeners on `*:2381`, `*:10257`, `*:10259`, and from node-1 etcd answers 200 and the other two 403 (token required) on all three node IPs. Edits the live static pod manifests one control plane at a time (`serial: 1`) and re-uploads `kubeadm/kubeadm-config.yaml` to the `kubeadm-config` ConfigMap so a later `kubeadm upgrade` keeps it. The HelmRelease already scrapes all three (Prometheus targets are down until the playbook runs). etcd metrics are plain HTTP on :2381 and unauthenticated on the LAN. Verified afterwards: the `kubeadm-config` ConfigMap has the new `extraArgs` (so `kubeadm upgrade` keeps them) and Prometheus shows etcd, scheduler and controller-manager 3/3 up, plus apiserver, kubelet, node-exporter and the monitoring stack itself.
- [ ] [repo] Longhorn ServiceMonitor prepared (`infrastructure/configs/monitoring/longhorn-servicemonitor.yaml`, selects `app: longhorn-manager`, port `manager`; not yet applied). Cilium/Hubble ServiceMonitors still to do: they need the Cilium metrics values (changing `cilium/values.yaml` and the HelmRelease together, which restarts the agents)
- [x] [repo] Applied 2026-11-01 (loki-0 and 4 Alloy pods Running, 30Gi PVC Bound; logs from cert-manager, flux-system, kube-system, longhorn-system and monitoring arrive in Loki, no Alloy errors; Grafana Explore not yet tried): Loki 3.7.9 (chart 20.2.1 from `grafana-community/helm-charts`; the OSS chart moved there in March 2026 and `grafana/helm-charts` is now Enterprise only) as a single process on a 30Gi Longhorn volume with 14 day retention, and Grafana Alloy 1.15.0 as a DaemonSet shipping pod logs (`loki.source.kubernetes`, node-local via `NODE_NAME`). Loki is added to Grafana as a datasource. Checked locally: the Loki chart renders with these values and the Alloy config passes `alloy fmt`. Observed once running: each kube-apiserver logs ~20 `grpc: addrConn.createTransport failed ... 127.0.0.1:2379: operation was canceled` warnings per 10 minutes. Etcd is healthy with 0 restarts, so this is probably benign gRPC reconnect noise; not confirmed, and not known whether it predates the metrics change. Recheck if it grows or etcd latency rises. TODO: ServiceMonitors for Loki and Alloy (Loki's needs the monitoring CRDs, so it would need `dependsOn` the stack)
- [ ] [repo] Cilium NetworkPolicies; Pod Security Admission labels (`baseline`/`restricted`)
- [ ] [repo] Longhorn recurring backups and Velero. Blocked on the TrueNAS dataset decision in [open questions](/open-questions.md).
- [ ] [human] First test restore, then quarterly
- [ ] Offsite Duplicati: deferred, see [backups](/decisions/backups.md)
