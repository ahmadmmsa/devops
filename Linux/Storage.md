




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