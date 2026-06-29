# GCP (gcloud)

- [Config & auth](#config--auth)
- [Compute instances](#compute-instances)
- [File transfer](#file-transfer)
- [GKE / Kubernetes](#gke--kubernetes)
- [Networking / VPC](#networking--vpc)
- [HA VPN / Cloud Router / BGP](#ha-vpn--cloud-router--bgp)
- [Network load balancer (L4)](#network-load-balancer-l4)
- [HTTP load balancer (L7)](#http-load-balancer-l7)
- [Cloud SQL](#cloud-sql)
- [Logging](#logging)
- [Cloud Spanner](#cloud-spanner)
- [Terraform](#terraform)

## Config & auth
```bash

env | grep DEVSHELL     # print injected DEVSHELL_ environment variables

echo $GOOGLE_CLOUD_PROJECT
echo $DEVSHELL_PROJECT_ID
echo $WEB_HOST
echo $CLOUD_SHELL
# Always set to true. 
# Excellent for adding guards to scripts e.g. 
if [ "$CLOUD_SHELL" != "true" ]; then exit 1; fi).

gcloud auth list
gcloud config list project
gcloud config list --format 'value(core.project)'
gcloud components list

# set defaults
gcloud config set compute/region europe-southwest1
gcloud config set compute/zone   europe-southwest1-a
gcloud config set project ${DEVSHELL_PROJECT_ID}

# get defaults
gcloud config get-value compute/region
gcloud config get-value compute/zone
gcloud config get-value project

# stash them in env vars for reuse
export REGION=$(gcloud config get-value compute/region)
export ZONE=$(gcloud config get-value compute/zone)
export PROJECT_ID=$(gcloud config get-value project)

echo -e "PROJECT ID: $PROJECT_ID\nZONE: $ZONE"

gcloud compute project-info describe --project $(gcloud config get-value project)
gcloud compute regions list
```

## Compute instances
```bash
gcloud compute instances list
# --filter="status=RUNNING"          # or --filter="name=instance-0342"
# --zones="europe-southwest1-a"
# --sort-by=$ZONE
# --format="json"  |  --format="yaml"
# --format="table(name, zone, status)"
# --format="table(name, id, networkInterfaces[0].accessConfigs[0].natIP:label=EXTERNAL_IP, status)"
# --format="value(networkInterfaces[0].networkIP)"
# --format="table(name, disks[].deviceName:flatten())"
gcloud compute instances list --filter="name=('<instance-name>')"
```

```bash
# create VM (full)
gcloud compute instances create privatenet-us-vm --zone=europe-southwest1-b \
    --machine-type=e2-micro --subnet=privatesubnet-us \
    --image-family=debian-11 --image-project=debian-cloud \
    --boot-disk-size=10GB --boot-disk-type=pd-standard \
    --boot-disk-device-name=privatenet-us-vm

# create VM (minimum)
gcloud compute instances create <vm-name> --zone=europe-southwest1-b \
    --machine-type=f1-micro --subnet=<subnet-name>

gcloud compute instances create gcelab2 --machine-type e2-medium --zone=$ZONE
```

```bash
# connect over SSH
gcloud compute ssh <instance-name> --zone $ZONE
# shell.cloud.google.com?authuser=user@email.com

# make standard OpenSSH (~/.ssh/config) recognize GCE instances
gcloud compute config-ssh

# move VM within a region
gcloud compute instances move <vm-name> --zone $ZONE --destination-zone <new-zone>

# tags drive firewall targeting
gcloud compute instances add-tags gcelab2 --tags http-server,https-server
```

```bash
# startup-script bakes config at boot (web server example)
gcloud compute instances create www1 \
    --zone=$ZONE \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

```bash
# delete instance
gcloud compute instances delete <instance-name> --zone europe-southwest1-b
```

## File transfer
```bash
# copy a dir up to a VM (Cloud Shell)
gcloud compute scp -r ./odoo-custom-addons <instance-name>:/tmp --zone=europe-west3-c

# rsync straight to the instance's external SSH
rsync -avzP ./folder-name/ \
    instance-0321.europe-southwest1-b.projectid-01:/home/username/folder-name/

# rsync tunneled through `gcloud compute ssh` (no public IP / IAP-friendly)
rsync -avzP -e "gcloud compute ssh --zone=europe-southwest1-b" \
    ./folder-name/ instance-0321:/home/username/folder-name/
```

## GKE / Kubernetes
```bash
Cluster='kube-test'
Zone='europe-southwest1-b'
Region='europe-southwest1'

# zonal cluster spanning multiple node locations
gcloud container clusters create $Cluster --zone $Zone \
    --node-locations europe-southwest1-a,europe-southwest1-b,europe-southwest1-c

gcloud container clusters create --machine-type=e2-medium --zone=$Zone $Cluster
gcloud container clusters create <cluster-name> --create-subnetwork name=my-subnet

gcloud container clusters create griffin-dev --num-nodes=2 --zone=griffin-dev \
    --subnet=griffin-dev-wp --machine-type=e2-standard-4

# autopilot (Google-managed nodes), regional
gcloud container clusters create-auto $Cluster --region $Region

# auth: writes kubeconfig context
gcloud container clusters get-credentials $Cluster
gcloud container clusters delete $Cluster
```

```bash
# deploy
kubectl create deployment --image nginx nginx-1
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0

# expose
kubectl expose pod $nginx_pod --port 80 --type LoadBalancer
kubectl expose deployment hello-server --type=LoadBalancer --port 8083

# shell into a container
kubectl exec -it new-nginx -- /bin/bash

# switch context between clusters
kubectl config use-context gke_${DEVSHELL_PROJECT_ID}_us-east4_autopilot-cluster-1

# push a file into a pod
kubectl cp ~/test.html $nginx_pod:/usr/share/nginx/html/test.html
```

## Networking / VPC
```bash
# FQDN format:  [hostname].[zone].c.[projectid].internal
#   example:    myserver.me-central1-a.c.pid-133434.internal

# Cloud IP ranges by region:  https://www.gstatic.com/ipranges/cloud.json
# geofeed (Maxmind/neustar/IP2Location): https://www.gstatic.com/ipranges/cloud_geofeed

# ingress = inbound, egress = outbound

gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK
```

```bash
# custom-mode network (no auto subnets)
gcloud compute networks create vpc-demo --subnet-mode custom

# subnets
gcloud compute networks subnets create vpc-demo-subnet1 --network vpc-demo --range 10.1.1.0/24 --region "us-east4"
gcloud compute networks subnets create vpc-demo-subnet2 --network vpc-demo --range 10.2.1.0/24 --region "us-west1"
```

```bash
# firewall: list
gcloud compute firewall-rules list
gcloud compute firewall-rules list --filter="network='default'"
gcloud compute firewall-rules list --filter="NETWORK:'default' AND ALLOW:'icmp'"
gcloud compute firewall-rules list --filter=ALLOW:'80'

# firewall: create
gcloud compute firewall-rules create vpc-demo-allow-custom --network vpc-demo \
    --allow tcp:0-65535,udp:0-65535,icmp --source-ranges 10.0.0.0/8

gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 \
    --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

# allow SSH + ICMP from anywhere
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp --network vpc-demo --allow tcp:22,icmp

gcloud compute firewall-rules create vpc-demo-allow-subnets-from-on-prem \
    --network vpc-demo --allow tcp,udp,icmp --source-ranges 192.168.1.0/24

gcloud compute firewall-rules create on-prem-allow-subnets-from-vpc-demo \
    --network on-prem --allow tcp,udp,icmp --source-ranges 10.1.1.0/24,10.2.1.0/24
```

```bash
# DELETE (order matters: rules -> subnets -> network)
gcloud compute firewall-rules delete vpc-demo-allow-custom
gcloud compute networks subnets delete vpc-demo-subnet2 --region us-west1
gcloud compute networks delete vpc-demo
```

## HA VPN / Cloud Router / BGP
```bash
# HA VPN gateway
gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network vpc-demo --region us-east4
gcloud compute vpn-gateways describe vpc-demo-vpn-gw1 --region us-east4

# Cloud Router (ASN per side)
gcloud compute routers create vpc-demo-router1 --region us-east4 --network vpc-demo --asn 65001
```

```bash
# VPN tunnels (interface 0 and 1 for HA)
gcloud compute vpn-tunnels create vpc-demo-tunnel0 \
    --peer-gcp-gateway on-prem-vpn-gw1 --region us-east4 --ike-version 2 \
    --shared-secret [SHARED_SECRET] --router vpc-demo-router1 \
    --vpn-gateway vpc-demo-vpn-gw1 --interface 0

gcloud compute vpn-tunnels create vpc-demo-tunnel1 \
    --peer-gcp-gateway on-prem-vpn-gw1 --region us-east4 --ike-version 2 \
    --shared-secret [SHARED_SECRET] --router vpc-demo-router1 \
    --vpn-gateway vpc-demo-vpn-gw1 --interface 1

gcloud compute vpn-tunnels create on-prem-tunnel0 \
    --peer-gcp-gateway vpc-demo-vpn-gw1 --region us-east4 --ike-version 2 \
    --shared-secret [SHARED_SECRET] --router on-prem-router1 \
    --vpn-gateway on-prem-vpn-gw1 --interface 0

gcloud compute vpn-tunnels list
```

```bash
# BGP: add a router interface + peer per tunnel
gcloud compute routers add-interface vpc-demo-router1 --interface-name if-tunnel0-to-on-prem \
    --ip-address 169.254.0.1 --mask-length 30 --vpn-tunnel vpc-demo-tunnel0 --region us-east4
gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel0 \
    --interface if-tunnel0-to-on-prem --peer-ip-address 169.254.0.2 --peer-asn 65002 --region us-east4

gcloud compute routers add-interface vpc-demo-router1 --interface-name if-tunnel1-to-on-prem \
    --ip-address 169.254.1.1 --mask-length 30 --vpn-tunnel vpc-demo-tunnel1 --region us-east4
gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel1 \
    --interface if-tunnel1-to-on-prem --peer-ip-address 169.254.1.2 --peer-asn 65002 --region us-east4

gcloud compute routers add-interface on-prem-router1 --interface-name if-tunnel0-to-vpc-demo \
    --ip-address 169.254.0.2 --mask-length 30 --vpn-tunnel on-prem-tunnel0 --region us-east4
gcloud compute routers add-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel0 \
    --interface if-tunnel0-to-vpc-demo --peer-ip-address 169.254.0.1 --peer-asn 65001 --region us-east4

gcloud compute routers add-interface on-prem-router1 --interface-name if-tunnel1-to-vpc-demo \
    --ip-address 169.254.1.2 --mask-length 30 --vpn-tunnel on-prem-tunnel1 --region us-east4
gcloud compute routers add-bgp-peer on-prem-router1 --peer-name bgp-vpc-demo-tunnel1 \
    --interface if-tunnel1-to-vpc-demo --peer-ip-address 169.254.1.1 --peer-asn 65001 --region us-east4

# remove a peer
gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name bgp-on-prem-tunnel0 --region us-east4
```

## Network load balancer (L4)
```bash
# pool of backend VMs (www1/2/3 created with the startup-script above), then:
gcloud compute firewall-rules create www-firewall-network-lb --target-tags network-lb-tag --allow tcp:80

gcloud compute addresses create network-lb-ip-1 --region $Region   # reserve regional IP

gcloud compute http-health-checks create basic-check

gcloud compute target-pools create www-pool --region $Region --http-health-check basic-check
gcloud compute target-pools add-instances www-pool --instances www1,www2,www3

gcloud compute forwarding-rules create www-rule \
    --region $Region --ports 80 --address network-lb-ip-1 --target-pool www-pool
```

```bash
# test the LB until a backend answers
while [ -z "$RESULT" ]; do
  echo "Waiting for Load Balancer";
  sleep 5;
  RESULT=$(curl -m1 -s 34.36.92.252:80 | grep Apache);
done

# stress test
ab -n 500000 -c 1000 http://34.36.92.252:80/
```

## HTTP load balancer (L7)
```bash
# 1) instance template (startup-script installs Apache, serves its hostname)
gcloud compute instance-templates create lb-backend-template \
   --region=$Region --network=default --subnet=default \
   --tags=allow-health-check --machine-type=e2-medium \
   --image-family=debian-11 --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | tee /var/www/html/index.html
     systemctl restart apache2'

# 2) managed instance group from the template
gcloud compute instance-groups managed create lb-backend-group \
    --template=lb-backend-template --size=2 --zone=$Zone

# 3) firewall for Google health-check ranges
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default --action=allow --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=allow-health-check --rules=tcp:80

# 4) global static IP
gcloud compute addresses create lb-ipv4-1 --ip-version=IPV4 --global

# 5) health check
gcloud compute health-checks create http http-basic-check --port 80

# 6) backend service + attach the MIG
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP --port-name=http --health-checks=http-basic-check --global
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group --instance-group-zone=$Zone --global

# 7) URL map -> HTTP proxy -> forwarding rule
gcloud compute url-maps create web-map-http --default-service web-backend-service
gcloud compute target-http-proxies create http-lb-proxy --url-map web-map-http
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1 --global --target-http-proxy=http-lb-proxy --ports=80
```

```bash
# named port so the backend service knows where HTTP lives on the MIG
gcloud compute instance-groups set-named-ports lb-mig --named-ports http:80 --zone $Zone

# read back the reserved global IP
gcloud compute addresses describe lb-ipv4-1 --format="get(address)" --global

# HTTPS variant just swaps the URL map / proxy
gcloud compute url-maps create web-map-https --default-service web-backend-service
```

## Cloud SQL
```bash
gcloud sql instances create odoo-postgresql \
    --database-version=POSTGRES_14 --cpu=2 --memory=7680MB \
    --region=me-central2 --storage-size=20GB --storage-type=SSD

gcloud sql databases create odoo-db --instance=odoo-postgresql

gcloud sql users set-password postgres --instance=odoo-postgresql --password=mystrongpassword
```

## Logging
```bash
gcloud logging logs list
gcloud logging logs list --filter="compute"
gcloud logging read "resource.type=gce_instance" --limit 5
gcloud logging read "resource.type=gce_instance AND labels.instance_name='<instance-name>'" --limit 5
```

## Cloud Spanner
```sql
-- GQL: graph query (also-bought + full-text SEARCH)
GRAPH RetailGraph
MATCH (:Product {name: "TrailMaster All-Terrain Boots"})<-[:Purchase]-(:User)-[:Purchase]->(alsoBought:Product)
WHERE SEARCH(product.descriptionToken, "waterproof hiking boots")
RETURN DISTINCT alsoBought.name;
```
```sql
-- full-text search over a tokenized column
SELECT DocID, Content FROM Documents
WHERE SEARCH(Content_Tokens, "waterproof hiking boots", enhance_query=>true);
```

## Terraform
```bash
terraform init      # download provider plugins, init backend
terraform plan      # preview what apply will change
terraform apply     # create/update real infrastructure
terraform destroy   # tear it down
terraform fmt       # auto-format to canonical style
```

```hcl
# layout
# instances/
#   main.tf  providers.tf  variables.tf  outputs.tf
# terraform.tfstate   (created/updated automatically; can live remotely)
```

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

variable "project_id"     { description = "Your GCP project ID" }
variable "region"         { description = "The region where resources will be created" }
variable "num_webservers" {
  description = "Number of web server instances to create"
  default     = 2
}

resource "google_compute_network" "vpc_network" {
  name                = "my-vpc-network"
  auto_create_subnets = false
}

# multiple subnets with for_each
resource "google_compute_subnetwork" "subnet" {
  for_each      = toset(range(var.num_webservers))
  name          = "my-subnet-${each.key}"
  region        = google_compute_network.vpc_network.region
  ip_cidr_range = "10.1.${each.key}.0/16"
  network       = google_compute_network.vpc_network.self_link
}

resource "google_compute_firewall" "allow_http" {
  name    = "allow-http-traffic"
  network = google_compute_network.vpc_network.self_link
  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
  source_ranges = ["0.0.0.0/0"]
}

# web servers with count
resource "google_compute_instance" "web_server" {
  count        = var.num_webservers
  name         = format("my-webserver-%d", count.index)
  machine_type = "e2-micro"
  zone         = "${google_compute_region.region.zones[0]}"

  boot_disk {
    initialize_params { image = "debian-cloud/debian-11" }
  }
  network_interface {
    network    = google_compute_network.vpc_network.self_link
    subnetwork = google_compute_subnetwork.subnet[count.index].self_link
    access_config { nat_ip = "AUTO_ONLY" }
  }
  depends_on = [google_compute_firewall.allow_http]   # firewall first
}

resource "google_compute_region" "region" {
  name = var.region
}
```

A full HTTPS LB in Terraform wires together: an SSL cert
(`google_compute_ssl_certificate`) with your PEM cert + key; a backend service
(`google_compute_backend_service`, protocol HTTPS) bound to a health check
(`google_compute_health_check`); an instance group exposing port 443; a URL map
(`google_compute_url_map`) routing to the backend; and a forwarding rule
(`google_compute_forwarding_rule`) on 443 referencing the cert and an `ssl_policy`.

```hcl
resource "google_compute_ssl_certificate" "webserver_cert" {
  name              = "my-webserver-cert"
  description       = "SSL certificate for web server"
  certificate       = base64decode(var.certificate_pem)
  private_key       = base64decode(var.private_key_pem)
  certificate_chain = ""   # add your chain if applicable
}

resource "google_compute_backend_service" "webserver_backend" {
  name                  = "my-webserver-backend"
  protocol              = "HTTPS"
  load_balancing_scheme = "INTERNAL"
  health_checks         = [google_compute_health_check.webserver_check.self_link]
  backend_config {
    port          = 443   # where the web server listens for HTTPS
    instance_pool = google_compute_instance_group.webserver_pool.self_link
  }
}

resource "google_compute_health_check" "webserver_check" {
  name                  = "my-webserver-healthcheck"
  type                  = "HTTP"
  port                  = 443
  request_path          = "/"
  check_interval        = "60"
  timeout_after_seconds = "5"
}

resource "google_compute_instance_group" "webserver_pool" {
  name        = "my-webserver-pool"
  zone        = "${google_compute_region.region.zones[0]}"
  named_ports = [{ name = "https", port = 443 }]
  # ... instance template config
}

resource "google_compute_url_map" "webserver_map" {
  name            = "my-webserver-map"
  default_service = google_compute_backend_service.webserver_backend.self_link
  path_matcher {
    name            = "webserver-path-matcher"
    default_service = google_compute_backend_service.webserver_backend.self_link
    path_rule { paths = ["/*"] }
  }
}

resource "google_compute_forwarding_rule" "webserver_https_rule" {
  name            = "my-webserver-https-rule"
  port_range      = 443
  target          = google_compute_url_map.webserver_map.self_link
  region          = google_compute_region.region.name
  backend_service = google_compute_backend_service.webserver_backend.self_link
  ssl_certificate = google_compute_ssl_certificate.webserver_cert.self_link
  ssl_policy      = "TLS_V1_2"
}
```

```hcl
# minimal single-instance example
terraform {
  required_providers {
    google = { source = "hashicorp/google" }
  }
}

provider "google" {
  project = "qwiklabs-gcp-03-2ace8308b124"
  region  = "us-west1"
  zone    = "us-west1-c"
}

resource "google_compute_instance" "terraform" {
  name         = "terraform"
  machine_type = "e2-medium"
  tags         = ["web", "dev"]
  boot_disk {
    initialize_params { image = "debian-cloud/debian-11" }
  }
  network_interface {
    network = "default"
    access_config {}
  }
  allow_stopping_for_update = true
}
```

```hcl
# count vs for_each
resource "google_compute_instance" "dev_vm" {
  count = 3
  name  = "dev_vm${count.index + 1}"
  zone  = "us-central-1"
}

resource "google_compute_instance" "dev_vm" {
  for_each = toset(["us-central-1", "asia-east1-b", "europe-west4-a"])
  name     = "dev-${each.value}"
  zone     = each.value
}

# input validation
variable "mybucket_storageclass" {
  type        = string
  description = "set the storage class of the bucket"
  validation {
    condition     = contains(["STANDARD", "MULTI_REGIONAL", "REGIONAL"], var.mybucket_storageclass)
    error_message = "Allowed storage classes are STANDARD, MULTI_REGIONAL and REGIONAL."
  }
}
```

```hcl
# remote state in a GCS bucket: locking + IAM + encryption (vs local: no lock,
# no shared access, sensitive data exposed in the local state file)
provider "google" {
  project = "Project ID"
  region  = "Region"
}
resource "google_storage_bucket" "test-bucket-for-state" {
  name                        = "Project ID"
  location                    = "US"   # EU for Europe
  uniform_bucket_level_access = true
}
terraform {
  backend "gcs" {
    bucket = "Project ID"
    prefix = "terraform/state"
  }
}
```

> **Output value** arguments: `value`, `description`, `sensitive`.
