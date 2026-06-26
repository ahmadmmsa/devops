# HA NFS Server on Ubuntu 24.04+

Two-node **active/passive** NFS cluster. DRBD mirrors a block device between the nodes; Pacemaker/Corosync owns a floating VIP and starts NFS on whichever node holds the DRBD primary. Clients only ever talk to the VIP, so a node failure is a transparent (brief) reconnect.

<br>

## Architecture

```
                     Clients
              mount  nfs-vip:/srv/nfs/export
                        |
                 192.168.8.210 (VIP)
                        |
          +-------------+-------------+
          |                           |
        nfs1                        nfs2
   192.168.8.211               192.168.8.212
   Pacemaker/Corosync          Pacemaker/Corosync
   DRBD primary  ===========>  DRBD secondary   (TCP 7788 replication)
   /dev/drbd0 mounted          (disk in sync, not mounted)
   nfs-kernel-server ACTIVE    nfs-kernel-server STANDBY
```

Active/passive: **only one node mounts `/dev/drbd0` and serves NFS at a time.** The cluster fails the whole stack (DRBD promote → mount → NFS → VIP) over to the survivor.

| Port | Proto | Service |
|------|-------|---------|
| 2049 | TCP | NFSv4 |
| 111  | TCP/UDP | rpcbind (NFSv3 / portmapper) |
| 7788 | TCP | DRBD replication (per resource) |
| 5405 | UDP | Corosync (totem) |
| 5404 | UDP | Corosync (mcast) |
| 2224 | TCP | pcsd (cluster admin) |

<br>

## Prerequisites

All steps run on **both nodes** unless a node label (`nfs1` / `nfs2`) is shown.

- 2 nodes running Ubuntu 24.04+ with a **second, identical, empty block device** for data (here `/dev/sdb` — adjust to yours). The OS lives on `/dev/sda`.
- Static IPs, hostnames resolvable, time synced.

```bash
lsb_release -a          # confirm Ubuntu 24.04+
ip link show            # note the interface name (enp1s0 / ens3 ...) for the VIP
lsblk                   # confirm the spare data disk, e.g. /dev/sdb, is EMPTY
```
> `/dev/sdb` will be **wiped**. Triple-check `lsblk` — this is the irreversible step.

<br>

## Step 1 — /etc/hosts (both nodes)

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.8.211  nfs1
192.168.8.212  nfs2
192.168.8.210  nfs-vip
EOF
```

<br>

## Step 2 — Firewall (both nodes)

```bash
sudo ufw allow 22/tcp        # SSH — FIRST, so you don't lock yourself out
sudo ufw allow 2049/tcp      # NFSv4
sudo ufw allow 111           # rpcbind (NFSv3)
sudo ufw allow 7788/tcp      # DRBD replication
sudo ufw allow 2224/tcp      # pcsd
sudo ufw allow 5404,5405/udp # Corosync
sudo ufw allow proto vrrp    # (only if you later swap Pacemaker VIP for keepalived)
sudo ufw enable
```
> The DRBD port is per-resource starting at 7788. If you add a second DRBD resource it uses 7789, etc. — open those too.

<br>

## Step 3 — Time sync (both nodes)

Corosync membership and NFS lock recovery both hate clock drift.

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
sleep 30 && chronyc tracking   # System time offset must be < 1s
sudo chronyc makestep          # force immediate sync if offset is large
```

<br>

## Step 4 — Install DRBD (both nodes)

The DRBD kernel module ships in the mainline Ubuntu kernel; you only need the userspace tools.

```bash
sudo apt update
sudo apt install -y drbd-utils
sudo modprobe drbd
cat /proc/drbd 2>/dev/null || drbdadm --version   # confirm module + tools
echo drbd | sudo tee /etc/modules-load.d/drbd.conf  # load on boot
```

<br>

## Step 5 — Configure the DRBD resource (both nodes — identical file)

```bash
sudo tee /etc/drbd.d/nfs.res <<'EOF'
resource nfs {
  protocol C;                 # synchronous: write acked only after it hits BOTH disks
  device    /dev/drbd0;
  disk      /dev/sdb;         # the spare data disk (will be wiped)
  meta-disk internal;         # store DRBD metadata on the same disk
  net {
    verify-alg sha1;
  }
  on nfs1 {
    address 192.168.8.211:7788;
  }
  on nfs2 {
    address 192.168.8.212:7788;
  }
}
EOF
```
> The file must be **byte-identical** on both nodes. Protocol C (synchronous) is mandatory for HA — anything else can lose acknowledged writes on failover.

<br>

## Step 6 — Initialise DRBD

**On both nodes** — create metadata and bring the resource up:
```bash
sudo drbdadm create-md nfs    # answer 'yes' to overwrite the disk
sudo drbdadm up nfs
sudo drbdadm status nfs       # both should show Connected, Secondary/Secondary, Inconsistent
```

**On nfs1 ONLY** — force it to become the first primary and start the initial sync:
```bash
sudo drbdadm primary --force nfs
watch -n2 drbdadm status nfs  # wait until UpToDate/UpToDate (initial sync done)
```
> Run `primary --force` on **one node only**. Doing it on both = split-brain (two divergent copies you must manually reconcile). Wait for `UpToDate/UpToDate` before continuing.

**On nfs1 ONLY** — make the filesystem on the replicated device:
```bash
sudo mkfs.ext4 /dev/drbd0     # format the DRBD device, NOT /dev/sdb directly
```

<br>

## Step 7 — Prepare NFS (both nodes)

Install the server but **disable it** — Pacemaker, not systemd, decides when/where NFS runs.

```bash
sudo apt install -y nfs-kernel-server
sudo systemctl disable --now nfs-server nfs-kernel-server rpcbind 2>/dev/null || true
sudo systemctl disable --now nfs-server  # ensure it never autostarts
```

Create the mount point and export directory layout **on both nodes** (the dirs must exist on both; the *contents* live on the replicated volume):
```bash
sudo mkdir -p /srv/nfs            # DRBD mount target
```

**On nfs1 ONLY** (it holds the primary, so mount once to lay down the structure):
```bash
sudo mount /dev/drbd0 /srv/nfs
sudo mkdir -p /srv/nfs/export     # the actual share
sudo mkdir -p /srv/nfs/nfsinfo    # NFS state dir (lock/grace info) — MUST live on DRBD
sudo chown nobody:nogroup /srv/nfs/export
sudo umount /srv/nfs              # unmount — Pacemaker will manage mounting from now on
```
> Leave `/etc/exports` empty. The cluster's `exportfs` resource (Step 11) defines the share, so the export follows the failover instead of being statically pinned to a node.
> Keeping the NFS state dir (`nfsinfo`) on the DRBD volume is what lets file locks survive a failover — without it clients lose locks on every switch.

<br>

## Step 8 — Install the cluster stack (both nodes)

```bash
sudo apt install -y pacemaker corosync pcs resource-agents-extra
sudo systemctl enable --now pcsd
sudo passwd hacluster          # set the SAME password on both nodes (cluster admin user)
```

<br>

## Step 9 — Form the cluster (run on nfs1 ONLY)

```bash
# authenticate both nodes to each other
sudo pcs host auth nfs1 nfs2 -u hacluster -p <password>

# create + start a 2-node cluster
sudo pcs cluster setup nfs_cluster nfs1 nfs2
sudo pcs cluster start --all
sudo pcs cluster enable --all   # start cluster services on boot

sudo pcs status                 # both nodes Online
```

Set cluster-wide properties (run on nfs1):
```bash
# Lab/homelab: no fencing hardware. Production: configure STONITH instead (see Gotchas).
sudo pcs property set stonith-enabled=false
# 2-node clusters can't have real quorum; ignore it (Corosync two_node mode handles this)
sudo pcs property set no-quorum-policy=ignore
```
> `stonith-enabled=false` is acceptable **only** in a lab. In production, no fencing + a network partition = both nodes go primary = split-brain data corruption. See Gotchas.

<br>

## Step 10 — Define cluster resources (run on nfs1 ONLY)

Build the stack with pcs. Each resource is created, then ordered/colocated so the whole chain follows the DRBD primary.

```bash
# 1) DRBD as a promotable (master/slave) clone
sudo pcs resource create drbd_nfs ocf:linbit:drbd \
  drbd_resource=nfs \
  op monitor interval=20s role=Promoted \
  op monitor interval=30s role=Unpromoted
sudo pcs resource promotable drbd_nfs \
  promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 notify=true

# 2) Filesystem: mount /dev/drbd0 -> /srv/nfs
sudo pcs resource create nfs_fs ocf:heartbeat:Filesystem \
  device=/dev/drbd0 directory=/srv/nfs fstype=ext4 \
  op monitor interval=20s

# 3) NFS server daemon, with its state dir on the replicated volume
sudo pcs resource create nfs_daemon ocf:heartbeat:nfsserver \
  nfs_shared_infodir=/srv/nfs/nfsinfo nfs_no_notify=true \
  op monitor interval=30s

# 4) The export itself
sudo pcs resource create nfs_export ocf:heartbeat:exportfs \
  clientspec="192.168.8.0/24" \
  options="rw,sync,no_subtree_check,no_root_squash" \
  directory=/srv/nfs/export fsid=0 \
  op monitor interval=30s

# 5) Floating VIP clients connect to
sudo pcs resource create nfs_vip ocf:heartbeat:IPaddr2 \
  ip=192.168.8.210 cidr_netmask=24 nic=enp1s0 \
  op monitor interval=20s
```

Now wire up **order** (start sequence) and **colocation** (everything on the same node as the DRBD primary):
```bash
# put all the non-DRBD resources in one group so they start in order & move together
sudo pcs resource group add nfs_group nfs_fs nfs_daemon nfs_export nfs_vip

# the group runs only where DRBD is Promoted...
sudo pcs constraint colocation add nfs_group with Promoted drbd_nfs-clone INFINITY
# ...and only AFTER DRBD is promoted
sudo pcs constraint order promote drbd_nfs-clone then start nfs_group
```
> Order matters: DRBD promote → mount → nfsserver → exportfs → VIP. Bringing the VIP up first would advertise the service before the export exists. The group + constraints encode exactly this.

<br>

## Step 11 — Verify

```bash
sudo pcs status                 # all resources Started; note which node is Promoted/active
sudo drbdadm status nfs         # Primary/Secondary, UpToDate/UpToDate
sudo pcs constraint             # confirm colocation + order constraints
showmount -e nfs-vip            # the export should be visible via the VIP
```
Expected: one node owns `nfs_group` + DRBD `Promoted`; the other is `Secondary`/standby.

<br>

## Step 12 — Mount from a client

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nfs
sudo mount -t nfs4 nfs-vip:/ /mnt/nfs        # fsid=0 makes the export the NFSv4 root
# persistent (note _netdir + soft/hard tradeoff):
echo 'nfs-vip:/  /mnt/nfs  nfs4  _netdev,hard,timeo=50,retrans=2  0 0' | sudo tee -a /etc/fstab
df -h /mnt/nfs
```
> Use `hard` mounts for data integrity (client retries forever through a failover) plus a sane `timeo`/`retrans` so it recovers quickly. `soft` can silently corrupt data on a switchover.

<br>

## Step 13 — Test failover

```bash
# from a client, start a continuous write
while true; do date >> /mnt/nfs/heartbeat.log; sleep 1; done &

# on the ACTIVE node — simulate failure
sudo pcs node standby nfs1          # graceful: hand resources to nfs2
#   or hard:  sudo pcs cluster stop nfs1   /   reboot the node

# watch on the other node
sudo pcs status                     # nfs_group + DRBD Promoted should land on nfs2
sudo drbdadm status nfs             # nfs2 now Primary/UpToDate
```
After verifying the client write paused briefly then continued on nfs2, bring nfs1 back:
```bash
sudo pcs node unstandby nfs1        # rejoins as Secondary; DRBD resyncs any delta
sudo pcs status
```
> A failover causes a short stall (DRBD promote + NFS grace period, ~tens of seconds), not a clean cutover. With `hard` mounts the client transparently resumes. Always test failover **before** trusting it with real data.

<br>

## Gotchas

- **DRBD split-brain** — `primary --force` on both nodes, or both going primary during a partition. Symptom: `StandAlone` / `Split-Brain detected`. Fix: pick a victim, `drbdadm secondary nfs` → `drbdadm connect --discard-my-data nfs` on it; `drbdadm connect nfs` on the survivor. The discarded node loses its divergent writes.
- **No fencing = real data loss** — in production set up **STONITH** (IPMI/iLO/PDU) and `stonith-enabled=true`. Without it, a network split can run NFS on both nodes against diverging DRBD copies. `stonith-enabled=false` is lab-only.
- **NFS state dir on local disk** — if `nfs_shared_infodir` isn't on the DRBD volume, clients lose all file locks on every failover. It MUST live under `/srv/nfs`.
- **`/etc/exports` populated** — don't. A static export pins the share to the local node and fights the cluster's `exportfs` resource. Leave it empty; the cluster owns exports.
- **Formatting `/dev/sdb` instead of `/dev/drbd0`** — always `mkfs` the DRBD device. Formatting the backing disk under a running DRBD corrupts replication.
- **Wrong NIC name in IPaddr2** — `nic=enp1s0` must match `ip link` on BOTH nodes. If the nodes have different interface names, the VIP fails to start on one of them.
- **Cluster won't start after reboot** — confirm `pcs cluster enable --all` was run; otherwise Corosync/Pacemaker don't autostart and resources stay down.
- **`no-quorum-policy=ignore` is for 2 nodes only** — if you grow to 3+ nodes, remove it and run real quorum, or a single failure can still strand the cluster.
- **Clients hang on failover with `soft`** — use `hard` mounts; `soft` returns I/O errors mid-failover and can corrupt in-flight data.
- **DRBD resync saturates the link** — a long resync after a node returns can throttle the replication NIC. Put DRBD on a dedicated/back-to-back link or cap it with `drbdadm disk-options --resync-rate`.
