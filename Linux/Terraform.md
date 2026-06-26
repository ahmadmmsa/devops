
# Terraform

Declarative infrastructure-as-code. You describe the desired end state in `.tf` files; Terraform diffs it against real infra (tracked in *state*) and makes only the changes needed.

- [Install](#install)
- [Core workflow](#core-workflow)
- [Project layout](#project-layout)
- [HCL basics](#hcl-basics)
- [Variables, outputs, locals](#variables-outputs-locals)
- [Data sources](#data-sources)
- [State](#state)
- [Workspaces](#workspaces)
- [Modules](#modules)
- [Provider: libvirt (KVM)](#provider-libvirt-kvm)
- [Provider: Docker](#provider-docker)
- [Handy commands](#handy-commands)
- [Gotchas](#gotchas)

## Install
```bash
# HashiCorp apt repo (Ubuntu)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

terraform version                    # confirm install
terraform -install-autocomplete      # shell tab-completion
```

## Core workflow
The whole loop is four commands: **init → plan → apply → destroy**.
```bash
terraform init        # download providers + modules, set up .terraform/ and backend
terraform plan        # dry-run: show what WOULD change, change nothing
terraform apply       # apply changes (prompts yes/no, re-runs plan first)
terraform destroy     # tear down everything in state (prompts)

# common flags
terraform plan  -out=tf.plan          # save an exact plan to apply later
terraform apply tf.plan               # apply that saved plan (no re-prompt)
terraform apply -auto-approve         # skip the yes/no prompt (CI)
terraform apply -target=libvirt_domain.vm   # act on ONE resource only (escape hatch, not routine)
terraform apply -var="vm_count=3"           # override a variable inline
terraform apply -var-file="prod.tfvars"     # load vars from a file
```
> `plan` is read-only and safe to run anytime. Always read the plan's `+ create / ~ update / - destroy` summary before typing `yes` — a `~` that becomes `-/+` means **replace** (destroy then recreate).

## Project layout
```
infra/
├── main.tf            # providers + resources (entry point by convention, not magic)
├── variables.tf       # input variable declarations
├── outputs.tf         # values to print / expose to other modules
├── terraform.tfvars   # actual variable values (gitignore if it holds secrets)
├── versions.tf        # required_version + required_providers
└── .terraform/        # downloaded providers/modules — gitignore this
```
Terraform loads **all `*.tf` files** in the directory and concatenates them — filenames are organizational only.
```gitignore
# .gitignore essentials
.terraform/
*.tfstate
*.tfstate.*
*.tfvars        # if they contain secrets
tf.plan
```

## HCL basics
```hcl
# versions.tf — pin Terraform + provider versions
terraform {
  required_version = ">= 1.6"
  required_providers {
    libvirt = {
      source  = "dmacvicar/libvirt"
      version = "~> 0.7"        # ~> 0.7 = >=0.7.0, <0.8.0 (pessimistic constraint)
    }
  }
}

# provider block — configures HOW to talk to the platform
provider "libvirt" {
  uri = "qemu:///system"
}

# resource block — a thing to create: resource "<type>" "<local name>"
resource "libvirt_volume" "disk" {
  name = "vm-disk.qcow2"
  pool = "default"
  size = 10 * 1024 * 1024 * 1024   # 10 GiB
}
```
> Reference other resources' attributes as `<type>.<name>.<attr>` (e.g. `libvirt_volume.disk.id`). These references build the **dependency graph** automatically — Terraform orders creation/destruction from them, so you rarely need `depends_on`.

## Variables, outputs, locals
```hcl
# variables.tf — inputs
variable "vm_count" {
  type    = number
  default = 1
}
variable "ssh_key" {
  type        = string
  description = "Public key injected into the VM"
  sensitive   = true       # hides the value from CLI output
}

# locals — computed/reused values, not overridable from outside
locals {
  base_name = "homelab"
  full_name = "${local.base_name}-vm"
}

# outputs.tf — printed after apply, readable by parent modules
output "vm_ip" {
  value = libvirt_domain.vm.network_interface[0].addresses[0]
}
```
```bash
terraform output            # show all outputs
terraform output -raw vm_ip # raw value (no quotes) — pipe into scripts
```
Precedence (low → high): `default` < `terraform.tfvars` < `*.auto.tfvars` < `-var-file` < `-var` < `TF_VAR_<name>` env.
```bash
export TF_VAR_ssh_key="$(cat ~/.ssh/id_ed25519.pub)"   # env-var way to set a variable
```

## Data sources
Read existing infra/info you didn't create (lookup, not manage).
```hcl
data "libvirt_network" "default" {
  name = "default"
}
# use it: data.libvirt_network.default.id
```

## State
State maps your config to real-world resources. Treat it as sensitive — it can contain secrets in plaintext.
```bash
terraform state list                       # list tracked resources
terraform state show libvirt_domain.vm     # inspect one resource's recorded attrs
terraform state rm libvirt_domain.vm       # stop tracking (does NOT destroy real infra)
terraform state mv <old.addr> <new.addr>   # rename/move without recreate (e.g. after refactor)
terraform refresh                          # reconcile state with reality (now part of plan/apply)
terraform import libvirt_domain.vm <id>    # bring an existing resource UNDER terraform's management
```
```hcl
# Remote backend — share state + lock it so two people can't apply at once
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "homelab/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "tf-locks"   # state locking
  }
}
```
> Local `terraform.tfstate` is fine solo, but **never commit it** (secrets + merge conflicts). For any team/CI use a remote backend with locking.

## Workspaces
Multiple independent states from one config (e.g. dev vs prod) — lightweight, share the same code.
```bash
terraform workspace list
terraform workspace new prod
terraform workspace select dev
terraform workspace show
# reference inside config: terraform.workspace
```
> Workspaces are NOT a strong isolation boundary for prod. For real prod/dev separation, prefer separate dirs/backends. Good for ephemeral/test stacks.

## Modules
Reusable, parameterized groups of resources.
```hcl
module "vm" {
  source   = "./modules/kvm-vm"   # local path, registry, or git URL
  vm_count = 3
  ssh_key  = var.ssh_key
}
# consume its outputs: module.vm.vm_ips
```
```bash
terraform get -update    # pull/refresh module sources
terraform init           # also installs modules
```

## Provider: libvirt (KVM)
Spin up a VM on your local KVM host — fits the homelab.
```hcl
provider "libvirt" {
  uri = "qemu:///system"
}

# cloud image as the base volume
resource "libvirt_volume" "ubuntu" {
  name   = "ubuntu-base.qcow2"
  pool   = "default"
  source = "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
  format = "qcow2"
}

# per-VM disk backed by the base image
resource "libvirt_volume" "vm_disk" {
  name           = "vm-${count.index}.qcow2"
  base_volume_id = libvirt_volume.ubuntu.id
  pool           = "default"
  size           = 20 * 1024 * 1024 * 1024
  count          = var.vm_count          # count -> creates N copies, indexed [0..N-1]
}

# cloud-init to set user/ssh key on first boot
resource "libvirt_cloudinit_disk" "init" {
  name      = "cloudinit.iso"
  user_data = <<-EOF
    #cloud-config
    users:
      - name: ahmad
        sudo: ALL=(ALL) NOPASSWD:ALL
        ssh_authorized_keys:
          - ${var.ssh_key}
  EOF
}

resource "libvirt_domain" "vm" {
  name      = "${local.full_name}-${count.index}"
  count     = var.vm_count
  memory    = 2048
  vcpu      = 2
  cloudinit = libvirt_cloudinit_disk.init.id

  disk { volume_id = libvirt_volume.vm_disk[count.index].id }

  network_interface {
    network_name   = "default"
    wait_for_lease = true     # block until DHCP gives an IP, so outputs have it
  }
  console {
    type        = "pty"
    target_type = "serial"
    target_port = "0"
  }
}

output "vm_ips" {
  value = libvirt_domain.vm[*].network_interface[0].addresses[0]   # [*] = splat over all VMs
}
```
> Needs the `dmacvicar/libvirt` provider + libvirt/qemu on the host. Use `qemu:///system` (not `:///session`) for system-wide VMs; your user must be in the `libvirt` group.

## Provider: Docker
```hcl
terraform {
  required_providers {
    docker = { source = "kreuzwerker/docker", version = "~> 3.0" }
  }
}
provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = true
}
resource "docker_container" "web" {
  name  = "web"
  image = docker_image.nginx.image_id
  ports {
    internal = 80
    external = 8080
  }
}
```

## Handy commands
```bash
terraform fmt -recursive       # auto-format all .tf files
terraform validate             # check syntax + internal consistency (no API calls)
terraform graph | dot -Tpng > graph.png   # render the dependency graph
terraform show                 # human-readable current state
terraform console              # interactive REPL to test expressions/interpolations
terraform taint <addr>         # (legacy) mark for recreation next apply
terraform apply -replace=<addr># modern: force-recreate one resource
terraform providers            # list providers this config requires
terraform force-unlock <id>    # release a stuck state lock (use with care)
```

## Gotchas
- **`apply` shows `-/+` (replace)** — a change forced destroy-and-recreate, not in-place edit. For a VM that means data loss; check *why* (often an immutable field like name/image) before saying yes.
- **Never edit `terraform.tfstate` by hand** — use `terraform state mv/rm/import`. Hand-edits corrupt the file and desync from reality.
- **`state rm` ≠ destroy** — it only forgets the resource; the real VM/container keeps running (orphaned). Use `destroy` to actually delete.
- **Don't commit state or `.tfvars` with secrets** — state stores values in plaintext; use a remote backend + gitignore.
- **Drift** — someone changed infra outside Terraform; next `plan` shows unexpected diffs. Re-plan to reconcile, then decide: import the change or let apply revert it.
- **`-target` is an escape hatch, not a workflow** — partial applies skip the dependency graph and can leave state inconsistent. Use it to break a deadlock, then run a full apply.
- **Pin provider versions** (`~>`) — an unpinned provider can auto-upgrade and break your config on the next `init`.
- **State locking** — if an apply crashes you may get "state locked"; only `force-unlock` once you're sure no other apply is running.
