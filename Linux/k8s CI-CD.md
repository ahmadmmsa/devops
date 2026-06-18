 
 
 
               [ CI: Automated Build & Test ]
                             │
                             ▼
               [ CD: The Final 'D' Phase ]
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
    [ Continuous Delivery ]       [ Continuous Deployment ]
    Stops before production       Goes straight to production
    Requires a manual click       Fully automated release





third-party platforms:
- GitHub Actions
- GitLab CI/CD

GitHub Actions
- Free tiers
- Simple setup
- Massive ecosystem
- GitHub dominates source hosting

hosted SaaS platforms integrated with Git
- GitHub Actions
- GitLab CI/CD
- Bitbucket Pipelines
- Azure DevOps Pipelines
- AWS CodePipeline
- Google Cloud Build

Dedicated Platforms popular before GitHub Actions
- CircleCI
- Travis CI
- Semaphore CI
- Buildkite
- TeamCity
- Bamboo

Deployment Platforms (PaaS) CI and CD together.
- Vercel
- Netlify
- Render
- Fly.io
- Railway
- Heroku

Self-Hosted
- Jenkins
- Jenkins X now JayeX
- Woodpecker CI
- Drone CI
- Concourse CI
- GoCD

Image Update Automation
These watch registries and update manifests.
- Argo CD Image Updater
- Flux Image Automation Controller
- Keel
- Renovate
- Dependabot









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

## Creating a Local Registry 

configure local registry example 192.168.8.100

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
