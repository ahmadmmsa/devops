# Kubernetes workflow

# Table of Contents
- [Installation](##Installation)
- [Kubectl commands](#kubectl-commands)
-- [Apply](#apply)
-- [Get](#get)
-- [Describe](#describe)
-- [Label](#label)
-- [Delete](#delete)
- [Docker](#docker)


## Installation

### System Prerequisites

Disable Swap

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
free -h
```

add kernel modules
```bash
modprobe overlay
modprobe br_netfilter
```


configuration file to start on bootup
```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
apply the settings & start filtering the bridge traffic
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
```bash
sysctl --system
```
check if active
```bash
lsmod | grep -e overlay -e br_netfilter
```

 <br>

Install container runtime
```bash
apt update
apt install -y containerd
```

Generate Default Config
```bash
mkdir -p /etc/containerd
```

```bash
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```


<br>

Edit and Enable SystemdCgroup ⚠️
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```
```bash
systemctl restart containerd
systemctl enable containerd
```
check
```bash
systemctl status containerd
```

<br>

Install Kubernetes packages

```bash
apt update
apt install -y apt-transport-https ca-certificates curl gpg
```

```bash
mkdir -p /etc/apt/keyrings
```

Installs the GPG (GNU Privacy Guard) key
check version [kubernetes.io/releases](https://kubernetes.io/releases/)
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Tell system where to find Kubernetes
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

Install Packages
```bash
apt update
apt install -y kubelet kubeadm kubectl
```
 
Prevent automatic update ⚠️
```bash
apt-mark hold kubelet kubeadm kubectl
```
 
<br>
 

Add hosts to all vms
```bash
nano /etc/hosts
```
```bash
192.168.8.111 kube-1
192.168.8.112 kube-2
```

 
<br><br>
 

### Initialize control plane
Only for Control Plane ⚠️

Initialize
```bash
kubeadm init --apiserver-advertise-address=192.168.8.111 --pod-network-cidr=192.168.0.0/16
```
 or custom range use Tigera-operator
```bash
kubeadm init --apiserver-advertise-address=192.168.8.111 --pod-network-cidr=10.10.0.0/16
```
optional flags: 
- --node-name k8s-control
- --kubernetes-version 1.30

<br>
 
Configure kubectl access

```bash
mkdir -p $HOME/.kube
```
```bash
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
 
<br>
 
Install CNI 

if pods range is 192.168.0.0/16
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

if custom pods range 10.10.0.0/16 

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
```
```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
```
```bash
nano custom-resources.yaml
```
change cidr: 192.168.0.0/16 to 10.10.0.0/16

```YAML
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

```
kubectl apply -f custom-resources.yaml
```

<br>

**control plane Join command**
```bash
kubeadm token create --print-join-command
```
>[!NOTE]
>Copy returned command and run on worker node


 
<br><br>
 
## Kubectl commands

use before apply
to see exactly what will change compared to what's currently running

```bash
kubectl diff -f . 
```

<br><br>

### Apply

Apply all manifests in folder
```bash
kubectl apply -f k8s/
```
Apply all in subfolders
```bash
kubectl apply -f k8s/ -R
```
Single apply
```bash
kubectl apply -f k8s/frontend-deployment.yaml
```
Dry Run Check for syntax errors
```bash
kubectl apply -f . --dry-run=client 
```
 
<br><br>

### Get

```bash
kubectl get all
```

```bash
kubectl get pods
```

```bash
kubectl get pods --show-labels
```

<br>

Detailed Listing (IPs and Nodes) --output
```bash
kubectl get pods -o wide
```
in YAML format
```bash
kubectl get pods -o yaml
```
in JSON format
```bash
kubectl get pods -o json
```

<br>

Scans every single namespace in the cluster. --all-namespaces 
```bash
kubectl get pods -A
```
Keeps the session open and waits for changes. --watch
```bash
kubectl get pods -w
```
Filters the list based on labels. --selector
```bash
kubectl get pods -l app=backend
```
Label Columns. -L

```bash
kubectl get pods -L app,env
```

<br>

**Combining Them**
```bash
kubectl get pods -A -l app=my-web-app -o wide -w
```
- Look in all namespaces (-A) 
- With the label app=my-web-app (-l)
- Show IPs and Nodes (-o wide)
- Keep the screen updated as things change (-w)

<br><br>

### Describe
check if Pod is failing, crashing, or won't start
```bash
kubectl describe pod <pod-name>
```

```bash
kubectl describe pod <pod-name> | grep -i Events -A 10
```

<br><br>

### Label
Add label with role worker
```bash
kubectl label node kube-2 node-role.kubernetes.io/worker=worker
```
Change label name
```bash
kubectl label pods -l app=old-name app=my-project --overwrite
```

<br><br>


### Delete

force Kubernetes to recreate the Pods.
```bash
kubectl delete pods --all
```

```bash
kubectl delete pods --all --force --grace-period=0
```

<br>


if you want to keep:
ConfigMaps, Secrets, Ingresses, Persistent Volume Claims (PVCs)
```bash
kubectl delete all --all
```

<br>

completely stop and remove your applications from the cluster (**nuke’em**)
```bash
kubectl delete deployments --all
```

<br>


Find every object that has the label app=my-project and remove it from the cluster.
```bash
kubectl delete -l app=my-project.
```

check before deleting
```bash
kubectl delete all -l app=my-project --dry-run=client
```

Wipe Entire Namespace
```bash
kubectl delete ns <namespace_name>
```

remove PersistentVolumeClaims (PVCs)
```bash
kubectl delete pvc --all
```


<br><br>


### Docker

```bash
docker build -t my-app:1.0 .
```


```bash
docker build --network host -t my-app/backend:latest .
```


```bash
docker images | grep abrahamic-frontend
```



Remove all images
```bash
docker rmi -f $(docker images -q)
```

remove all containers
```bash
docker rm $(docker ps -aq)
``` 







Start everything

```bash
docker compose up -d
```

Stop everything

```bash
docker compose down
```

Rebuild after code changes

```bash
docker compose up --build
```

View combined logs

```bash
docker compose logs -f
```

<br>

### Dockerhub workflow

from the control plane
```bash
docker login
docker build -t johndoe/backend:latest .
docker push johndoe/backend:latest
```

<br>

example:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-name
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app-name
  template:
    metadata:
      labels:
        app: my-app-name
    spec:
      containers:
      - name: my-app-name
        image: dockerhub-username/dockerhub-image:latest
        ports:
        - containerPort: 3001
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-name
spec:
  type: NodePort
  selector:
    app: my-app-name
  ports:
  - port: 3001
    targetPort: 3001

```

<br>

### Setting up a local Registry VM

on 192.168.8.113

```
apt update
apt install docker -y
```

```
ufw allow 5000/tcp
```

```
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

```
cd /home/user/myapp
docker build -t localhost:5000/my-react-app:v1 .

docker push localhost:5000/my-react-app:v1
```


Run these steps on BOTH 192.168.8.111 and 192.168.8.112:

```
sudo nano /etc/docker/daemon.json
```


```
{
  "insecure-registries" : ["192.168.8.113:5000"]
}
```


```
sudo systemctl restart docker
```





Kube-1 (Control Plane) pulls it

```
kubectl create deployment my-app --image=192.168.8.113:5000/my-react-app:v1
```

```
kubectl expose deployment frontend --port=80 --target-port=80 --type=NodePort
```

```
kubectl get svc frontend
```
http://192.168.8.111:3XXXX


<br><br>

### Installing a load balancer (MetalLB)
check version [github](https://github.com/metallb/metallb/releases)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

Create a ConfigMap with IP Range

```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: metallb-config
  namespace: metallb-system
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.8.240-192.168.8.250
```
apply the config:
```bash
kubectl apply -f metallb-config.yaml
```

```
kubectl get pods -n metallb-system
```

<br><br>

### NGINX Ingress Controller
check version [github](https://github.com/kubernetes/ingress-NGINX)

Bare-metal install command
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/baremetal/deploy.yaml
```

Cloud install command
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
```

<br>

After installing MetalLB:
- Patch the ingress service to LoadBalancer:

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec": {"type": "LoadBalancer"}}'
```
MetalLB assigns an external IP behaves similar to cloud


