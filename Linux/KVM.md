

- [Network](#network)
- [qemu-img commands](#qemu-img-commands)
- [virt-customize commands](#virt-customize-commands)
- [Automate Vm creation](#automate-vm-creation)
- [other](#other)


## virsh commands

```bash
virsh list                  # active/running
virsh list --all            # all VMs
virsh net-list --all        # list network profiles

virsh start <vm_name>
virsh shutdown <vm_name>
virsh reboot <vm-name>
virsh destroy <vm_name>
virsh suspend <vm_name>
virsh resume <vm_name>

virsh autostart <vm_name>               #start automatically
virsh autostart --disable <vm_name>     # to disable

# Delete
# list all the block devices (vhds,CD-ROM,ISOs) attached
# dom "domain", blklist "block device list"
virsh domblklist <vm_name>
# Deletes registration/config XML profile
virsh undefine <vm_name>
# remove all boot drives and disks for that vm
virsh undefine <vm_name> --remove-all-storage
virsh undefine <vm-name> --remove-all-storage --snapshots-metadata
# Delete the physical file
rm /var/lib/libvirt/images/vm_name.qcow2

```

get a cloud image:
```bash
aria2c -x 16 -s 16 https://*.img
```
 - https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img
- https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img
- https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2
- https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2
- https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
- https://download.fedoraproject.org/pub/fedora/linux/releases/44/Cloud/x86_64/images/Fedora-Cloud-Base-Generic-44-1.7.x86_64.qcow2


# install

stable libvirt stack:
```bash
sudo apt install ovmf qemu-system-x86 qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager cloud-image-utils libguestfs-tools osinfo-db libosinfo-bin -y
```

add user to libvirt:
```bash
sudo usermod -aG libvirt $USER
newgrp libvirt
```
<br>

# Network

## use libvirt Default bridge (virbr0)

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```
> and in virt-install use the flag --network network=default 

## Creat a Bridge using networkmanager(desktop) optional
```bash
#Check interface Look for: enp3s0, eth0, ens33
ip link
#Create bridge br0
sudo nmcli connection add type bridge ifname br0 con-name br0
#Assign IP method (DHCP recommended for homelab)
sudo nmcli connection modify br0 ipv4.method auto
sudo nmcli connection modify br0 ipv6.method auto
#Attach your physical NIC to the bridge
#disable auto IP on NIC
sudo nmcli connection modify enp4s0 ipv4.method disabled
sudo nmcli connection modify enp4s0 ipv6.method ignore
#enslave it to bridge
sudo nmcli connection add type bridge-slave ifname enp4s0 master br0
#Bring everything up & Verify:
sudo nmcli connection up br0
sudo nmcli connection up bridge-slave-enp4s0
ip a show br0
```
```bash
#edit default network
sudo virsh net-edit default
```
create VM NIC:
```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```
```bash
#check connection
nmcli connection show
```

## Create a Network Definition for libvirt (optional)

```bash
cat <<EOF > /tmp/bridge.xml
<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
</network>
EOF
```
custom isolated network:

```xml
<network>
  <name>custom-isolated-net</name>
  <bridge name='virbr10'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
```
internet access <forward mode='nat'/>
```xml
<network>
  <name>custom-public-net</name>
  <forward mode='nat'/> <bridge name='virbr10'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
```
```bash
# Define the network from the XML file
sudo virsh net-define /tmp/bridge.xml
# Start the network
sudo virsh net-start br0
# Configure the network to start automatically when the host boots
sudo virsh net-autostart br0
# virefy
sudo virsh net-list --all

```





<br>

## Create cloud-init config files

### meta-data
```bash
cat <<EOF > meta-data
instance-id: $(uuidgen || echo i-abcdefg)
local-hostname: vm-$(uuidgen | tr -d '-' | cut -c1-8)
EOF
```

### user-data
```bash
cat <<EOF > user-data.yaml
#cloud-config
users:
  - name: ahmad
    shell: /bin/bash
    lock_passwd: false
    passwd: "hashed-password"
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    groups: users, admin
    ssh_authorized_keys:
      - ssh-ed25519 ... ahmad@homelab
growpart:
  mode: auto
  devices: ['/']

resize_rootfs: true      
EOF
```

<br>

### runcmd Examples:

install and enable a service
```bash
package_update: true

runcmd:
  - apt-get install -y nginx
  - systemctl enable nginx
  - systemctl start nginx
```
deploy app from git
```bash
runcmd:
  - apt-get install -y git
  - git clone https://github.com/example/app.git /opt/app
  - cd /opt/app && ./install.sh
```
create user environment
```bash
runcmd:
  - useradd -m devuser
  - mkdir -p /home/devuser/.ssh
  - echo "ssh-rsa AAAA..." > /home/devuser/.ssh/authorized_keys
  - chown -R devuser:devuser /home/devuser/.ssh
```
kubernetes Prerequisites
```bash
write_files:
  - path: /etc/modules-load.d/k8s.conf
    content: |
      overlay
      br_netfilter

  - path: /etc/sysctl.d/99-kubernetes.conf
    content: |
      net.bridge.bridge-nf-call-iptables = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward = 1

runcmd:
  - swapoff -a
  - sysctl --system
```

### bootcmd Examples:

sysctl / kernel tuning
```bash
bootcmd:
  - sysctl -w net.core.somaxconn=1024
  - sysctl -w vm.swappiness=10

# somaxconn: controls the Maximum Socket Connections
# To prevent "Connection Refused" errors during traffic spikes.

# swappiness: controls the Swap Tendency
# running a Database (PostgreSQL, MongoDB) or In-Memory Cache (Redis)
# To keep apps in RAM and avoid slow disk-access speeds.  
```

ensure module loaded
```bash
bootcmd:
  - modprobe br_netfilter
  - sysctl -w net.bridge.bridge-nf-call-iptables=1
```

fix datasource / networking quirk
```bash
bootcmd:
  - ip link set dev eth0 up
  - dhclient -v eth0
```

<br>

> to generate hashed yescrypt password & ssh key

```bash
mkpasswd -m yescrypt "password"
```
```bash
ssh-keygen -t ed25519 -C "user"
```

<br>

### network-config

assign macaddress & ip (recommended)
```bash
cat <<EOF > network-config
version: 2
renderer: networkd
ethernets:
  net0:
    match:
      macaddress: 52:54:00:c8:ed:73
    set-name: eth0
    dhcp4: no
    addresses:
      - 192.168.8.200/24
    routes:
      - to: default
        via: 192.168.8.1
    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]
EOF
```

or interface specific
```bash
cat <<EOF > network-config
version: 2
renderer: networkd
ethernets:
  ens2:
    dhcp4: no
    addresses:
      - 192.168.8.202/24
    routes:
      - to: default
        via: 192.168.8.1
    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]
EOF
```

<br>

## Build cloud-init ISO
```bash
cloud-localds -v seed.iso user-data.yaml meta-data -N network-config
```

<br>

Create VM using virt-install

mac assigned
```bash
virt-install \
  --name server-mac-config --memory 2048 --vcpus 2 \
  --disk path=./ubuntu-26.04-server-3.img,device=disk,bus=virtio --disk path=./seed.iso,device=cdrom \
  --network bridge=br0,mac=52:54:00:c8:ed:73 \
  --graphics none --osinfo generic --import
```

interface specific
```bash
virt-install \
  --name my-cloud-vm2 --memory 2048 --vcpus 2 \
  --disk path=./ubuntu-26.04-server.img,device=disk,bus=virtio --disk path=./seed.iso,device=cdrom \
  --network bridge=br0 \
  --graphics none --osinfo generic --import
```

<br>


# qemu-img commands


```bash
# Create qcow2
qemu-img create -f qcow2 -F qcow2 -b \ 
 ubuntu-26.04-server-cloudimg-amd64.img ubuntu-26.04-server.qcow2

# Change img size
qemu-img resize /var/lib/libvirt/images/vms/disk.qcow2 +20G
#in vm run to expand partition
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
df -h /

qemu-img check /var/lib/libvirt/images/vms/disk.qcow2

qemu-img convert -f raw -O qcow2 ubuntu-base.img ubuntu-base.qcow2
```

<br>

# virt-customize commands

```bash
virt-customize -a /var/lib/libvirt/images/vms/disk.qcow2 \
  --run-command 'useradd -m -s /bin/bash admin' \
  --run-command 'echo "admin:SecretPassword123" | chpasswd' \
  --run-command 'usermod -aG sudo admin'

virt-customize -a /var/lib/libvirt/images/vms/disk.qcow2 \
  --upload /path/to/local/01-netcfg.yaml:/etc/netplan/01-netcfg.yaml

virt-customize -a /var/lib/libvirt/images/vms/disk.qcow2 \
  --install curl,git,htop    
```





<br>

# Automate VM creation

```bash
sudo chmod +x create-vm.sh
```
```bash
sudo ./create-vm.sh
```

```bash
cat <<'CREATE_NEW_VM_SCRIPT' > create-vm.sh
#!/bin/bash
set -euo pipefail

# Paths
# Base images live in BASE_IMG_DIR (top-level only, no vm- files)
# VM disks and seeds go into VM_STORE (separate subdir)
BASE_IMG_DIR="/var/lib/libvirt/images"
VM_STORE="/var/lib/libvirt/images/vms"
mkdir -p "$VM_STORE"

# Dependency check
REQUIRED_CMDS=(virt-install virsh cloud-localds qemu-img uuidgen openssl)
for cmd in "${REQUIRED_CMDS[@]}"; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "ERROR: Required command '$cmd' not found. Install it and retry."
        exit 1
    fi
done

# Image selection — only top-level base images, never vm- overlays
mapfile -t IMG_FILES < <(
    find "$BASE_IMG_DIR" -maxdepth 1 -type f \( -name "*.img" -o -name "*.qcow2" \) \
        ! -name "vm-*" | sort
)

VM_IMG=""

if [ ${#IMG_FILES[@]} -eq 0 ]; then
    echo "No base images found in $BASE_IMG_DIR."
    read -p "Would you like to download a fresh image? (y/n): " DL_CONFIRM
    [[ ! "$DL_CONFIRM" =~ ^[Yy]$ ]] && { echo "No image selected. Exiting."; exit 1; }
else
    echo ""
    echo "Available base images:"
    for i in "${!IMG_FILES[@]}"; do
        echo "  $((i+1))) ${IMG_FILES[$i]}"
    done
    echo "  $((${#IMG_FILES[@]}+1))) Download a fresh image"
    echo ""
    read -p "Select [1-$((${#IMG_FILES[@]}+1))]: " IMG_CHOICE

    if ! [[ "$IMG_CHOICE" =~ ^[0-9]+$ ]] || (( IMG_CHOICE < 1 || IMG_CHOICE > ${#IMG_FILES[@]}+1 )); then
        echo "Invalid selection."
        exit 1
    fi

    if (( IMG_CHOICE != ${#IMG_FILES[@]}+1 )); then
        VM_IMG="${IMG_FILES[$((IMG_CHOICE-1))]}"
    fi
fi

if [ -z "$VM_IMG" ]; then
    echo ""
    echo "Select image to download:"
    echo "  1) Ubuntu 24.04 LTS"
    echo "  2) Ubuntu 26.04 LTS"
    echo "  3) Debian 12 (Bookworm)"
    echo "  4) Rocky 9 (RHEL-compatible)"
    echo "  5) AlmaLinux 9 (RHEL-compatible)"
    echo "  6) Fedora 41"
    echo ""
    read -p "Select [1-6]: " DL_CHOICE
    case $DL_CHOICE in
        1) DL_URL="https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img" ;;
        2) DL_URL="https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img" ;;
        3) DL_URL="https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2" ;;
        4) DL_URL="https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2" ;;
        5) DL_URL="https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2" ;;
        6) DL_URL="https://download.fedoraproject.org/pub/fedora/linux/releases/44/Cloud/x86_64/images/Fedora-Cloud-Base-Generic-44-1.7.x86_64.qcow2" ;;
        *) echo "Invalid selection."; exit 1 ;;
    esac

    VM_IMG="$BASE_IMG_DIR/$(basename "$DL_URL")"
    echo "Downloading to $BASE_IMG_DIR ..."

    if command -v aria2c &>/dev/null; then
        echo "Using aria2c for faster download..."
        aria2c -x 8 -s 8 -o "$(basename "$DL_URL")" -d "$BASE_IMG_DIR" "$DL_URL" \
            || { echo "Download failed."; exit 1; }
    else
        read -p "aria2c not found (faster downloads). Install it? (y/n): " INSTALL_ARIA
        if [[ "$INSTALL_ARIA" =~ ^[Yy]$ ]]; then
            apt-get install -y aria2 2>/dev/null || true
            if command -v aria2c &>/dev/null; then
                aria2c -x 8 -s 8 -o "$(basename "$DL_URL")" -d "$BASE_IMG_DIR" "$DL_URL" \
                    || { echo "Download failed."; exit 1; }
            else
                wget -O "$VM_IMG" "$DL_URL" || { echo "Download failed."; exit 1; }
            fi
        else
            wget -O "$VM_IMG" "$DL_URL" || { echo "Download failed."; exit 1; }
        fi
    fi
fi

echo ""
echo "Using base image: $VM_IMG"


# Inspect image to detect OS (filename-independent)
echo "Inspecting image, please wait..."

INSPECTOR_XML=$(virt-inspector -a "$VM_IMG" 2>/dev/null)

if [ -z "$INSPECTOR_XML" ]; then
    echo "WARNING: virt-inspector returned no output. Falling back to generic."
    OS_FAMILY="generic"
    OSINFO="generic"
else
    DISTRO=$(echo "$INSPECTOR_XML"  | grep -oP '(?<=<distro>)[^<]+'        | head -1 | tr '[:upper:]' '[:lower:]')
    VERSION=$(echo "$INSPECTOR_XML" | grep -oP '(?<=<major_version>)[^<]+' | head -1)
    MINOR=$(echo "$INSPECTOR_XML"   | grep -oP '(?<=<minor_version>)[^<]+' | head -1)

    echo "Detected: distro=$DISTRO version=$VERSION minor=$MINOR"

    # Build OSINFO string and OS_FAMILY
    case "$DISTRO" in
        ubuntu)
            OS_FAMILY="ubuntu"
            # Ubuntu uses major.minor format e.g. ubuntu24.04
            MINOR_PAD=$(printf "%02d" "$MINOR")
            OSINFO="ubuntu${VERSION}.${MINOR_PAD}"
            ;;
        debian)
            OS_FAMILY="debian"
            OSINFO="debian${VERSION}"
            ;;
        rhel)
            OS_FAMILY="rhel"
            OSINFO="rhel${VERSION}.0"
            ;;
        rocky*)
            OS_FAMILY="rhel"
            OSINFO="rocky${VERSION}"
            ;;
        alma*|almalinux*)
            OS_FAMILY="rhel"
            OSINFO="almalinux${VERSION}"
            ;;
        fedora)
            OS_FAMILY="rhel"
            OSINFO="fedora${VERSION}"
            ;;
        centos)
            OS_FAMILY="rhel"
            OSINFO="centos-stream${VERSION}"
            ;;
        *)
            echo "WARNING: Unknown distro '$DISTRO' from virt-inspector."
            OS_FAMILY="generic"
            OSINFO="generic"
            ;;
    esac
fi
echo "OS family : $OS_FAMILY"
echo "OSINFO    : $OSINFO"

# Validate OSINFO against local osinfo-db
if [ "$OSINFO" != "generic" ]; then
    if command -v osinfo-query &>/dev/null; then
        if ! osinfo-query os short-id="$OSINFO" &>/dev/null; then
            echo ""
            echo "WARNING: '$OSINFO' not found in local osinfo-db."
            echo "Your osinfo-db may be outdated. To update:"
            echo ""
            if command -v apt &>/dev/null; then
                echo "  sudo apt install --reinstall osinfo-db libosinfo-bin"
            elif command -v dnf &>/dev/null; then
                echo "  sudo dnf install osinfo-db-tools"
            fi
            echo ""
            # Show closest available matches
            echo "Closest available entries:"
            osinfo-query os | grep -i "$DISTRO" \
                | awk '{print $1}' | sort -V | tail -5 || true
            echo ""
            # Attempt to find the latest known version of this distro as a better fallback
            BEST_MATCH=$(osinfo-query os | grep -oP "(?<=\s)${DISTRO}[\d.()]+" \
                | sort -V | tail -1)
            if [ -n "$BEST_MATCH" ]; then
                echo "Using closest match: $BEST_MATCH"
                OSINFO="$BEST_MATCH"
            else
                echo "Note: 'generic' disables OS-specific CPU/driver optimizations."
                read -rp "Continue with generic? (y/N): " CONTINUE_GENERIC
                [[ ! "$CONTINUE_GENERIC" =~ ^[Yy]$ ]] && exit 1
                OSINFO="generic"
            fi
        else
            echo "osinfo '$OSINFO' confirmed in local db."
        fi
    else
        echo "WARNING: osinfo-query not installed, using generic."
        OSINFO="generic"
    fi
fi
echo "Final osinfo: $OSINFO"


# User / SSH inputs
REAL_USER=${SUDO_USER:-$USER}
ACTUAL_HOME=$(getent passwd "$REAL_USER" | cut -d: -f6)
DOT_SSH="$ACTUAL_HOME/.ssh"

echo ""
echo "Running as: $(whoami) | Original user: $REAL_USER | Home: $ACTUAL_HOME"
echo ""

read -p "VM Name [leave empty for random]: " VM_NAME
VM_NAME="${VM_NAME:-$(uuidgen | tr -d '-' | cut -c1-8)}"

read -p "Username [default: sysadmin]: " USR_NAME
USR_NAME=${USR_NAME:-sysadmin}

read -sp "User Password: " USR_PASSWRD
echo ""

read -p "Enter IP Last Octet (192.168.8.X): " IP_NUM
if ! [[ "$IP_NUM" =~ ^[0-9]+$ ]] || (( IP_NUM < 1 || IP_NUM > 254 )); then
    echo "Invalid IP octet: $IP_NUM"
    exit 1
fi

read -p "Memory in MB [default: 2048]: " VM_MEMORY
VM_MEMORY=${VM_MEMORY:-2048}

read -p "VCPUs [default: 2]: " VM_CPU
VM_CPU=${VM_CPU:-2}

read -p "Disk Size [default: 20G]: " VM_DISK_SIZE
VM_DISK_SIZE=${VM_DISK_SIZE:-20G}

read -p "Paste SSH public key (or leave blank to auto-detect/generate): " USR_SSHKEY
echo ""

KEY_FILE=""

if [ -z "$USR_SSHKEY" ]; then
    if [ -f "$DOT_SSH/id_ed25519.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_ed25519.pub")
        KEY_FILE="$DOT_SSH/id_ed25519"
        echo "Found existing ed25519 key: $KEY_FILE"
    elif [ -f "$DOT_SSH/id_rsa.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_rsa.pub")
        KEY_FILE="$DOT_SSH/id_rsa"
        echo "Found existing RSA key: $KEY_FILE"
    fi
fi

if [ -z "$USR_SSHKEY" ]; then
    echo "No SSH key found."
    read -p "Generate one now? (y/n): " GEN_CONFIRM
    if [[ "$GEN_CONFIRM" =~ ^[Yy]$ ]]; then
        echo "  1) ed25519 (recommended)"
        echo "  2) rsa"
        read -p "Key type [1-2, default: 1]: " KEY_CHOICE
        case ${KEY_CHOICE:-1} in
            2) KEY_TYPE="rsa";     KEY_FILE="$DOT_SSH/id_rsa" ;;
            *) KEY_TYPE="ed25519"; KEY_FILE="$DOT_SSH/id_ed25519" ;;
        esac
        mkdir -p "$DOT_SSH"
        chown "$REAL_USER:$REAL_USER" "$DOT_SSH"
        chmod 700 "$DOT_SSH"
        sudo -u "$REAL_USER" ssh-keygen -t "$KEY_TYPE" -f "$KEY_FILE" -N ""
        USR_SSHKEY=$(cat "$KEY_FILE.pub")
        echo "Generated $KEY_TYPE key at $KEY_FILE"
    else
        echo "Proceeding without SSH key (password-only access)."
    fi
fi

# Hash password using openssl
USR_PASSWRD_HASHED=$(openssl passwd -6 "$USR_PASSWRD")

if [ -z "$USR_PASSWRD_HASHED" ]; then
    echo "ERROR: Failed to hash password with openssl."
    exit 1
fi

# VM identity
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))
VM_IP="192.168.8.$IP_NUM"
VM_DISK="$VM_STORE/$VM_NAME.qcow2"
SEED_ISO="$VM_STORE/$VM_NAME-seed.iso"

echo ""
echo "VM name  : $VM_NAME"
echo "MAC addr : $MAC_ADDR"
echo "IP       : $VM_IP"
echo "Disk     : $VM_DISK"
echo ""

# cloud-init files
TMPDIR_CI=$(mktemp -d)
trap 'rm -rf "$TMPDIR_CI"' EXIT

cat > "$TMPDIR_CI/meta-data" <<EOF
instance-id: $(uuidgen)
local-hostname: $VM_NAME
EOF

# SSH key block — only if provided
SSH_KEY_BLOCK=""
if [ -n "$USR_SSHKEY" ]; then
    SSH_KEY_BLOCK="    ssh_authorized_keys:
      - $USR_SSHKEY"
fi

# Rocky/RHEL cloud-init datasource config
WRITE_FILES_BLOCK=""
if [ "$OS_FAMILY" = "rhel" ]; then
    WRITE_FILES_BLOCK='write_files:
  - path: /etc/cloud/cloud.cfg.d/99-datasource.cfg
    content: |
      datasource_list: [NoCloud, None]
    owner: root:root
    permissions: "0644"'
fi

# cloud-init user-data
cat > "$TMPDIR_CI/user-data" <<EOF
#cloud-config
users:
  - name: $USR_NAME
    shell: /bin/bash
    lock_passwd: false
    passwd: "$USR_PASSWRD_HASHED"
    sudo: ALL=(ALL) NOPASSWD:ALL
$SSH_KEY_BLOCK
chpasswd:
  expire: false
ssh_pwauth: true
$WRITE_FILES_BLOCK
growpart:
  mode: auto
  devices: ['/']
resize_rootfs: true
EOF

# network-config (cloud-init v1 for RHEL, v2 for others)
if [ "$OS_FAMILY" = "rhel" ]; then
cat > "$TMPDIR_CI/network-config" <<EOF
version: 1
config:
  - type: physical
    name: eth0
    mac_address: "$MAC_ADDR"
    subnets:
      - type: static
        address: $VM_IP
        netmask: 255.255.255.0
        gateway: 192.168.8.1
        dns_nameservers:
          - 8.8.8.8
          - 1.1.1.1
EOF
else
cat > "$TMPDIR_CI/network-config" <<EOF
version: 2
ethernets:
  id0:
    match:
      macaddress: "$MAC_ADDR"
    dhcp4: false
    addresses:
      - $VM_IP/24
    routes:
      - to: default
        via: 192.168.8.1
    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]
EOF
fi

# Sanity check user-data
if ! grep -q "^#cloud-config" "$TMPDIR_CI/user-data"; then
    echo "ERROR: user-data generation failed."
    exit 1
fi

# Debug output
echo "── user-data ──────────────────────────"
cat "$TMPDIR_CI/user-data"
echo "───────────────────────────────────────"
echo ""

# Disk overlay — lives in VM_STORE, not beside the base image
BACKING_FMT=$(qemu-img info "$VM_IMG" | awk '/file format/ {print $3}')
qemu-img create -f qcow2 -F "$BACKING_FMT" -b "$(realpath "$VM_IMG")" "$VM_DISK" "$VM_DISK_SIZE"
echo "Created overlay disk: $VM_DISK with size $VM_DISK_SIZE"

# Seed ISO
cloud-localds -v "$SEED_ISO" "$TMPDIR_CI/user-data" "$TMPDIR_CI/meta-data" -N "$TMPDIR_CI/network-config"

if command -v isoinfo &>/dev/null; then
    isoinfo -i "$SEED_ISO" -l 2>/dev/null | grep -Ei "user|meta|network" || true
fi

# virt-install
echo ""
echo "Starting VM..."
# --graphics none --osinfo "$OSINFO" --boot hd,cdrom,menu=off \
# --graphics spice --video virtio --console pty,target_type=serial --osinfo rocky9 --boot uefi --cpu host-passthrough --machine q35

# RHEL-based distros often require UEFI boot and CPU passthrough for best compatibility/performance
if [ "$OS_FAMILY" = "rhel" ]; then
 BOOT_OPTS="--boot uefi --cpu host-passthrough"
else
 BOOT_OPTS="--boot hd,cdrom,menu=off"
fi

virt-install --name "$VM_NAME" --description "$OSINFO $VM_IP" \
    --memory "$VM_MEMORY" --vcpus "$VM_CPU" \
    --disk path="$VM_DISK",device=disk,bus=virtio,cache=none --disk path="$SEED_ISO",device=cdrom,bus=sata,readonly=on \
    --network bridge=br0,mac="$MAC_ADDR",model=virtio \
    --graphics none --osinfo "$OSINFO" \
    $BOOT_OPTS \
    --import \
    --noautoconsole

# Wait for VM running state (120 attempts with 2s sleep = 4 minutes max)
echo "Waiting for VM to start..."
for i in $(seq 1 120); do
    STATE=$(virsh domstate "$VM_NAME" 2>/dev/null | tr -d '\r')
    if [[ "$STATE" == "running" ]]; then
        echo "VM is running."
        break
    fi
    echo "Current state: $STATE"
    sleep 2
done
if [[ "$(virsh domstate "$VM_NAME" 2>/dev/null | tr -d '\r')" != "running" ]]; then
    echo "ERROR: VM did not reach running state."
    virsh dominfo "$VM_NAME" || true
    exit 1
fi

# Poll SSH until cloud-init finishes (10 min max)
echo "Waiting for SSH at $VM_IP (up to 10 minutes)..."
SSH_OPTS="-o StrictHostKeyChecking=no -o ConnectTimeout=5 -o BatchMode=yes -o LogLevel=ERROR"
SSH_KEY_OPT=""
[ -n "$KEY_FILE" ] && SSH_KEY_OPT="-i $KEY_FILE"

READY=0
for i in $(seq 1 120); do
    # shellcheck disable=SC2086
    STATUS=$(ssh $SSH_OPTS $SSH_KEY_OPT "$USR_NAME@$VM_IP" \
        "cloud-init status 2>/dev/null || echo 'no-cloud-init'" 2>/dev/null || echo "unreachable")
    case "$STATUS" in
        *"status: done"*)
            echo "Cloud-init finished successfully."
            READY=1; break ;;
        *"status: error"*)
            echo "WARNING: Cloud-init reported an error."
            echo "  Check: sudo cat /var/log/cloud-init-output.log"
            READY=1; break ;;
        *"no-cloud-init"*)
            echo "VM reachable (no cloud-init)."
            READY=1; break ;;
    esac
    echo "  attempt $i/120 — $STATUS"
    sleep 5
done

if [ "$READY" -eq 0 ]; then
    echo "WARNING: VM not reachable after 10 minutes."
    echo "It may still be booting. Try: ssh $USR_NAME@$VM_IP"
fi

# Eject and delete seed ISO and unset sensitive variables
CDROM_DEV=$(virsh domblklist "$VM_NAME" 2>/dev/null | awk '/seed\.iso/ {print $1}')
if [ -n "$CDROM_DEV" ]; then
    virsh change-media "$VM_NAME" "$CDROM_DEV" --eject --config 2>/dev/null || true
    rm -f "$SEED_ISO"
    unset USR_PASSWORD USR_SSHKEY
    echo "Seed ISO ejected and removed."
fi

# Done
echo ""
echo "════════════════════════════════════════"
echo " VM Ready"
echo "════════════════════════════════════════"
echo " Name   : $VM_NAME"
echo " IP     : $VM_IP"
echo " User   : $USR_NAME"
echo " Memory : ${VM_MEMORY}MB"
echo " vCPUs  : $VM_CPU"
echo " Disk   : $VM_DISK"
if [ -n "$KEY_FILE" ]; then
    echo " SSH    : ssh -i $KEY_FILE $USR_NAME@$VM_IP"
else
    echo " SSH    : ssh $USR_NAME@$VM_IP"
fi
echo "════════════════════════════════════════"
echo ""

CREATE_NEW_VM_SCRIPT
```


## other 

```bash
# transfering imgages to another server
scp -r root@192.168.8.70:/var/lib/libvirt/images/vms/ .
# or if rsync installed on both server and client
rsync -avz --sparse -e ssh root@192.168.8.70:/var/lib/libvirt/images/vms/ ./vms/

# allow kvm server 192.168.8.70 to route traffic using the default bridged nat
sudo iptables -I FORWARD -o virbr0 -d 192.168.122.0/24 -j ACCEPT

# add route to client to connect to kvm's VMs
sudo ip route add 192.168.122.0/24 via 192.168.8.70

#port forwarding examples 192.168.8.70 kvm server
# For SSH (Forwarding Host Port 2222 -> VM Port 22)
# To connect: ssh user@192.168.8.70 -p 2222
sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.122.50:22
sudo iptables -A FORWARD -p tcp -d 192.168.122.50 --dport 22 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

# For PostgreSQL (Forwarding Host Port 5432 -> VM Port 5432)
# To connect: 192.168.8.70:5432
sudo iptables -t nat -A PREROUTING -p tcp --dport 5432 -j DNAT --to-destination 192.168.122.50:5432
sudo iptables -A FORWARD -p tcp -d 192.168.122.50 --dport 5432 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```