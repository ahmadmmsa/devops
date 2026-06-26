
# Proxmox VE 9.2

Debian-based (PVE 9.x = Debian 13 "Trixie") type-1 hypervisor. Runs **QEMU/KVM VMs** (`qm`) and **LXC containers** (`pct`) under one web UI / REST API, with ZFS, Ceph, and clustering built in.

- [Post-install](#post-install)
- [Node & cluster basics](#node--cluster-basics)
- [Storage](#storage)
- [Templates & images](#templates--images)
- [VMs (qm)](#vms-qm)
- [Cloud-init VMs](#cloud-init-vms)
- [LXC containers (pct)](#lxc-containers-pct)
- [Snapshots & clones](#snapshots--clones)
- [Backup & restore (vzdump)](#backup--restore-vzdump)
- [Networking](#networking)
- [Clustering & HA](#clustering--ha)
- [pvesh / API](#pvesh--api)
- [Updates](#updates)
- [Gotchas](#gotchas)

## Post-install
Fresh PVE points apt at an *enterprise* repo you can't use without a subscription. Switch to the no-subscription repos first (PVE 9 uses the new **deb822** `.sources` format).
```bash
# disable enterprise repos
rm -f /etc/apt/sources.list.d/pve-enterprise.sources
rm -f /etc/apt/sources.list.d/ceph.sources

# add no-subscription repo (deb822 format, Debian 13 "trixie")
cat > /etc/apt/sources.list.d/pve-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt full-upgrade -y

# remove the "No valid subscription" nag (re-apply after major upgrades)
pveversion                                  # confirm version
```
> The no-subscription repo is fine for homelab/testing but **not** recommended for production by Proxmox. Don't mix enterprise + no-subscription at the same time.

## Node & cluster basics
```bash
pveversion -v                  # full version of every PVE component
pvesh get /version             # via API
systemctl status pveproxy      # web UI service (port 8006)
pvecm status                   # cluster status (single node is fine too)
pvecm nodes                    # list nodes in the cluster
pveperf                        # quick CPU/disk/fsync benchmark of the node
ha-manager status              # HA resource status
```
Web UI: `https://<node-ip>:8006` (self-signed cert, log in as `root@pam`).

## Storage
Storage is defined in `/etc/pve/storage.cfg` and shared across the cluster config. Each storage has *content types* it's allowed to hold (images, rootdir, vztmpl, iso, backup, snippets).
```bash
pvesm status                              # list storages + usage
pvesm list <storage>                      # list volumes on a storage
pvesm scan zfs                            # find importable ZFS pools
pvesm add dir backups --path /mnt/backups --content backup,iso,vztmpl
pvesm add zfspool tank --pool tank --content images,rootdir
pvesm remove <storage>
```
```bash
# ZFS (very common on PVE)
zpool status
zpool list
zfs list
zpool create -f tank mirror /dev/sdb /dev/sdc   # mirrored pool from 2 disks
arc_summary | head                              # ZFS ARC (RAM cache) stats
```
> ZFS ARC eats RAM by design. On PVE 9 the default ARC max is capped lower than old defaults, but verify with `arc_summary` and set `zfs_arc_max` in `/etc/modprobe.d/zfs.conf` if VMs are getting starved.

## Templates & images
```bash
# LXC container templates
pveam update                              # refresh template catalog
pveam available | grep -i debian         # list downloadable templates
pveam download local debian-13-standard_*_amd64.tar.zst

# ISOs / cloud images go on a storage with 'iso'/'import' content
cd /var/lib/vz/template/iso
aria2c -x 16 -s 16 https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img
```

## VMs (qm)
`qm` = QEMU/KVM VM management. VMs are identified by a numeric **VMID** (e.g. 100).
```bash
qm list                                   # all VMs on this node
qm status <vmid>
qm start <vmid>
qm shutdown <vmid>                        # ACPI graceful
qm stop <vmid>                            # hard power-off (pull the plug)
qm reboot <vmid>
qm reset <vmid>                           # hard reset

qm config <vmid>                          # show config
qm set <vmid> --memory 4096 --cores 4     # live-edit config
qm set <vmid> --onboot 1                  # autostart on host boot
qm monitor <vmid>                         # QEMU monitor console
qm terminal <vmid>                        # serial console (needs serial0 + console in guest)

qm destroy <vmid> --purge                 # delete VM + remove from backups/replication
```
```bash
# create a VM from scratch
qm create 100 --name web01 --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --ostype l26
qm set 100 --scsi0 local-zfs:32          # add a 32 GiB disk on local-zfs
qm set 100 --ide2 local:iso/ubuntu-24.04.iso,media=cdrom
qm set 100 --boot order=scsi0\;ide2
```

## Cloud-init VMs
The fast path: import a cloud image, attach a cloud-init drive, set user/SSH key, turn it into a template, then clone. Mirrors how you do KVM with virt-customize.
```bash
qm create 9000 --name ubuntu-2404-tmpl --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --ostype l26

# import the downloaded cloud image as the boot disk
qm set 9000 --scsi0 local-zfs:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
qm set 9000 --ide2 local-zfs:cloudinit          # cloud-init drive
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0      # cloud images need a serial console

# cloud-init settings
qm set 9000 --ciuser ahmad --sshkeys ~/.ssh/id_ed25519.pub
qm set 9000 --ipconfig0 ip=dhcp                 # or ip=192.168.8.50/24,gw=192.168.8.1
qm template 9000                                # freeze as a template

# clone it (linked = fast/thin, full = independent copy)
qm clone 9000 101 --name web01 --full
qm set 101 --ipconfig0 ip=192.168.8.51/24,gw=192.168.8.1
qm start 101
```
> Resize the disk *after* cloning if the image is small: `qm resize 101 scsi0 +20G` (cloud-init grows the rootfs on boot).

## LXC containers (pct)
`pct` = LXC container management. Lighter than VMs (shared kernel) — great for services.
```bash
pct list
pct create 200 local:vztmpl/debian-13-standard_*_amd64.tar.zst \
  --hostname svc01 --cores 2 --memory 1024 --rootfs local-zfs:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 --features nesting=1 \
  --ssh-public-keys ~/.ssh/id_ed25519.pub

pct start 200
pct enter 200                             # shell inside the container
pct exec 200 -- apt update                # run a command inside
pct config 200
pct set 200 --memory 2048
pct stop 200
pct destroy 200
```
> Prefer **unprivileged** containers (`--unprivileged 1`). Docker-in-LXC needs `--features nesting=1,keyctl=1`; for anything that fights the shared kernel (some kernel modules, NFS server), use a VM instead.

## Snapshots & clones
```bash
qm snapshot <vmid> <name> --description "before upgrade"
qm listsnapshot <vmid>
qm rollback <vmid> <name>
qm delsnapshot <vmid> <name>
# same verbs exist for containers:
pct snapshot <ctid> <name>
pct rollback <ctid> <name>
```
> Snapshots need a storage that supports them (ZFS, qcow2, Ceph, LVM-thin). Plain LVM and raw-on-dir **can't** snapshot. Snapshots are not backups — they live on the same disk.

## Backup & restore (vzdump)
```bash
# back up one guest
vzdump 100 --storage backups --mode snapshot --compress zstd
vzdump 100 --mode stop                    # stop = consistent but downtime; snapshot = live
vzdump --all --storage backups            # everything on the node

# restore (note: qmrestore for VMs, pct restore for containers)
qmrestore /mnt/backups/dump/vzdump-qemu-100-*.vma.zst 105 --storage local-zfs
pct restore 205 /mnt/backups/dump/vzdump-lxc-200-*.tar.zst --storage local-zfs
```
Scheduled jobs live in the UI (Datacenter → Backup) or `/etc/pve/jobs.cfg`. For real retention/dedup use **Proxmox Backup Server** (PBS) as a storage target.
> `--mode snapshot` needs snapshot-capable storage or it silently falls back to suspend. Test a restore periodically — an untested backup is a hope, not a backup.

## Networking
Config lives in `/etc/network/interfaces`. The default `vmbr0` is a Linux bridge tying VMs to the physical NIC.
```bash
cat /etc/network/interfaces
ifreload -a                               # apply changes (ifupdown2, no reboot)
ip -br a                                   # brief address list
brctl show                                 # bridge membership (or: bridge link)
```
```ini
# /etc/network/interfaces — typical bridge
auto vmbr0
iface vmbr0 inet static
    address 192.168.8.10/24
    gateway 192.168.8.1
    bridge-ports enp1s0       # physical NIC enslaved to the bridge
    bridge-stp off
    bridge-fd 0
```
> Edit via the UI when possible — it stages changes and `ifreload`s safely. A typo in `interfaces` + reboot = locked out; keep a console/IPMI path. VLAN-aware bridges add `bridge-vlan-aware yes` + `bridge-vids 2-4094`.

## Clustering & HA
```bash
# on the first node — create the cluster
pvecm create <cluster-name>
# on each additional node — join (run FROM the joining node)
pvecm add <ip-of-first-node>
pvecm status
pvecm delnode <node-name>                 # remove a node (do it cleanly, from a remaining node)

# High Availability for a guest
ha-manager add vm:100 --state started
ha-manager status
ha-manager remove vm:100
```
> Clustering needs an **odd number of votes** for quorum (3+ nodes, or 2 nodes + a QDevice). Lose quorum and `/etc/pve` goes read-only — you can't start/edit guests. Don't build a 2-node cluster without a tiebreaker.

## pvesh / API
`pvesh` walks the same REST API the UI uses — great for scripting.
```bash
pvesh get /nodes                          # list nodes
pvesh get /nodes/<node>/qemu              # VMs on a node
pvesh get /cluster/resources --type vm    # all VMs/CTs cluster-wide
pvesh create /nodes/<node>/qemu/100/status/start
pvesh get /nodes/<node>/qemu/100/config
# raw REST with a token:
curl -k -H "Authorization: PVEAPIToken=user@pam!tokenid=<secret>" https://<node>:8006/api2/json/version
```
Create an API token: Datacenter → Permissions → API Tokens (uncheck "Privilege Separation" to inherit the user's perms).

## Updates
```bash
apt update && apt full-upgrade -y         # standard package upgrade
pveupgrade                                # PVE wrapper: warns about reboots/kernel
pveversion -v                             # verify after
# kernel pinning (avoid booting a bad new kernel)
proxmox-boot-tool kernel list
proxmox-boot-tool kernel pin <version>
```
> Always **snapshot/back up guests before a major PVE upgrade** (e.g. 8→9). Read the official upgrade guide for major jumps — repos, kernel, and Debian base all change. Reboot into the new kernel during a maintenance window.

## Gotchas
- **"No valid subscription" + apt 401** — you're still on the enterprise repo. Switch to `pve-no-subscription` (see Post-install).
- **Lost quorum, `/etc/pve` read-only** — too few cluster votes. Bring a node back, or `pvecm expected 1` *temporarily* to force quorum on a single survivor (then fix it properly).
- **`qm stop` vs `shutdown`** — `stop` is a hard kill (risk of FS corruption); `shutdown` is graceful ACPI. Use `stop` only when a guest is hung.
- **Cloud image won't boot / no console** — add `--serial0 socket --vga serial0`; cloud images expect a serial console, not VGA.
- **Snapshot option greyed out** — the guest is on storage that can't snapshot (raw-on-dir, plain LVM). Move the disk to ZFS/LVM-thin/qcow2.
- **Can't snapshot a VM with a passthrough device** — PCIe/USB passthrough disables live snapshots; stop the VM first.
- **LXC can't run Docker** — needs `--features nesting=1,keyctl=1`; some workloads still won't work in a container — use a VM.
- **Edited `/etc/network/interfaces` and rebooted into no network** — always `ifreload -a` and verify before rebooting; keep IPMI/console access.
- **VMID/hostname reuse after destroy** — `qm destroy --purge` to also clear backup/replication/HA references, otherwise stale entries linger.
- **ZFS pool import fails after disk reshuffle** — `zpool import` by name/ID; don't rely on `/dev/sdX` ordering, it isn't stable across reboots.
