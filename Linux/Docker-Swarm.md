# Docker Swarm

Built-in orchestrator for multi-host Docker. You declare a desired state (a **service** =
N replicas of an image), the **managers** run a Raft store and schedule **tasks** (containers)
onto **workers**. Lighter than k8s, ships with Docker.

```
                 ┌──────── Raft quorum ────────┐
   manager-1 ●───────── manager-2 ●─────────● manager-3      (odd # of managers)
      │ scheduler / API                          │
      └──────────── overlay network ─────────────┘
            │            │             │
        worker-1 ○   worker-2 ○    worker-3 ○                 (run the tasks)
```

| Port  | Proto   | Purpose                                  |
|-------|---------|------------------------------------------|
| 2377  | TCP     | cluster management (managers only)       |
| 7946  | TCP+UDP | node-to-node gossip / discovery          |
| 4789  | UDP     | overlay network data plane (VXLAN)       |

> **Gotcha:** open 7946 + 4789 between *all* nodes or overlay DNS/routing silently breaks.
> Run an **odd** number of managers (1/3/5). 3 managers tolerate 1 loss; lose quorum and the
> cluster is read-only — you can't deploy until it's back.

## Contents
- [Bootstrap the cluster](#bootstrap-the-cluster)
- [Nodes](#nodes)
- [Services (imperative)](#services-imperative)
- [Scale / rolling update / rollback](#scale--rolling-update--rollback)
- [Stacks (declarative — what you actually use)](#stacks-declarative--what-you-actually-use)
- [Secrets & configs](#secrets--configs)
- [Networks](#networks)
- [Debug & inspect](#debug--inspect)
- [Maintenance & teardown](#maintenance--teardown)
- [Full stack example](#full-stack-example)

---

## Bootstrap the cluster

```bash
# 1. on the first manager
docker swarm init --advertise-addr <manager-IP>   # pin the IP; multi-NIC hosts guess wrong
docker info | grep Swarm                            # verify > Swarm: active

# 2. print join commands to hand to other nodes
docker swarm join-token worker                      # copy the full line it prints
docker swarm join-token manager                     # for adding more managers

# 3. on each worker / manager-to-be (run what step 2 printed)
docker swarm join --token <token> <manager-IP>:2377

# rotate a leaked token (old tokens stop working)
docker swarm join-token --rotate worker
```

## Nodes

```bash
docker node ls                                  # * marks the node you're on; MANAGER STATUS = Leader/Reachable
docker node inspect <node> --pretty             # human-readable detail
docker node ps <node>                           # tasks running on that node

# drain a node before maintenance — reschedules its tasks elsewhere
docker node update --availability drain <node>
docker node update --availability active <node> # put it back in rotation
# pause = keep running tasks but schedule no new ones
docker node update --availability pause <node>

# labels for placement constraints (e.g. pin DB to a node with SSDs)
docker node update --label-add disk=ssd <node>
docker node update --label-add zone=a <node>

docker node promote <node>      # worker -> manager
docker node demote  <node>      # manager -> worker (do this BEFORE removing a manager)
docker node rm <node>           # remove (must be down or drained); --force if stuck
```

> **Gotcha:** never `docker node rm` a live manager — demote it first or you risk losing quorum.

## Services (imperative)

Use stacks for real deployments; these are for quick one-offs / debugging.

```bash
docker service create --name web --replicas 3 -p 8080:80 nginx:alpine
docker service ls                       # REPLICAS shows 3/3 when converged
docker service ps web                   # one row per task; CURRENT STATE + ERROR column
docker service ps web --no-trunc        # full error message when a task won't start
docker service logs -f web              # aggregated logs across all replicas
docker service inspect web --pretty

# global service = exactly one task per node (agents: node-exporter, log shippers)
docker service create --mode global --name node-exporter prom/node-exporter

# placement: pin where tasks land
docker service create --name db \
  --constraint 'node.labels.disk==ssd' \
  --constraint 'node.role==worker' \
  postgres:18

docker service rm web
```

## Scale / rolling update / rollback

```bash
docker service scale web=10                       # or: docker service update --replicas 10 web

# rolling image update — Swarm replaces tasks in batches, no downtime
docker service update --image nginx:1.27 web

# control the rollout (set these so a bad image can't take the whole service down)
docker service update \
  --update-parallelism 1 \      # update 1 task at a time
  --update-delay 10s \          # wait 10s between batches
  --update-order start-first \  # start new task before killing old (avoids capacity dip)
  --update-failure-action rollback \  # auto-undo if a task fails to start
  --image myapp:2.0 my-service

docker service rollback web        # manual revert to the previous spec

# change env / resources without a new image
docker service update --env-add LOG_LEVEL=debug web
docker service update --limit-cpu 1 --limit-memory 512M web
docker service update --force web  # redeploy/redistribute tasks with no spec change
```

> **Gotcha:** `start-first` needs a published-port that allows two tasks briefly, or spare
> capacity. With `stop-first` (default) you get a momentary replica dip during each batch.

## Stacks (declarative — what you actually use)

A **stack** = a compose file deployed across the swarm. This is the day-to-day workflow.

```bash
docker stack deploy -c docker-compose.yml myapp     # deploy/update the whole app
docker stack deploy -c docker-compose.yml --with-registry-auth myapp  # private registry pull
docker stack ls                                     # stacks + service count
docker stack services myapp                         # services in the stack
docker stack ps myapp                               # all tasks (add --no-trunc for errors)
docker stack rm myapp                               # tear the whole stack down
```

> **Gotcha:** `docker stack deploy` ignores `build:`, `depends_on` ordering, and `container_name`.
> Build & push images first (`docker build` + `docker push`); reference them by tag in the file.
> Use `deploy:` keys (replicas/placement/resources) which plain `docker compose` ignores.

## Secrets & configs

Secrets are encrypted in the Raft log and mounted at `/run/secrets/<name>` in the container —
never in env vars or image layers.

```bash
# create from a file or stdin
docker secret create pg_password ./pg_password.txt
printf 'S3cr3t!' | docker secret create pg_password -      # trailing - = read stdin
echo "$API_KEY" | docker secret create api_key -

docker secret ls
docker secret inspect pg_password          # metadata only — value is never shown

# attach to a service -> appears at /run/secrets/pg_password
docker service create --name db --secret pg_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg_password postgres:18

# non-sensitive config files (nginx.conf, etc.) — same idea, not encrypted
docker config create nginx_conf ./nginx.conf
docker service create --name web --config source=nginx_conf,target=/etc/nginx/nginx.conf nginx
```

> **Gotcha:** secrets/configs are **immutable**. To "change" one: create `pg_password_v2`,
> `service update --secret-rm pg_password --secret-add pg_password_v2`, then remove the old.

In a stack file:
```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/pg_password
    secrets:
      - pg_password
secrets:
  pg_password:
    external: true        # created out-of-band with `docker secret create`
    # file: ./pg_password.txt   # or let the stack create it from a file
```

## Networks

```bash
# overlay = multi-host virtual network; services on it reach each other by service name
docker network create -d overlay --attachable backend
#   --attachable lets standalone `docker run` containers join too (handy for debugging)

docker network ls                       # SCOPE = swarm for overlays
docker network inspect backend

# encrypt the data plane (IPSec) for sensitive east-west traffic
docker network create -d overlay --opt encrypted secure-net
```

Service discovery & the routing mesh:
- Any task reaches another service at `http://<service-name>:<port>` (built-in DNS, VIP load-balanced).
- A published port (`-p 8080:80`) is reachable on **every** node's IP — the **routing mesh**
  forwards to a healthy task wherever it lives.
- Skip the mesh and bind directly on the host (1 task/node, e.g. ingress LB):
```yaml
ports:
  - target: 80
    published: 80
    mode: host        # default is "ingress" (routing mesh)
```

## Debug & inspect

```bash
docker service ps <svc> --no-trunc                     # #1 tool — why a task won't start
docker service ps <svc> -f 'desired-state=running'     # filter noise from old failed tasks
docker service logs --since 10m -f <svc>
docker stack ps <stack> --no-trunc

docker node ls                          # any node Down / Unreachable?
docker events                           # live stream of scheduling decisions
docker inspect <task-id>                # task-level detail

# task stuck "Pending" -> no node satisfies constraints/resources. Check placement:
docker service inspect <svc> --format '{{json .Spec.TaskTemplate.Placement}}'
```

> **Gotcha:** a crash-looping task shows a new failed task ID each retry. `--no-trunc` on
> `service ps` reveals the real error ("no suitable node", "image not found", OOM, etc.).

## Maintenance & teardown

```bash
# autolock — managers need an unlock key after reboot (protects Raft store at rest)
docker swarm update --autolock=true
docker swarm unlock-key                 # print current key (save it!)
docker swarm unlock                     # paste key after a manager reboots
docker swarm unlock-key --rotate

# backup the swarm state (stop docker on a manager first for a clean copy)
systemctl stop docker
tar czf swarm-backup.tgz /var/lib/docker/swarm
systemctl start docker

docker swarm leave                      # worker leaves the swarm
docker swarm leave --force              # last manager / force-leave
```

> **Gotcha:** losing manager quorum (e.g. 2 of 3 managers gone) = no scheduling/changes until
> restored. Recover from the survivor with `docker swarm init --force-new-cluster`, then
> re-add managers.

## Full stack example

`docker-compose.yml` — deploy with `docker stack deploy -c docker-compose.yml odoo`:

```yaml
services:
  web:
    image: registry.example.com/odoo:19   # pre-built & pushed; stack deploy won't build
    depends_on:
      - db
    ports:
      - "8069:8069"
    networks:
      - frontend
      - backend
    environment:
      HOST: db
      USER: odoo
      PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
      restart_policy:
        condition: on-failure
        max_attempts: 3
      resources:
        limits:       { cpus: "2",   memory: 1G }
        reservations: { cpus: "0.5", memory: 256M }
      placement:
        constraints:
          - node.role == worker

  db:
    image: postgres:18
    networks:
      - backend
    environment:
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - db-data:/var/lib/postgresql/data/pgdata
    secrets:
      - db_password
    deploy:
      replicas: 1                 # stateful: keep it 1 and pin it
      placement:
        constraints:
          - node.labels.disk == ssd

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    driver_opts:
      encrypted: "true"

volumes:
  db-data:

secrets:
  db_password:
    external: true               # docker secret create db_password ./db_password.txt
```

> **Gotcha:** Swarm volumes are **local to the node** the task lands on. For a stateful DB
> either pin it to one node (constraint, as above) or back the volume with NFS/shared storage —
> otherwise a reschedule loses the data.
