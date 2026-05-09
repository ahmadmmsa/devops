
# PostgreSQL 18 HA on Ubuntu 26.04

Architecture Overview

```
                   Applications
                           |
                    192.168.8.200 (VIP)
                           |
                    HAProxy + Keepalived
                           |
        +------------------+------------------+
        |                  |                  |
     Node1              Node2              Node3
 192.168.8.201      192.168.8.202      192.168.8.203
 Patroni            Patroni            Patroni
 PostgreSQL 18      PostgreSQL 18      PostgreSQL 18
 etcd               etcd               etcd
 HAProxy            HAProxy            HAProxy
 Keepalived         Keepalived         Keepalived
```

<br>

## Installation

```bash
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/pgdg.gpg >/dev/null
```
```bash
echo "deb [signed-by=/usr/share/keyrings/pgdg.gpg] https://apt.postgresql.org/pub/repos/apt resolute-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
```
```bash
sudo apt update
```
```bash
sudo apt install -y postgresql-18 postgresql-contrib postgresql-18-pgvector etcd-server etcd-client patroni python3-etcd3 python3-psycopg2 haproxy keepalived jq chrony
```

<br>

## Prepare PostgreSQL Data Directory

> Stop the default postgres service so it doesn't lock the files
```bash
sudo systemctl stop postgresql
```
```bash
sudo systemctl disable postgresql
```

> Remove the default data (Patroni will recreate this)
```bash
sudo rm -rf /var/lib/postgresql/18/main
```

> Create the empty directory
```bash
sudo mkdir -p /var/lib/postgresql/18/main
```

> Give the postgres user ownership
```bash
sudo chown -R postgres:postgres /var/lib/postgresql/18/
sudo chmod 700 /var/lib/postgresql/18/main
```

<br>


## Node servers Configuration

/etc/hosts

```bash
sudo tee -a /etc/hosts <<EOF
192.168.8.201  node1
192.168.8.202  node2
192.168.8.203  node3
192.168.8.200 pg-vip
EOF
```

firewall ports to open

```bash
sudo ufw allow 5432/tcp
sudo ufw allow 8008/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow proto vrrp
sudo ufw allow 5000/tcp
sudo ufw allow 5001/tcp
```
config network connections

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-postgres-ha.conf
net.ipv4.ip_nonlocal_bind = 1
net.core.somaxconn = 1024
EOF
```

```bash
sudo sysctl --system
```

<br>

## etcd Configuration

```bash
sudo tee /etc/default/etcd <<EOF
ETCD_NAME="node1"
ETCD_DATA_DIR="/var/lib/etcd"

ETCD_INITIAL_CLUSTER="node1=http://192.168.8.201:2380,node2=http://192.168.8.202:2380,node3=http://192.168.8.203:2380"
ETCD_INITIAL_CLUSTER_STATE="new"

ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

# change on each node
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.8.201:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.8.201:2379"
EOF
```

<br>

## HAProxy & Keepalived Configuration

```bash
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    maxconn 4096

defaults
    mode tcp
    timeout connect 5s
    timeout client 30m
    timeout server 30m

listen postgres_primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server node1 192.168.8.201:5432 check port 8008
    server node2 192.168.8.202:5432 check port 8008
    server node3 192.168.8.203:5432 check port 8008

listen postgres_replicas
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server node1 192.168.8.201:5432 check port 8008
    server node2 192.168.8.202:5432 check port 8008
    server node3 192.168.8.203:5432 check port 8008    
EOF
```

```bash
sudo tee /etc/keepalived/keepalived.conf <<EOF
global_defs {
    enable_script_security
    script_user root
}
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    state MASTER          # BACKUP on nodes 2 & 3
    interface enp1s0        # Adjust to your interface name (check with: ip link)
    virtual_router_id 51
    priority 100          # 90 on node2, 80 on node3
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




<br>


## Patroni YAML Configuration

```bash
sudo tee /etc/patroni/patroni.yml <<EOF
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
        shared_buffers: 2GB
        effective_cache_size: 6GB

      pg_hba:
        - local all all trust
        - host replication replicator 192.168.8.0/24 md5
        - host all all 192.168.8.0/24 md5

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

EOF
```


set env var
```bash
echo "export PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml" >> ~/.bashrc
```

Post-init Script
```bash
sudo tee /usr/local/bin/patroni_post_init.sh <<'EOF'
#!/bin/bash
export PGPASSWORD="StrongPostgresPassword"

PSQL="/usr/lib/postgresql/18/bin/psql"

$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS unaccent;"
$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
$PSQL -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
EOF
```

```bash
sudo chmod +x /usr/local/bin/patroni_post_init.sh
```

```bash
sudo chown postgres:postgres /usr/local/bin/patroni_post_init.sh
```

<br>


## systemd service 

```bash
which patroni
```
> if /usr/bin/patroni good else change ExecStart

```bash
sudo tee /etc/systemd/system/patroni.service <<EOF
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target

[Service]
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

set patroni permission


>Give postgres ownership of the config directory

```bash
sudo mkdir -p /etc/patroni
sudo chown -R postgres:postgres /etc/patroni
```
>systemd unit file permission
```bash
sudo chown root:root /etc/systemd/system/patroni.service
sudo chmod 644 /etc/systemd/system/patroni.service
```
>Reload systemd
```bash
sudo systemctl daemon-reload
```


<br>


## Initialization Order

Node 1

```bash
sudo systemctl enable --now etcd chrony haproxy keepalived
```

```bash
sudo -u postgres patroni /etc/patroni/patroni.yml
```
> Wait until node1 becomes leader..

```bash
sudo systemctl enable --now patroni
```

<br>

Nodes 2 and 3

```bash
sudo systemctl start patroni
```

```bash
sudo systemctl enable --now etcd chrony haproxy keepalived patroni
```


<br>

## Verification

Check Patroni Cluster: 
```bash
patronictl -c /etc/patroni/patroni.yml list
```
Check etcd Health: 
```bash
etcdctl endpoint health --cluster
```

Check VRRP (Keepalived):

```bash
ip addr show enp1s0 
```
> vIP 192.168.8.200 should appear on the Leader.

```bash
curl http://localhost:8008/primary
```

```bash
psql -h 192.168.8.200 -p 5000 -U postgres -c "SELECT pg_is_in_recovery();"
```

> Expected output: f

<br>

## Failover Test

Node 1

```bash
sudo systemctl stop patroni
```

```bash
patronictl list
```

<br>

## Postgresql app config

odoo 19 setup
```bash
patronictl -c /etc/patroni/patroni.yml exec node1 -- psql -c "CREATE USER odoo WITH PASSWORD 'your_strong_password';"
patronictl -c /etc/patroni/patroni.yml exec node1 -- psql -c "CREATE DATABASE odoo OWNER odoo;"
```

 odoo.conf

```bash
[options]
db_host = 192.168.8.200
db_port = 5000
db_user = odoo
db_password = your_strong_password
db_maxconn = 64
```
