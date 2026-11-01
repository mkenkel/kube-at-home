# Debian preseed for kube nodes

`preseed.cfg` drives an unattended minimal Debian install of node-1..4
(192.168.15.7-.10, see `knowledge/decisions/cluster-topology.md`).

What it does:

- DHCP networking. The hostname comes from the OPNsense static mapping, nothing is hardcoded.
  `netcfg/get_hostname` and `netcfg/get_domain` hold throwaway placeholders purely so the installer never prompts.
- Single disk `/dev/nvme0n1`, GPT, plain partitions: 512 MiB EFI plus the rest as ext4 `/` (noatime).
  No LVM and no swap (kubelet wants swap off; etcd sits on the NVMe root).
  The second bay, intended for Longhorn, is not touched.
- No root login. User `ops` with a locked password, SSH key only, passwordless sudo (for Ansible).
- Installs only `openssh-server`, `python3`, `python3-debian` (needed by Ansible's `deb822_repository`) and `sudo`. No tasks, no desktop. Ansible does the rest.

## Before first use

1. Edit the `KEY=` line in the `preseed/late_command` of `preseed.cfg` and put in your real **public** key.
   The install fails on purpose while the `TODO` placeholder is still there.
2. Confirm the OS disk is `/dev/nvme0n1` on every node (`lsblk` in the installer shell). If a node has two
   NVMe drives the numbering can differ; adjust `partman-auto/disk` and `grub-installer/bootdev`.
3. The installer **wipes that disk**. Do not point it at a node holding data.
4. Click "Apply changes" on the OPNsense DHCP page first so the reservations are live.

## Serving the file

The file is uploaded to the netboot.xyz container's assets directory as `assets/preseed/preseed.cfg`.
nginx serves it on port 8081 on this setup, so the installer fetches
`http://<netboot-host-LAN-IP>:8081/preseed/preseed.cfg`. Check it from another machine first:

```sh
curl http://<netboot-host-LAN-IP>:8081/preseed/preseed.cfg
```

The file holds no secrets (public key only), but it is plain HTTP, so keep it on the LAN.
Re-upload it whenever `preseed.cfg` changes in this repo.

## netboot.xyz boot entry

`custom.ipxe` in this directory adds a "Custom Boot Options" menu with one entry that boots the
trixie installer with the preseed URL already set, so there is nothing to type per node.

1. In the netboot.xyz web app go to Menus and paste `custom.ipxe` as a file named exactly `custom.ipxe`.
   It uses netboot.xyz's own Debian variables (mirror, `${netcfg}`, `${cmdline}`) and `preseed_host`
   is the container host, `192.168.15.25`.
2. Upload `preseed.cfg` to the assets dir as `assets/preseed/preseed.cfg` (re-upload when it changes).
3. In `boot.cfg` (web app, Menus), add under `:global_vars`, after `boot_timeout`:
   `set custom_url http://192.168.15.25:8081`
   The stock `menu.ipxe` shows a "Custom URL Menu" entry when `custom_url` is set and chains
   `${custom_url}/custom.ipxe`. Saving `custom.ipxe` in Menus alone does not add a menu item (its
   `:custom-user` label looks for a `custom/` folder and nothing links to it). Re-add this line if a
   container upgrade overwrites `boot.cfg`.
4. Netboot a node, pick "Custom URL Menu", then the Kubernetes entry.

Untested against the real instance. Check first, from another machine:
`curl -sI http://192.168.15.25:8081/custom.ipxe` and `.../preseed/preseed.cfg` should both return 200.
If `custom.ipxe` is not served at the web root, say so and the `custom_url` path needs a prefix.

Built-in alternative with no custom file: netboot.xyz menu, Debian, trixie, Preseed, then type the URL above.
It prompts for the URL on every boot.

Notes:

- `auto=true priority=critical` defers the locale/keyboard questions until the preseed is loaded and
  suppresses non-critical prompts. `preseed/url=` is what points the installer at the file.
- The two `netcfg/...` parameters on the kernel line cover the hostname/domain prompts that appear before
  the preseed is downloaded. A name supplied by DHCP still wins.
- `initrd=initrd.magic` on the kernel line is what netboot.xyz uses so a UEFI boot finds the initrd.

## Verifying after install

```sh
ssh ops@192.168.15.7 'hostname; sudo -n true && echo sudo ok; lsblk'
```

`hostname` must print `node-1`, not `unassigned-hostname`. If it does, the DHCP mapping has no
hostname set (check OPNsense) and the node must be fixed with `hostnamectl set-hostname` or Ansible.
