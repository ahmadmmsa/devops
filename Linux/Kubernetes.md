# Kubernetes workflow

# Table of Contents
- [Installation](##Installation)
- [Creating a Local Registry](#creating-a-local-registry)
- [DockerHub](#dockerhub-workflow)
- [worker join cmd](#control-plane-join-command)
- [init control plane](#initialize-control-plane)



<br>

 
## Kubectl 


### Apply

```bash
#Apply all manifests in folder
kubectl apply -f k8s/
# Apply all in subfolders
kubectl apply -f k8s/ -R
# Single apply
kubectl apply -f k8s/frontend-deployment.yaml
# Dry Run Check for syntax errors
kubectl apply -f . --dry-run=client 
# diffrence use before apply
kubectl diff -f . 
```

### Get

```bash
kubectl get all
kubectl get pods
kubectl get pods --show-labels
# detailed Listing (IPs and Nodes)
kubectl get pods -o wide
# in YAML format
kubectl get pods -o yaml
# in JSON format
kubectl get pods -o json
```

```bash
# scans every single namespace in the cluster
kubectl get pods -A
# Keeps the session open and waits for changes
kubectl get pods -w
# Filters the list based on labels.
kubectl get pods -l app=backend
# Label Columns. -L
kubectl get pods -L app,env
```

```bash
kubectl get pods -A -l app=my-web-app -o wide -w
#  Look in all namespaces (-A) 
#  With the label app=my-web-app (-l)
#  Show IPs and Nodes (-o wide)
#  Keep the screen updated as things change (-w)
```

### Describe
```bash
# check if Pod is failing, crashing, or won't start
kubectl describe pod <pod-name>
kubectl describe pod <pod-name> | grep -i Events -A 10
```

### Label
```bash
# Add label with role worker
kubectl label node kube-2 node-role.kubernetes.io/worker=worker
# Change label name
kubectl label pods -l app=old-name app=my-project --overwrite
```

### Delete


```bash
# force Kubernetes to recreate the Pods.
kubectl delete pods --all
kubectl delete pods --all --force --grace-period=0
# keep ConfigMaps, Secrets, Ingresses, (PVCs)
kubectl delete all --all
 # stop and remove ALL (nuke’em)
kubectl delete deployments --all

# Find & delete with label app=my-project
kubectl delete -l app=my-project.
#Wipe Entire Namespace
kubectl delete ns <namespace_name>
```




<br>




## Installation

- Disable Swap
- add kernel modules
- configuration to start on bootup
- apply the settings &  filter the bridge traffic

example structure:
- 192.168.8.111 kube-1
- 192.168.8.112 kube-2

```bash
nano /etc/hosts
```

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
free -h
```

```bash
modprobe overlay
modprobe br_netfilter
```

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

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

- Install container runtime
- Generate Default Config
- Enable SystemdCgroup

```bash
apt update
apt install -y containerd
```

```bash
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

```bash
systemctl restart containerd
systemctl enable containerd
```

<br>

Install Kubernetes packages
- check version [kubernetes.io/releases](https://kubernetes.io/releases/)
```bash
apt update
apt install -y apt-transport-https ca-certificates curl gpg
```

```bash
mkdir -p /etc/apt/keyrings
```


```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
apt update
apt install -y kubelet kubeadm kubectl
```
 ```bash
apt-mark hold kubelet kubeadm kubectl
```
 
<br><br>

## Initialize control plane

```bash
kubeadm init --apiserver-advertise-address=192.168.8.100 --pod-network-cidr=192.168.0.0/16
```
 or custom range use Tigera-operator
```bash
kubeadm init --apiserver-advertise-address=192.168.8.100 --pod-network-cidr=10.10.0.0/16
```
> other flags --node-name k8s-control --kubernetes-version 1.30

 
Configure kubectl access

```bash
mkdir -p $HOME/.kube
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

## control plane Join command
```bash
kubeadm token create --print-join-command
```
>Copy returned command and run on worker node


<br>

## NFS

remove PersistentVolumeClaims (PVCs)
```bash
kubectl delete pvc --all
```

PresistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: odoo-filestore-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany        # important for multiple Odoo pods
  nfs:
    server: 192.168.8.10
    path: /srv/odoo/filestore
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-filestore-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
securityContext:
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001      
```



<br><br>

## Installing a load balancer (MetalLB)
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

<br>


## NGINX Ingress Controller
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
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'
```
MetalLB assigns an external IP behaves similar to cloud




configure the insecure registry on the k8s node 


```bash
# Create the directory for your registry
sudo mkdir -p /etc/containerd/certs.d/192.168.8.100:5000

# Create the hosts.toml file for it
sudo tee /etc/containerd/certs.d/192.168.8.100:5000/hosts.toml <<EOF
server = "http://192.168.8.100:5000"

[host."http://192.168.8.100:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
```

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```


<br>

## Dockerhub workflow

from the control plane
```bash
docker login
docker build -t johndoe/backend:latest .
docker push johndoe/backend:latest
```
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

# Creating a Local Registry 

Kubernetes pulls container images from a registry

- 192.168.8.25 → Build server + private Docker registry
- 192.168.8.100 → Kubernetes control plane
- 192.168.8.101 → Kubernetes worker

 workflow:

- Build image on 192.168.8.25
- Push image to registry on 192.168.8.25:5000
- Configure Kubernetes nodes to trust that registry
- Reference the image in your Deployment YAML
- Kubernetes pulls the image automatically

<br>

## Building and deploying

 on host > goto project folder build > tag & push
```bash
docker compose up -d --build
```
```bash
docker tag app-name:latest 192.168.8.25:5000/app-name:latest
```
```bash
docker push 192.168.8.25:5000/app-name:latest
```
<br>

 on k8s control plane > create YAML & apply

example: deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-name
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-name
  template:
    metadata:
      labels:
        app: app-name
    spec:
      containers:
        - name: backend
          image: 192.168.8.25:5000/app-name:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
```

```bash
kubectl apply -f deployment.yaml
```
<br>

## Setting up registry & Configure k8s Nodes

creating Local Registry
```bash
sudo apt install -y docker.io docker-compose-v2 docker-buildx
```

```bash
ufw allow 5000/tcp
```

```bash
docker run -d --name registry -p 5000:5000 --restart unless-stopped registry:2
```


<br>


Run on All k8s:


```bash
sudo mkdir -p /etc/containerd/certs.d/192.168.8.25:5000
sudo tee /etc/containerd/certs.d/192.168.8.25:5000/hosts.toml > /dev/null <<EOF
server = "http://192.168.8.25:5000"
[host."http://192.168.8.25:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
sudo systemctl restart containerd
```


