## 1. Cluster Installation Method
> [!ATTENTION] Look into helm chart for listing all deploys in a file

- [x] Cluster was installed using kubeadm
- [x] Uses Containerd
- [x] Uses Kubelet

## 2. Storage
- [ ] Longhorn for app configs/databases
- [ ] Media on existing TrueNAS over NFS

## 3. Networking
> [!QUESTION] Heavy WIP - but my choice for now is Cilium
- [ ] Cilium
	- [ ] L2 Announcement
	- [ ] L4 BGP (deferred; L2 first)
	- [ ] L7... thing
- [x] Ingress Controller
- [x] Load Balancer
- [ ] BGP (deferred; OPNsense via os-frr)
## 4. Monitoring

## 5. Logging

## 6. Security

## 7. CI/CD

## 8. Applications
### Media
- [ ] Jackett
- [ ] Jellyseer
- [ ] Lidarr
- [ ] Notifiarr
- [ ] Radarr
- [ ] Sabnzbd
- [ ] Sonarr
- [ ] Tautulli
- [ ] Wizarr

### Monitoring
- [ ] Prometheus
- [ ] Grafana

### Administration
- [ ] Duplicati
- [x] Gitea
- [ ] Glance
