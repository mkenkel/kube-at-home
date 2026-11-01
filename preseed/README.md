# Debian preseed for kube nodes

`preseed.cfg` drives an unattended minimal Debian install of kube-node-1..4
(192.168.15.7-.10, see `knowledge/decisions/cluster-topology.md`).

What it does:

- DHCP networking. The hostname comes from the OPNsense static mapping, nothing is hardcoded.
  `netcfg/get_hostname` and `netcfg/get_domain` hold throwaway placeholders purely so the installer never prompts.
- Single disk `/dev/nvme0n1`, GPT, plain partitions: 512 MiB EFI plus the rest as ext4 `/` (noatime).
  No LVM and no swap (kubelet wants swap off; etcd sits on the NVMe root).
  The second bay, intended for Longhorn, is not touched.
- No root login. User `ops` with a locked password, SSH key only, passwordless sudo (for Ansible).
- Installs only `openssh-server`, `python3` and `sudo`. No tasks, no desktop. Ansible does the rest.

## Before first use

1. Edit the `KEY=` line in the `preseed/late_command` of `preseed.cfg` and put in your real **public** key.
   The install fails on purpose while the `TODO` placeholder is still there.
2. Confirm the OS disk is `/dev/nvme0n1` on every node (`lsblk` in the installer shell). If a node has two
   NVMe drives the numbering can differ; adjust `partman-auto/disk` and `grub-installer/bootdev`.
3. The installer **wipes that disk**. Do not point it at a node holding data.
4. Click "Apply changes" on the OPNsense DHCP page first so the reservations are live.

## Serving the file

Any HTTP server reachable from the Servers VLAN works, from the repo root:

```sh
python3 -m http.server 8080 --directory preseed
# preseed is then at http://<your-host>:8080/preseed.cfg
```

Or host the file on any existing web server. Check with `curl http://<host>:8080/preseed.cfg`.
The file holds no secrets, but it is plain HTTP, so keep it on the LAN. The server only needs to run during installs.

## netboot.xyz custom menu entry

netboot.xyz is hosted outside the cluster. Add a custom menu (for example `custom/menu.ipxe`
in the netboot.xyz config, with custom menus enabled) containing an entry like this.
Adjust the kernel and initrd URLs to the Debian release you want; this example uses trixie's netboot files:

```ipxe
:debian_kube_preseed
set mirror http://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/debian-installer/amd64
kernel ${mirror}/linux auto=true priority=critical url=http://<your-host>:8080/preseed.cfg netcfg/get_hostname=unassigned-hostname netcfg/get_domain=unassigned-domain ---
initrd ${mirror}/initrd.gz
boot
```

Notes:

- `auto=true priority=critical` defers the locale/keyboard questions until the preseed is loaded and
  suppresses non-critical prompts. `url=` is what points the installer at the file.
- The two `netcfg/...` parameters on the kernel line cover the hostname/domain prompts that appear before
  the preseed is downloaded. A name supplied by DHCP still wins.

## Verifying after install

```sh
ssh ops@192.168.15.7 'hostname; sudo -n true && echo sudo ok; lsblk'
```

`hostname` must print `kube-node-1`, not `unassigned-hostname`. If it does, the DHCP mapping has no
hostname set (check OPNsense) and the node must be fixed with `hostnamectl set-hostname` or Ansible.
