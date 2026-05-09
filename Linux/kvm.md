


Standard stable libvirt stack:
```bash
sudo apt install ovmf qemu-system-x86 qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager cloud-image-utils libguestfs-tools osinfo-db libosinfo-bin -y
```

add your user to libvirt:
```bash
sudo usermod -aG libvirt $USER
newgrp libvirt
```

verify:
```bash
virsh list --all
```

--------------------

Bridged networking on Ubuntu + libvirt gives your VMs real LAN IPs

Check interface first:
```bash
ip link
```
Look for something like:
enp3s0
eth0
ens33



NetworkManager method


2. Create bridge br0
```bash
sudo nmcli connection add type bridge ifname br0 con-name br0
```

3. Assign IP method (DHCP recommended for homelab)
```bash
sudo nmcli connection modify br0 ipv4.method auto
sudo nmcli connection modify br0 ipv6.method auto
```

4. Attach your physical NIC to the bridge

disable auto IP on NIC:
```bash
sudo nmcli connection modify enp4s0 ipv4.method disabled
sudo nmcli connection modify enp4s0 ipv6.method ignore
```

enslave it to bridge:
```bash
sudo nmcli connection add type bridge-slave ifname enp4s0 master br0
```
Bring everything up & Verify:
```bash
sudo nmcli connection up br0
sudo nmcli connection up bridge-slave-enp4s0
```

```bash
ip a show br0
```

<br>

Connect libvirt to the bridge
1. open virt-manager
When creating VM:
NIC → “Bridge device”
Source: br0

2. edit default network (virsh / XML)
```bash
sudo virsh net-edit default
```

create VM NIC like:
```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

check connection
```bash
nmcli connection show
```

<br>

get a cloud image:
```bash
aria2c -x 16 -s 16 https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img
```
```bash
aria2c -x 16 -s 16 https://cloud-images.ubuntu.com/releases/resolute/release-20260421/ubuntu-26.04-server-cloudimg-amd64.img
```

create qcow2
```bash
qemu-img create -f qcow2 -F qcow2 -b ubuntu-26.04-server-cloudimg-amd64.img ubuntu-26.04-server.qcow2
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

## Create an isolated VM network

Use libvirt NAT:

```bash
virsh net-start default
```

or define custom isolated network:

```xml
<network>
  <name>isolated</name>
  <bridge name='virbr10'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
```

<br>

## Create a SHELL script 

grant file execution permissions
```bash
sudo chmod +x create-vm.sh
```
to run script
```bash
sudo ./create-vm.sh
```

```bash
cat <<'CREATE_NEW_VM_SCRIPT' > create-vm.sh
#!/bin/bash

# Find all .img in current directory
SEARCH_DIRS=("." "/var/lib/libvirt/images/")

mapfile -t IMG_FILES < <(
    for dir in "${SEARCH_DIRS[@]}"; do
        [ -d "$dir" ] && find "$dir" -maxdepth 2 -type f -name "*.img"
    done | sort -u
)

if [ ${#IMG_FILES[@]} -eq 0 ]; then
    echo "No base images found in current directory."
    exit 1
else
    echo "Available base images:"
    for i in "${!IMG_FILES[@]}"; do
        echo "$((i+1))) ${IMG_FILES[$i]}"
    done
    read -p "Select image [1-${#IMG_FILES[@]}]: " IMG_CHOICE
    if ! [[ "$IMG_CHOICE" =~ ^[0-9]+$ ]] || (( IMG_CHOICE < 1 || IMG_CHOICE > ${#IMG_FILES[@]} )); then
        echo "Invalid selection."
        exit 1
    fi
    VM_IMG="${IMG_FILES[$((IMG_CHOICE-1))]}"
fi

REAL_USER=${SUDO_USER:-$USER}
ACTUAL_HOME=$(getent passwd "$REAL_USER" | cut -d: -f6)
DOT_SSH="$ACTUAL_HOME/.ssh"

echo "Running as: $(whoami) | Original user: ${SUDO_USER:-$USER} | Home: $ACTUAL_HOME"
read -p "Enter Username [default: sysadmin]: " USR_NAME
USR_NAME=${USR_NAME:-sysadmin} 
read -sp "Enter User Password: " USR_PASSWRD
echo "" 
read -p "Enter IP Last Octet (192.168.8.X): " IP_NUM
read -p "Paste SSH Key (or leave blank to search/generate): " USR_SSHKEY

if [ -z "$USR_SSHKEY" ]; then
    if [ -f "$DOT_SSH/id_ed25519.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_ed25519.pub")
    elif [ -f "$DOT_SSH/id_rsa.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_rsa.pub")
    fi
fi

if [ -z "$USR_SSHKEY" ]; then
    echo "No SSH key found."
    read -p "Would you like to generate one now? (y/n): " GEN_CONFIRM
    if [[ "$GEN_CONFIRM" =~ ^[Yy]$ ]]; then
        echo "Choose key type:"
        echo "1) ed25519 (Modern, recommended)"
        echo "2) rsa (Classic, widely compatible)"
        read -p "Selection [1-2]: " KEY_CHOICE
        case $KEY_CHOICE in
            2)
                KEY_TYPE="rsa"
                KEY_FILE="$DOT_SSH/id_rsa"
                ;;
            *)
                KEY_TYPE="ed25519"
                KEY_FILE="$DOT_SSH/id_ed25519"
                ;;
        esac
        mkdir -p "$DOT_SSH"
        chown "$REAL_USER:$REAL_USER" "$DOT_SSH"
        chmod 700 "$DOT_SSH"
        sudo -u "$REAL_USER" ssh-keygen -t "$KEY_TYPE" -f "$KEY_FILE" -N ""
        USR_SSHKEY=$(cat "$KEY_FILE.pub")
        echo "Successfully generated $KEY_TYPE key."
    else
        echo "Proceeding without an SSH key..."
    fi
fi

VM_NAME="vm-$(uuidgen | tr -d '-' | cut -c1-8)"
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))
USR_PASSWRD_HASHED=$(mkpasswd -m yescrypt "$USR_PASSWRD")
VM_IMG="ubuntu-26.04-server-cloudimg-amd64.img"

cat <<EOF > meta-data
instance-id: $(uuidgen || echo i-abcdefg)
local-hostname: $VM_NAME
EOF

cat <<EOF > user-data
#cloud-config
users:
  - name: $USR_NAME
    shell: /bin/bash
    lock_passwd: false
    passwd: "$USR_PASSWRD_HASHED"
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    groups: users, admin
    ssh_authorized_keys:
      - $USR_SSHKEY
growpart:
  mode: auto
  devices: ['/']
resize_rootfs: true      
EOF

cat <<EOF > network-config
version: 2
renderer: networkd
ethernets:
  eth0: # Changed from net0 to eth0 for standard naming
    match:
      macaddress: $MAC_ADDR
    set-name: eth0
    dhcp4: no
    addresses:
      - 192.168.8.$IP_NUM/24
    routes:
      - to: default
        via: 192.168.8.1
    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]
EOF


qemu-img create -f qcow2 -F qcow2 -b "$VM_IMG" "$VM_NAME.qcow2"

cloud-localds -v "$VM_NAME-seed.iso" user-data meta-data -N network-config

rm -rf user-data meta-data network-config

virt-install \
  --name "$VM_NAME" \
  --memory 2048 \
  --vcpus 2 \
  --disk path="./$VM_NAME.qcow2",device=disk,bus=virtio \
  --disk path="./$VM_NAME-seed.iso",device=cdrom \
  --network bridge=br0,mac="$MAC_ADDR" \
  --graphics none \
  --osinfo generic \
  --import
CREATE_NEW_VM_SCRIPT
```
<br>

## virsh commands to control vms

```bash
# active/running
virsh list
# all VMs
virsh list --all
```


Start	
```bash
virsh start <vm_name>
```

Shut down normally
```bash
virsh shutdown <vm_name>
```

Force Stop (Power Off)
```bash
virsh destroy	<vm_name>
```

Suspend	
```bash
virsh suspend <vm_name>
```

Resume	
```bash
virsh resume <vm_name>
```
start automatically on host boot up
```bash
virsh autostart <vm_name>
# to disable
virsh autostart --disable <vm_name>
```

Delete

```bash
# Locate the disk
virsh domblklist <vm_name>
```
```bash
# Deletes registration/config
virsh undefine <vm_name>
```
```bash
# Delete the physical file
rm /var/lib/libvirt/images/vm_name.qcow2
```

or 

```bash
virsh undefine <vm_name> --remove-all-storage
# will remove all boot drives and disks for that vm
```




grant file execution permissions
```bash
sudo chmod +x create-vm.sh
```
to run script
```bash
sudo ./create-vm.sh


```bash

cat <<'CREATE_NEW_VM_SCRIPT' > create-vm.sh
#!/bin/bash

SEARCH_DIRS=("/var/lib/libvirt/images/")

mapfile -t IMG_FILES < <(
    for dir in "${SEARCH_DIRS[@]}"; do
        [ -d "$dir" ] && find "$dir" -maxdepth 2 -type f \( -name "*.img" -o -name "*.qcow2" \)
    done | sort -u
)

if [ ${#IMG_FILES[@]} -eq 0 ]; then
    echo "No base images found in /var/lib/libvirt/images/."
    read -p "Would you like to download a fresh image? (y/n): " DL_CONFIRM
    [[ ! "$DL_CONFIRM" =~ ^[Yy]$ ]] && { echo "No image selected. Exiting."; exit 1; }
else
    echo "Available base images:"
    for i in "${!IMG_FILES[@]}"; do
        echo "$((i+1))) ${IMG_FILES[$i]}"
    done
    echo "$((${#IMG_FILES[@]}+1))) Download a fresh image"
    read -p "Select [1-$((${#IMG_FILES[@]}+1))]: " IMG_CHOICE

    if ! [[ "$IMG_CHOICE" =~ ^[0-9]+$ ]] || (( IMG_CHOICE < 1 || IMG_CHOICE > ${#IMG_FILES[@]}+1 )); then
        echo "Invalid selection."
        exit 1
    fi

    if (( IMG_CHOICE != ${#IMG_FILES[@]}+1 )); then
        VM_IMG="${IMG_FILES[$((IMG_CHOICE-1))]}"
    fi
fi

  if [ -z "${VM_IMG:-}" ]; then
      echo ""
      echo "Select image to download:"
      echo "1) Ubuntu 24.04 LTS"
      echo "2) Ubuntu 26.04 LTS"
      echo "3) Debian 12 (Bookworm)"
      echo "4) Rocky 9 (RHEL-compatible)"
      echo "5) AlmaLinux 9 (RHEL-compatible)"
      echo "6) Fedora 44"
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
      VM_IMG="/var/lib/libvirt/images/$(basename "$DL_URL")"
      echo "Downloading to /var/lib/libvirt/images/..."
      
      if command -v aria2c &>/dev/null; then
      echo "using aria2 for faster download.."
      aria2c -x 8 -s 8 -o "$(basename "$DL_URL")" -d "/var/lib/libvirt/images/" "$DL_URL" || { echo "Download failed."; exit 1; }
  else
      read -p "aria2c not found (faster downloads). Install it? (y/n): " INSTALL_ARIA
      if [[ "$INSTALL_ARIA" =~ ^[Yy]$ ]]; then
          apt-get install -y aria2 || { echo "Install failed, falling back to wget."; wget -O "$VM_IMG" "$DL_URL" || { echo "Download failed."; exit 1; }; }
          aria2c -x 8 -s 8 -o "$(basename "$DL_URL")" -d "/var/lib/libvirt/images/" "$DL_URL" || { echo "Download failed."; exit 1; }
      else
          wget -O "$VM_IMG" "$DL_URL" || { echo "Download failed."; exit 1; }
      fi
  fi
fi

REAL_USER=${SUDO_USER:-$USER}
ACTUAL_HOME=$(getent passwd "$REAL_USER" | cut -d: -f6)
DOT_SSH="$ACTUAL_HOME/.ssh"

echo "Running as: $(whoami) | Original user: ${SUDO_USER:-$USER} | Home: $ACTUAL_HOME"
read -p "Enter Memory [default: 2048]: " VM_MEMORY
VM_MEMORY=${VM_MEMORY:-2048}
read -p "Enter VCPUs [default: 2]: " VM_CPU
VM_CPU=${VM_CPU:-2}
read -p "Enter Username [default: sysadmin]: " USR_NAME
USR_NAME=${USR_NAME:-sysadmin} 
read -sp "Enter User Password: " USR_PASSWRD
echo "" 
read -p "Enter IP Last Octet (192.168.8.X): " IP_NUM
read -p "Paste SSH Key (or leave blank to search/generate): " USR_SSHKEY

if [ -z "$USR_SSHKEY" ]; then
    if [ -f "$DOT_SSH/id_ed25519.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_ed25519.pub")
        KEY_FILE="$DOT_SSH/id_ed25519"
    elif [ -f "$DOT_SSH/id_rsa.pub" ]; then
        USR_SSHKEY=$(cat "$DOT_SSH/id_rsa.pub")
        KEY_FILE="$DOT_SSH/id_rsa"
    fi
fi

if [ -z "$USR_SSHKEY" ]; then
    echo "No SSH key found."
    read -p "Would you like to generate one now? (y/n): " GEN_CONFIRM
    if [[ "$GEN_CONFIRM" =~ ^[Yy]$ ]]; then
        echo "Choose key type:"
        echo "1) ed25519 (Modern, recommended)"
        echo "2) rsa (Classic, widely compatible)"
        read -p "Selection [1-2]: " KEY_CHOICE
        case $KEY_CHOICE in
            2)
                KEY_TYPE="rsa"
                KEY_FILE="$DOT_SSH/id_rsa"
                ;;
            *)
                KEY_TYPE="ed25519"
                KEY_FILE="$DOT_SSH/id_ed25519"
                ;;
        esac
        mkdir -p "$DOT_SSH"
        chown "$REAL_USER:$REAL_USER" "$DOT_SSH"
        chmod 700 "$DOT_SSH"
        sudo -u "$REAL_USER" ssh-keygen -t "$KEY_TYPE" -f "$KEY_FILE" -N ""
        USR_SSHKEY=$(cat "$KEY_FILE.pub")
        echo "Successfully generated $KEY_TYPE key."
    else
        echo "Proceeding without an SSH key..."
    fi
fi

VM_NAME="vm-$(uuidgen | tr -d '-' | cut -c1-8)"
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))
USR_PASSWRD_HASHED=$(mkpasswd -m yescrypt "$USR_PASSWRD")

cat <<EOF > meta-data
instance-id: $(uuidgen || echo i-abcdefg)
local-hostname: $VM_NAME
EOF

cat <<EOF > user-data
#cloud-config
users:
  - name: $USR_NAME
    shell: /bin/bash
    lock_passwd: false
    passwd: "$USR_PASSWRD_HASHED"
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    groups: users, admin
    ssh_authorized_keys:
      - $USR_SSHKEY
growpart:
  mode: auto
  devices: ['/']
resize_rootfs: true      
EOF

cat <<EOF > network-config
version: 2
renderer: networkd
ethernets:
  eth0: # Changed from net0 to eth0 for standard naming
    match:
      macaddress: $MAC_ADDR
    set-name: eth0
    dhcp4: no
    addresses:
      - 192.168.8.$IP_NUM/24
    routes:
      - to: default
        via: 192.168.8.1
    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]
EOF


BACKING_FMT=$(qemu-img info "$VM_IMG" | awk '/file format/ {print $3}')
qemu-img create -f qcow2 -F "$BACKING_FMT" -b "$(realpath "$VM_IMG")" "$VM_NAME.qcow2"

cloud-localds -v "$VM_NAME-seed.iso" user-data meta-data -N network-config

rm -rf user-data meta-data network-config

unset USR_PASSWORD USR_SSHKEY

virt-install \
  --name "$VM_NAME" \
  --memory $VM_MEMORY \
  --vcpus $VM_CPU \
  --disk path="./$VM_NAME.qcow2",device=disk,bus=virtio \
  --disk path="./$VM_NAME-seed.iso",device=cdrom \
  --network bridge=br0,mac="$MAC_ADDR" \
  --graphics none \
  --osinfo generic \
  --import \
  --noautoconsole

echo "Waiting for VM to start..."
while ! virsh dominfo "$VM_NAME" 2>/dev/null | grep -q "State:.*running"; do
    sleep 2
done
echo "VM is running, waiting for cloud-init to complete..."

VM_IP="192.168.8.$IP_NUM"

while true; do
    STATUS=$(ssh -o StrictHostKeyChecking=no \
                 -o ConnectTimeout=5 \
                 -o BatchMode=yes \
                 -i "$KEY_FILE" \
                 "$USR_NAME@$VM_IP" \
                 "cloud-init status 2>/dev/null" 2>/dev/null)
    case "$STATUS" in
        *"status: done"*)
            echo "Cloud-init finished successfully."
            break
            ;;
        *"status: error"*)
            echo "Cloud-init reported an error — check the VM."
            break
            ;;
    esac
    sleep 5
done

CDROM_DEV=$(virsh domblklist "$VM_NAME" | awk '/seed\.iso/ {print $1}')
if [ -n "$CDROM_DEV" ]; then
    virsh change-media "$VM_NAME" "$CDROM_DEV" --eject --config
    rm -f "$VM_NAME-seed.iso"
    echo "Seed ISO detached and removed."
fi

echo ""
echo "VM ready: $VM_NAME"
echo "IP: $VM_IP"
echo "SSH: ssh $USR_NAME@$VM_IP"

CREATE_NEW_VM_SCRIPT


```



create a

```bash
#!/ prenatal/bash

if [ "$EUID" -ne 0 ]; then 
  echo "Please run as root (use sudo)"
  exit
fi

# Check for apt (Debian/Ubuntu)
if command -v apt &> /dev/null; then
    echo "Detected Debian-based system. Using apt..."
    sudo apt update
    sudo apt install -y git curl

# Check for dnf (Fedora/RHEL)
elif command -v dnf &> /dev/null; then
    echo "Detected Red Hat-based system. Using dnf..."
    sudo dnf install -y git curl

else
    echo "Error: Neither apt nor dnf found. This script may not support your distro."
    exit 1
fi
```

