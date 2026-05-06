


Standard stable libvirt stack:
```bash
sudo apt install qemu-system-x86 qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager cloud-image-utils libguestfs-tools -y
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

1. Install bridge utilities
```bash
sudo apt install bridge-utils -y
```

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
cat <<'INSTALL_SCRIPT' > create-vm.sh
#!/bin/bash

echo "Running as: $(whoami) | Original user: ${SUDO_USER:-$USER} | Home: $ACTUAL_HOME"
read -p "Enter Username [default: sysadmin]: " USR_NAME
USR_NAME=${USR_NAME:-sysadmin} 
read -sp "Enter User Password: " USR_PASSWRD
echo "" 
read -p "Enter IP Last Octet (192.168.8.X): " IP_NUM
read -p "Paste SSH Key (or leave blank to search/generate): " USR_SSHKEY

REAL_USER=${SUDO_USER:-$USER}
ACTUAL_HOME=$(getent passwd "$REAL_USER" | cut -d: -f6)
DOT_SSH="$ACTUAL_HOME/.ssh"

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
INSTALL_SCRIPT
```
