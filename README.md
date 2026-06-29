#  DevOps Notes

> The brain dump of a hands-on homelabber who refuses to learn anything twice.

This repo is my personal **command-first cheat-sheet vault** — the stuff I actually run, not the
theory I skim. If a topic lives here, it means I reduced it to copy-pasteable commands with the
flags decoded inline, the gotchas flagged with `>`, and (where it matters) a full numbered runbook
so a 2 a.m. me can rebuild it half-asleep. KVM VMs, k8s clusters, HA PostgreSQL, NFS failover,
Docker, Terraform, Ansible — if I've broken it in the homelab, it's documented here.

**House style:** lead with the runnable command → explain flags as trailing `# comments` →
use `<placeholders>` so everything is a reusable template → call out the trap *and* the workaround.
Reference goes in flat cheat-sheets (TOC on top); multi-step procedures get strict Step 1→N runbooks
with an architecture diagram and port table first. A topic isn't "done" until it's a self-running
systemd service or a runbook I can hand to someone else.

---

##  Repo Map

```
devops/
├── README.md                 # you are here
├── git.md                    # Git + gh CLI: workflow, conventional commits, recovery
├── GCP.md                    # gcloud: compute, GKE, VPC, HA VPN / Cloud Router / BGP
│
├── Linux/                    # the heart of the homelab
│   ├── Linux.md              # base toolbox: packages, networking, SSH, firewall, systemd
│   ├── Docker.md             # images, compose, build/push, inspect & debug
│   ├── Docker-Swarm.md       # multi-host orchestration: services, Raft quorum, stacks
│   ├── k8s.md                # kubeadm cluster: control plane, CNI, MetalLB, NGINX ingress
│   ├── k8s-CI-CD.md          # CI → CD pipeline notes for Kubernetes
│   ├── KVM.md                # virsh + qemu-img + virt-customize, automated VM provisioning
│   ├── Proxmox.md            # Proxmox VE 9.2: qm VMs, pct LXC, ZFS, Ceph, clustering
│   ├── PostgreSQL.md         # client/server install + day-to-day psql
│   ├── HA-PostgreSQL.md      # active/standby HA Postgres 18 runbook (architecture + steps)
│   ├── HA-NFS-Server.md      # DRBD + Pacemaker/Corosync active/passive NFS cluster
│   ├── Storage.md            # NFS, mounts, and storage plumbing
│   ├── Terraform.md          # IaC: install, core workflow, project layout, state
│   ├── Ansible.md            # agentless config mgmt: inventory, ad-hoc, playbooks
│   ├── RHEL.md               # hardening & compliance: SELinux, OpenSCAP, FIPS
│   ├── caddy.md              # Caddy install + Caddyfile reverse proxy
│   └── vscode.md             # editor tweaks (multi-command, settings.json)
│
├── SQL/                      # SQL command cheat-sheets (PostgreSQL-flavored)
│   ├── SELECT.md             # reading data: WHERE, JOINs, GROUP BY, CTEs, window fns
│   ├── CREATE.md             # DDL: tables, types, constraints, indexes, views, ALTER
│   ├── UPDATE.md             # writing data: INSERT, UPDATE, DELETE, UPSERT, transactions
│   └── pgloader.md           # migrating MSSQL / MySQL / SQLite → PostgreSQL
│
├── Windows Server/
│   └── Hyper-v.md            # Hyper-V VM lifecycle + PowerShell provisioning scripts
│
└── GCP/                      # (reserved — gcloud working files)
```

---

##  Quick jump

| ? | > |
|---|---|
| VM'ing | [KVM](Linux/KVM.md) · [Proxmox](Linux/Proxmox.md) · [Hyper-V](Windows%20Server/Hyper-v.md) |
| Clustering | [k8s](Linux/k8s.md) · [Docker Swarm](Linux/Docker-Swarm.md) |
| Get High | [HA PostgreSQL](Linux/HA-PostgreSQL.md) · [HA NFS](Linux/HA-NFS-Server.md) |
|  SQL syntax | [SELECT](SQL/SELECT.md) · [CREATE](SQL/CREATE.md) · [UPDATE](SQL/UPDATE.md) |
| Migrate to Postgres | [pgloader](SQL/pgloader.md) |
| Codify infra | [Terraform](Linux/Terraform.md) · [Ansible](Linux/Ansible.md) |
| Harden a box | [RHEL](Linux/RHEL.md) · [Linux › Firewall](Linux/Linux.md) |

---

*These are working notes — opinionated, homelab-tested, and occasionally ahead of (or behind) the
official docs. Trust the command, verify the gotcha.*
