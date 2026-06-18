




# MinIO
the industry standard for self-hosted, S3-compatible object storage.

Prerequisites:
```bash
sudo apt install -y docker.io docker-compose-v2 docker-buildx
```

```bash
# Create directory structure for MinIO configuration and persistent data
mkdir -p ~/minio-server/data

# Generate the Docker Compose file
cat <<EOF > ~/minio-server/docker-compose.yml
services:
  minio:
    image: quay.io/minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=StrongPassword123!
    volumes:
      - ./data:/data
    ports:
      - "9000:9000" # API Port
      - "9001:9001" # Web Console Port
    restart: unless-stopped
EOF

# Navigate to the directory and start the container in detached mode
cd ~/minio-server
docker compose up -d
```
web console at http://<your-server-ip>:9001 to create buckets and manage access keys



<br>

# NFS (Network File System)
if you require POSIX-compliant file mounting

```bash
# Update package lists and install the NFS server daemon
sudo apt update
sudo apt install -y nfs-kernel-server

# Create the directory you want to share
sudo mkdir -p /mnt/nfs_share

# Remove restrictive permissions so client machines can read/write
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share

# Append the export configuration (Replace 192.168.8.0/24 with actual client subnet)
echo "/mnt/nfs_share 192.168.8.0/24(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports

# Apply export configuration
sudo exportfs -a

# Restart and enable the service to persist across reboots
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

docker compose example

```yml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)
    volumes:
      - odoo-filestore:/var/lib/odoo
      - ./odoo.conf:/etc/odoo/odoo.conf:ro
      - ./extra-addons:/mnt/extra-addons
    environment:
      - ODOO_RC=/etc/odoo/odoo.conf
volumes:
  odoo-filestore:
    driver: local
    driver_opts:
      type: nfs
      # hard and timeout parameters. prevents the container from locking up completely if the network connection to the NFS server briefly drops.
      # o: addr=192.168.8.70,nfsvers=4,rw,hard,timeo=600,retrans=2
      o: addr=<NFS-SERVER-IP>,nfsvers=4,rw
      device: ":/srv/odoo/filestore"
```

<br>

optional: Configure Permanent Mount Settings


```bash
echo "192.168.8.70:/mnt/storage /mnt/odoo_data nfs rw,hard,timeo=600,retrans=2,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

```bash
# verify
df -h
```

```bash
# rw       Read/Write
# hard     if the connection drops, processes will wait for the server to come.
# _netdev  Tells OS to wait until the network is fully up and running.
# noresvport  use any available unreserved network port
# tcp alongside vers=4 is redundant
# nfsvers=n linux parameter, vers=n cross-platform compatibility macos,solaris,BSD.
```

docker compose yml example

```yml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)
    volumes:
      - /mnt/odoo_data:/var/lib/odoo   # Points directly to the fstab mount point
      - ./odoo.conf:/etc/odoo/odoo.conf:ro
      - ./extra-addons:/mnt/extra-addons
    environment:
      - ODOO_RC=/etc/odoo/odoo.conf
```




# Object Storage Testing Lab (Ubuntu 24.04 LTS)

Last reviewed: June 2026

## Purpose

This document provides hardened single-node test installations for:

* Garage (v2.x)
* SeaweedFS (v4.32)

The goal is:

* Ubuntu 24.04 compatibility
* Non-root services
* Systemd hardening
* Basic firewalling
* Local S3 testing

---

# 1. Common Preparation

Update system:

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

Install required packages:

```bash
sudo apt-get install -y \
  curl \
  wget \
  tar \
  unzip \
  openssl \
  ufw
```

Enable firewall:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
```

---

# 2. Garage v2.x

## Create Service User

```bash
sudo useradd \
  --system \
  --home /var/lib/garage \
  --shell /usr/sbin/nologin \
  garage
```

## Create Directories

```bash
sudo mkdir -p \
  /etc/garage \
  /var/lib/garage/meta \
  /var/lib/garage/data

sudo chown -R garage:garage \
  /var/lib/garage
```

## Download Garage

Replace VERSION as needed.

```bash
VERSION="2.3.0"

curl -LO \
https://garagehq.deuxfleurs.fr/_releases/${VERSION}/x86_64-unknown-linux-musl/garage

chmod +x garage

sudo mv garage /usr/local/bin/
```

Verify:

```bash
garage --version
```

## Generate RPC Secret

```bash
openssl rand -hex 32
```

Save the generated value.

Example:

```text
4b0b8ab8b02f4f7f4b6b6c1d5d38d9d68f90e3c1e2d6cbf7d15db43c7dcb8fa8
```

## Garage Configuration

Create:

```bash
sudo nano /etc/garage/garage.toml
```

Contents:

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"

db_engine = "sqlite"

replication_factor = 1

rpc_bind_addr = "127.0.0.1:3901"
rpc_public_addr = "127.0.0.1:3901"

rpc_secret = "REPLACE_WITH_SECRET"

[s3_api]
api_bind_addr = "0.0.0.0:3900"
s3_region = "us-east-1"

[s3_web]
bind_addr = "127.0.0.1:3902"
root_domain = ".web.local"
```

Set ownership:

```bash
sudo chown root:garage /etc/garage/garage.toml
sudo chmod 640 /etc/garage/garage.toml
```

## Systemd Service

Create:

```bash
sudo nano /etc/systemd/system/garage.service
```

```ini
[Unit]
Description=Garage Object Storage
After=network-online.target
Wants=network-online.target

[Service]
Type=simple

User=garage
Group=garage

ExecStart=/usr/local/bin/garage server -c /etc/garage/garage.toml

Restart=always
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true

ReadWritePaths=/var/lib/garage

MemoryDenyWriteExecute=true
LockPersonality=true
RestrictRealtime=true
RestrictNamespaces=true

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now garage
```

Check:

```bash
systemctl status garage
```

Logs:

```bash
journalctl -u garage -f
```

## Firewall

Allow S3 endpoint:

```bash
sudo ufw allow 3900/tcp
```

Verify:

```bash
ss -tulpn | grep garage
```

---

# 3. SeaweedFS 4.32

## Create Service User

```bash
sudo useradd \
  --system \
  --home /var/lib/seaweedfs \
  --shell /usr/sbin/nologin \
  seaweedfs
```

## Create Directories

```bash
sudo mkdir -p \
  /var/lib/seaweedfs/master \
  /var/lib/seaweedfs/volume

sudo chown -R seaweedfs:seaweedfs \
  /var/lib/seaweedfs
```

## Download SeaweedFS

```bash
SEAWEED_VERSION="4.32"

curl -LO \
https://github.com/seaweedfs/seaweedfs/releases/download/${SEAWEED_VERSION}/linux_amd64.tar.gz
```

Extract:

```bash
tar -xzf linux_amd64.tar.gz weed

sudo mv weed /usr/local/bin/

rm linux_amd64.tar.gz
```

Verify:

```bash
weed version
```

## Systemd Service

Create:

```bash
sudo nano /etc/systemd/system/seaweedfs.service
```

```ini
[Unit]
Description=SeaweedFS Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple

User=seaweedfs
Group=seaweedfs

ExecStart=/usr/local/bin/weed server \
  -dir=/var/lib/seaweedfs/volume \
  -mdir=/var/lib/seaweedfs/master \
  -s3 \
  -s3.port=8333 \
  -master.port=9333 \
  -volume.port=8080

Restart=always
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true

ReadWritePaths=/var/lib/seaweedfs

MemoryDenyWriteExecute=true
LockPersonality=true
RestrictRealtime=true
RestrictNamespaces=true

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now seaweedfs
```

Verify:

```bash
systemctl status seaweedfs
```

Logs:

```bash
journalctl -u seaweedfs -f
```

## Firewall

Allow S3 API:

```bash
sudo ufw allow 8333/tcp
```

Optional administration ports:

```bash
sudo ufw allow 9333/tcp
sudo ufw allow 8080/tcp
```

Verify:

```bash
ss -tulpn | grep weed
```

---

# 4. Functional Testing

## Garage

Check API:

```bash
curl http://localhost:3900
```

Expected:

```xml
<ListAllMyBucketsResult>
```

or S3 authentication response.

---

## SeaweedFS

Check S3 endpoint:

```bash
curl http://localhost:8333
```

Check master:

```bash
curl http://localhost:9333/dir/status
```

---

# 5. Resource Comparison

## Garage

Advantages:

* Very lightweight
* Small memory footprint
* Simple deployment
* Excellent homelab option

Disadvantages:

* Smaller ecosystem
* More manual administration

---

## SeaweedFS

Advantages:

* Scales extremely well
* Fast small-object workloads
* Mature distributed architecture

Disadvantages:

* More moving parts
* More operational complexity

---

# 6. Recommendation

For Ubuntu 24.04 single-node S3 testing:

1. Garage
2. SeaweedFS

For homelab and small production environments, Garage is usually the simplest modern replacement for MinIO.

For larger distributed clusters, SeaweedFS is generally the better long-term platform.
