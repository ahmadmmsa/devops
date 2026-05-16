# Production HA PostgreSQL 18 on Ubuntu 24.04

<br> 
 
## Architecture

```
                   Applications
                        |
                 192.168.8.200 (VIP)
                        |
              HAProxy + Keepalived
                        |
       +----------------+----------------+
       |                |                |
    Node1            Node2            Node3
192.168.8.201   192.168.8.202   192.168.8.203
Patroni         Patroni         Patroni
PostgreSQL 18   PostgreSQL 18   PostgreSQL 18
etcd            etcd            etcd
HAProxy         HAProxy         HAProxy
Keepalived      Keepalived      Keepalived
```

| Port | Service |
|------|---------|
| 5432 | PostgreSQL |
| 8008 | Patroni REST API |
| 2379 | etcd client |
| 2380 | etcd peer |
| 5000 | HAProxy → primary |
| 5001 | HAProxy → replicas |



 
## Prerequisites

All steps run on **all 3 nodes** unless a node label is shown.

```bash
# Confirm Ubuntu version
lsb_release -a
# Must show: Ubuntu 24.04

# Note your network interface name — you will need it for Keepalived
ip link show
# Common names: enp1s0, ens3, eth0 — note yours down
```
 <br> 
 
## Step 1 — /etc/hosts (all nodes)

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.8.201  node1
192.168.8.202  node2
192.168.8.203  node3
192.168.8.200  pg-vip
EOF
```

 <br> 
 
## Step 2 — Firewall (all nodes)

```bash
sudo ufw allow 5432/tcp   # PostgreSQL
sudo ufw allow 8008/tcp   # Patroni REST API
sudo ufw allow 2379/tcp   # etcd client
sudo ufw allow 2380/tcp   # etcd peer
sudo ufw allow 5000/tcp   # HAProxy primary
sudo ufw allow 5001/tcp   # HAProxy replicas
sudo ufw allow proto vrrp # Keepalived VRRP
sudo ufw allow 22/tcp     # SSH — ensure you don't lock yourself out
sudo ufw enable
```
 <br> 
 
## Step 3 — Kernel Parameters (all nodes)

```bash
sudo tee /etc/sysctl.d/99-postgres-ha.conf <<'EOF'
net.ipv4.ip_nonlocal_bind = 1
net.core.somaxconn = 1024
EOF
sudo sysctl --system
```
 <br> 
 
## Step 4 — Time Sync Verification (all nodes)

etcd is highly sensitive to clock drift. Verify chrony is working before proceeding.

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

```bash
# Wait 30 seconds then check sync status
chronyc tracking
```

All nodes must show `System time` offset under **1 second** before continuing. If offset is large:
```bash
sudo chronyc makestep   # Force immediate time sync
chronyc tracking        # Re-check
```
 <br> 
 
## Step 5 — Install PostgreSQL 18 on Ubuntu 24.04 (all nodes)

```bash
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/pgdg.gpg >/dev/null
```
```bash
echo "deb [signed-by=/usr/share/keyrings/pgdg.gpg] \
  https://apt.postgresql.org/pub/repos/apt noble-pgdg main" \
  | sudo tee /etc/apt/sources.list.d/pgdg.list
```
```bash
sudo apt update
sudo apt install -y postgresql-18 postgresql-contrib postgresql-18-pgvector
```

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```
 <br> 
 
## Step 6 — Prepare PostgreSQL Data Directory (all nodes)

```bash
# Remove the default data directory (Patroni will initialise it)
sudo rm -rf /var/lib/postgresql/18/main

# Re-create empty directory with correct permissions
sudo mkdir -p /var/lib/postgresql/18/main
sudo chown -R postgres:postgres /var/lib/postgresql/
sudo chmod 700 /var/lib/postgresql/18/main
```
 <br> 
 
## Step 7 — Install etcd 3.5 from GitHub (all nodes)

> Ubuntu 24.04's apt `etcd` package is v3.3 which has Patroni compatibility issues. 

```bash
ETCD_VER=v3.5.17
curl -fsSL https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd.tar.gz
tar xzf /tmp/etcd.tar.gz -C /tmp/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo chmod +x /usr/local/bin/etcd /usr/local/bin/etcdctl
```
```bash
# Verify
etcd --version
etcdctl version
```

```bash
# Create etcd data dir and user
sudo mkdir -p /var/lib/etcd
sudo useradd --system --no-create-home --shell /bin/false etcd 2>/dev/null || true
sudo chown -R etcd:etcd /var/lib/etcd
```
 <br> 
 
## Step 8 — Configure etcd (per node)

> **Critical:** Each node gets its own config. Only `ETCD_NAME`, `ETCD_INITIAL_ADVERTISE_PEER_URLS`, and `ETCD_ADVERTISE_CLIENT_URLS` differ per node.


## Node 1

```bash
sudo tee /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME="node1"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.8.201:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.8.201:2379"
ETCD_INITIAL_CLUSTER="node1=http://192.168.8.201:2380,node2=http://192.168.8.202:2380,node3=http://192.168.8.203:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-etcd-cluster-01"
EOF
```


## Node 2

```bash
sudo tee /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME="node2"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.8.202:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.8.202:2379"
ETCD_INITIAL_CLUSTER="node1=http://192.168.8.201:2380,node2=http://192.168.8.202:2380,node3=http://192.168.8.203:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-etcd-cluster-01"
EOF
```


## Node 3

```bash
sudo tee /etc/etcd/etcd.conf <<'EOF'
ETCD_NAME="node3"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.8.203:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.8.203:2379"
ETCD_INITIAL_CLUSTER="node1=http://192.168.8.201:2380,node2=http://192.168.8.202:2380,node3=http://192.168.8.203:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-etcd-cluster-01"
EOF
```

> **Note:** `ETCD_INITIAL_CLUSTER_STATE="new"` is correct on all nodes for the **initial bootstrap**. If you ever need to re-join a node to an existing cluster, change that node's value to `"existing"`.


## etcd systemd service (all nodes)

```bash
sudo mkdir -p /etc/etcd
sudo tee /etc/systemd/system/etcd.service <<'EOF'
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network-online.target
Wants=network-online.target
[Service]
User=etcd
Group=etcd
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=5
LimitNOFILE=40000
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
```
 <br> 
 
## Step 9 — Install Patroni 3.x via pip (all nodes)

> **Why pip?** The apt package on Ubuntu 24.04 is Patroni 2.x. Patroni 3.x is required for PostgreSQL 18 support and current etcd3 integration.

```bash
sudo apt install -y python3-pip python3-venv python3-psycopg2 python3-dev libpq-dev
```
```bash
sudo python3 -m venv /opt/patroni
sudo /opt/patroni/bin/pip install --upgrade pip
sudo /opt/patroni/bin/pip install "patroni[etcd3]" psycopg2-binary
```
```bash
sudo ln -sf /opt/patroni/bin/patroni /usr/local/bin/patroni
sudo ln -sf /opt/patroni/bin/patronictl /usr/local/bin/patronictl
```
```bash
patroni --version
# Must show 3.x
```
 <br> 
 
## Step 10 — Install HAProxy & Keepalived (all nodes)

```bash
sudo apt install -y haproxy keepalived
```
 
## Step 11 — Configure HAProxy (all nodes)

```bash
sudo tee /etc/haproxy/haproxy.cfg <<'EOF'
global
    log /dev/log local0
    maxconn 4096
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s

defaults
    mode tcp
    log global
    option tcplog
    timeout connect 5s
    timeout client 30m
    timeout server 30m

listen postgres_primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.8.201:5432 check port 8008
    server node2 192.168.8.202:5432 check port 8008
    server node3 192.168.8.203:5432 check port 8008

listen postgres_replicas
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.8.201:5432 check port 8008
    server node2 192.168.8.202:5432 check port 8008
    server node3 192.168.8.203:5432 check port 8008

listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats refresh 5s
    stats show-node
EOF
```
```bash
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
 <br> 
 
## Step 12 — Configure Keepalived (per node)

> **First:** Check your network interface name on each node
> ```bash
> ip link show
> ```

 
## Node 1 — MASTER (priority 100)

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface enp1s0        # ← Replace with your interface name
    virtual_router_id 51
    priority 100
    advert_int 1
    virtual_ipaddress {
        192.168.8.200/24
    }
    track_script {
        check_haproxy
    }
}
EOF
```

# 
## Node 2 — BACKUP (priority 90)

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp1s0        # ← Replace with your interface name
    virtual_router_id 51
    priority 90
    advert_int 1
    virtual_ipaddress {
        192.168.8.200/24
    }
    track_script {
        check_haproxy
    }
}
EOF
```


## Node 3 — BACKUP (priority 80)

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp1s0        # ← Replace with your interface name
    virtual_router_id 51
    priority 80
    advert_int 1
    virtual_ipaddress {
        192.168.8.200/24
    }
    track_script {
        check_haproxy
    }
}
EOF
```

```bash
# Enable on all nodes
sudo systemctl enable keepalived
sudo systemctl restart keepalived
sudo systemctl status keepalived
```
 <br> 
 
## Step 13 — Configure Patroni (per node)


## Create config directory

```bash
sudo mkdir -p /etc/patroni
sudo chown postgres:postgres /etc/patroni
```


## Create .pgpass for postgres user (all nodes)

This is required for the post_init script to connect without a password in the environment.

```bash
sudo -u postgres tee /var/lib/postgresql/.pgpass <<'EOF'
*:*:*:postgres:StrongPostgresPassword
EOF

sudo chmod 600 /var/lib/postgresql/.pgpass
```

<br>

## Patroni config — Node 1

```bash
sudo tee /etc/patroni/patroni.yml <<'EOF'
scope: pg_cluster
namespace: /db/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.8.201:8008

etcd3:
  hosts:
    - 192.168.8.201:2379
    - 192.168.8.202:2379
    - 192.168.8.203:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: logical
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        # Set to 25% of your server RAM: 4GB RAM→1GB | 8GB RAM→2GB | 16GB RAM→4GB
        shared_buffers: 1GB
        # Set to 75% of your server RAM: 4GB RAM→3GB | 8GB RAM→6GB | 16GB RAM→12GB
        effective_cache_size: 3GB
        wal_log_hints: "on"      # Required for pg_rewind

      pg_hba:
        - local   all         all                              trust
        - host    all         all         127.0.0.1/32         trust
        - host    replication replicator  127.0.0.1/32         trust
        - host    replication replicator  192.168.8.0/24       scram-sha-256
        - host    all         all         192.168.8.0/24       scram-sha-256

  initdb:
    - encoding: UTF8
    - data-checksums

  post_init: /usr/local/bin/patroni_post_init.sh

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.8.201:5432
  data_dir: /var/lib/postgresql/18/main
  bin_dir: /usr/lib/postgresql/18/bin

  authentication:
    superuser:
      username: postgres
      password: StrongPostgresPassword
    replication:
      username: replicator
      password: StrongReplicationPassword

watchdog:
  mode: automatic        # Will use watchdog if available, skip if not
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```


## Patroni config — Node 2

```bash
sudo tee /etc/patroni/patroni.yml <<'EOF'
scope: pg_cluster
namespace: /db/
name: node2

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.8.202:8008

etcd3:
  hosts:
    - 192.168.8.201:2379
    - 192.168.8.202:2379
    - 192.168.8.203:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: logical
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        # Set to 25% of your server RAM: 4GB RAM→1GB | 8GB RAM→2GB | 16GB RAM→4GB
        shared_buffers: 1GB
        # Set to 75% of your server RAM: 4GB RAM→3GB | 8GB RAM→6GB | 16GB RAM→12GB
        effective_cache_size: 3GB
        wal_log_hints: "on"

      pg_hba:
        - local   all         all                              trust
        - host    all         all         127.0.0.1/32         trust
        # Required: Patroni checks replication status locally via 127.0.0.1
        - host    replication replicator  127.0.0.1/32         trust
        - host    replication replicator  192.168.8.0/24       scram-sha-256
        - host    all         all         192.168.8.0/24       scram-sha-256

  initdb:
    - encoding: UTF8
    - data-checksums

  post_init: /usr/local/bin/patroni_post_init.sh

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.8.202:5432
  data_dir: /var/lib/postgresql/18/main
  bin_dir: /usr/lib/postgresql/18/bin

  authentication:
    superuser:
      username: postgres
      password: StrongPostgresPassword
    replication:
      username: replicator
      password: StrongReplicationPassword

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```


## Patroni config — Node 3

```bash
sudo tee /etc/patroni/patroni.yml <<'EOF'
scope: pg_cluster
namespace: /db/
name: node3

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.8.203:8008

etcd3:
  hosts:
    - 192.168.8.201:2379
    - 192.168.8.202:2379
    - 192.168.8.203:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: logical
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        # Set to 25% of your server RAM: 4GB RAM→1GB | 8GB RAM→2GB | 16GB RAM→4GB
        shared_buffers: 1GB
        # Set to 75% of your server RAM: 4GB RAM→3GB | 8GB RAM→6GB | 16GB RAM→12GB
        effective_cache_size: 3GB
        wal_log_hints: "on"

      pg_hba:
        - local   all         all                              trust
        - host    all         all         127.0.0.1/32         trust
        # Required: Patroni checks replication status locally via 127.0.0.1
        - host    replication replicator  127.0.0.1/32         trust
        - host    replication replicator  192.168.8.0/24       scram-sha-256
        - host    all         all         192.168.8.0/24       scram-sha-256

  initdb:
    - encoding: UTF8
    - data-checksums

  post_init: /usr/local/bin/patroni_post_init.sh

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.8.203:5432
  data_dir: /var/lib/postgresql/18/main
  bin_dir: /usr/lib/postgresql/18/bin

  authentication:
    superuser:
      username: postgres
      password: StrongPostgresPassword
    replication:
      username: replicator
      password: StrongReplicationPassword

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```
 <br> 
 
## Step 14 — post_init Script (all nodes)

```bash
sudo tee /usr/local/bin/patroni_post_init.sh <<'EOF'
#!/bin/bash
# Runs once on the primary after cluster initialisation
# Uses .pgpass for authentication — no password in env
PSQL="/usr/lib/postgresql/18/bin/psql"
export HOME="/var/lib/postgresql"
$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS unaccent;"
$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
EOF
```
```bash
sudo chmod +x /usr/local/bin/patroni_post_init.sh
sudo chown postgres:postgres /usr/local/bin/patroni_post_init.sh
```

 <br> 

 
## Step 15 — Patroni systemd service (all nodes)

```bash
# Confirm patroni binary location
# Should return /usr/local/bin/patroni
which patroni
```
```bash
sudo tee /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target
[Service]
User=postgres
Group=postgres
Environment=PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
Restart=always
RestartSec=5
LimitNOFILE=65536
TimeoutStopSec=60
[Install]
WantedBy=multi-user.target
EOF
```
```bash
sudo chown -R postgres:postgres /etc/patroni
sudo chown root:root /etc/systemd/system/patroni.service
sudo chmod 644 /etc/systemd/system/patroni.service
sudo systemctl daemon-reload
```
 <br> 
 
## Step 16 — Watchdog Setup (all nodes)

```bash
sudo modprobe softdog
```
```bash
echo "softdog" | sudo tee /etc/modules-load.d/softdog.conf
```


Grant postgres user access:
```bash
sudo tee /etc/udev/rules.d/99-watchdog.rules <<'EOF'
KERNEL=="watchdog", OWNER="postgres", GROUP="postgres", MODE="0600"
EOF
```
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```
```bash
# Verify ownership
# Should show: postgres postgres
ls -la /dev/watchdog
```

 <br> 
 
## Step 17 — Bootstrap Order


## On all 3 nodes — start etcd first

```bash
sudo systemctl enable --now etcd
```

verify the etcd cluster is healthy on any node:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.8.201:2379,http://192.168.8.202:2379,http://192.168.8.203:2379 \
  endpoint health
```
```bash
# Expected output — all three must show "is healthy"
# http://192.168.8.201:2379 is healthy
# http://192.168.8.202:2379 is healthy
# http://192.168.8.203:2379 is healthy
```

> when all three etcd nodes are healthy continue

<br> 

## On Node 1 only — bootstrap the primary

```bash
sudo systemctl enable --now patroni
```
```bash
sudo journalctl -u patroni -f
```
```bash
patronictl -c /etc/patroni/patroni.yml list
```
> Watch the logs until node1 is elected leader

 <br> 


## On Node 2

```bash
sudo systemctl enable --now patroni
```

```bash
sudo journalctl -u patroni -f
```
```bash
# Wait for: "replica has joined the cluster" or similar
```
 
 <br> 

## On Node 3

```bash
sudo systemctl enable --now patroni
```

```bash
sudo journalctl -u patroni -f
```

<br> 

## Final cluster check (from any node)

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Expected output:
```
+ Cluster: pg_cluster (xxxxxxxxxx) +----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+--------+-----------------+---------+---------+----+-----------+
| node1  | 192.168.8.201:8008 | Leader | running |  1 |           |
| node2  | 192.168.8.202:8008 | Replica| running |  1 |         0 |
| node3  | 192.168.8.203:8008 | Replica| running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

<br> 
 
## Step 18 — Verify End-to-End

```bash
# 1. VIP is on node1 (the keepalived master)
ip addr show | grep 192.168.8.200

# 2. Primary returns false for pg_is_in_recovery
psql -h 192.168.8.200 -p 5000 -U postgres -c "SELECT pg_is_in_recovery();"
# Expected: f

# 3. Replicas return true
psql -h 192.168.8.200 -p 5001 -U postgres -c "SELECT pg_is_in_recovery();"
# Expected: t

# 4. Patroni health endpoints
curl -s http://192.168.8.201:8008/primary    # 200 on leader only
curl -s http://192.168.8.201:8008/replica    # 200 on replicas only
curl -s http://192.168.8.201:8008/health     # 200 on all running nodes

# 5. etcd cluster
ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.8.201:2379,http://192.168.8.202:2379,http://192.168.8.203:2379 \
  endpoint status --write-out=table

# 6. HAProxy stats
curl http://192.168.8.200:7000/
```
 <br> 
 
## Step 19 — Failover Test

```bash
# On node1 — stop patroni to trigger failover
sudo systemctl stop patroni

# Watch from another terminal
patronictl -c /etc/patroni/patroni.yml list

# A new leader should be elected within ~30 seconds
# Then restore node1
sudo systemctl start patroni
patronictl -c /etc/patroni/patroni.yml list
# node1 should rejoin as replica
```
 <br> 
 
## Step 20 — Application Setup (Odoo example)

Run on whichever node is the **primary** (Leader).

```bash
sudo -u postgres psql -c "CREATE USER odoo WITH PASSWORD 'your_strong_password';"
sudo -u postgres psql -c "CREATE DATABASE odoo OWNER odoo;"
sudo -u postgres psql -c "ALTER ROLE odoo CREATEDB;"
sudo -u postgres psql -c "ALTER USER odoo WITH SUPERUSER;"
sudo -u postgres psql -c "ALTER DATABASE odoo OWNER TO odoo;"
sudo -u postgres psql -c "DROP DATABASE odoo;"
sudo -u postgres psql -c "DROP DATABASE odoo WITH (FORCE);"
```

`/etc/odoo/odoo.conf`:
```ini
[options]
db_host = 192.168.8.200
db_port = 5000
db_user = odoo
db_password = your_strong_password
db_maxconn = 64
```

> **Why port 5000 and not 5432?**
> Port `5000` is the HAProxy listener that routes exclusively to the current primary, verified via Patroni's `/primary` health check. Connecting directly to port `5432` on a specific node bypasses HA entirely — if that node goes down, the application loses its database connection. With `192.168.8.200:5000`, HAProxy automatically redirects to the new primary after any failover with no changes needed in Odoo.
 <br> 
 
## Commands

```bash
# Cluster status
patronictl -c /etc/patroni/patroni.yml list

# Manual switchover (graceful — use for maintenance)
patronictl -c /etc/patroni/patroni.yml switchover pg_cluster

# Manual failover (forced)
patronictl -c /etc/patroni/patroni.yml failover pg_cluster

# Reload config without restart
patronictl -c /etc/patroni/patroni.yml reload pg_cluster

# Restart a specific node's postgres (not patroni)
patronictl -c /etc/patroni/patroni.yml restart pg_cluster node2

# View DCS config
patronictl -c /etc/patroni/patroni.yml show-config

# Edit DCS config live
patronictl -c /etc/patroni/patroni.yml edit-config

# Check replication lag
psql -h 192.168.8.200 -p 5000 -U postgres \
  -c "SELECT client_addr, state, sent_lsn, write_lsn, replay_lsn FROM pg_stat_replication;"
```
