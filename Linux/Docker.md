
# Docker

## packages
```bash
sudo apt install -y docker.io docker-compose-v2 docker-buildx
```
```bash
sudo usermod -aG docker $USER
```
```bash
newgrp docker
```

## Inspect / debug 
```bash
docker compose logs -f
docker compose logs -f odoo  # follow logs
docker exec -it app sh
docker inspect --format '{{.State.Pid}}' app
nsenter -t <pid> -n ip addr      # enter container netns from host
#Inspect image layers / size
docker history --no-trunc app
dive user/app                    # interactive layer explorer
```

## List
```bash
docker images
docker images | grep my-app

docker compose ps   # see what's running
docker ps           # running containers with ports
docker ps -aq       # all container IDs (running & stopped)

docker volume ls    # List Volumes
docker ps -a --format '{{ .Names }} - {{ .Mounts }}' # volumes mounted to container

sudo ss -tulpn | grep docker   # Show Docker-related listening ports
# show with filters
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

docker ps -q | xargs -I {} docker inspect -f \
'{{.Name}} | {{.Config.Image}} | IP={{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} | Ports={{range $p, $conf := .NetworkSettings.Ports}}{{$p}} -> {{(index $conf 0).HostIp}}:{{(index $conf 0).HostPort}} {{end}}' {}

docker inspect -f '{{.Name}} -> {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -q)
```

## Build

```bash
docker compose up -d --build   # build and run

docker build -t my-api:latest -t my-api:1.0.0 --no-cache .
docker build --network host -t my-app/backend:latest .
docker build -t my-app:1.0 .

# tag & push to registry
docker tag my-api:latest 192.168.8.25:5000/my-api:latest
docker push 192.168.8.25:5000/my-api:latest
```

## Run & Stop

```bash
# stop & remove
docker stop $(docker ps -q) && docker rm $(docker ps -aq)

docker compose up -d           # subsequent runs (detached)
docker compose down            # stop and remove containers
docker compose down -v         # same + deletes volumes (careful: wipes filestore)

docker compose stop          # stop containers but don't remove them
docker compose start         # start them back up after stop
docker compose restart       # restart containers

docker run --env-file .env -p 8000:8000 --name my-api my-api:latest
docker run -d --env-file .env -p 8000:8000 --name my-api my-api:latest  # run -d detached

# Network
docker network create --internal backend # isolated unreachable from internet
docker run --network backend db
```

## Remove

```bash
docker rm -f container-id     # remove container
docker rmi -f image-id        # remove image
docker volume rm volume-id    # remove volume

docker rm -f $(docker ps -aq)             # all containers
docker rmi -f $(docker images -q)         # all images
docker volume rm $(docker volume ls -q)   # all volumes

docker image prune                # remove all unused images
docker system prune -a --volumes  # remove anonymous volumes
docker system prune -a            # removes all stopped containers
docker volume prune -f            # remove all unused volumes

# prune aggressively
docker system prune -af --volumes
docker builder prune -af
```

## Save & Load
```bash
docker save -o my-api.tar my-api:latest   # Save image to a file
docker load -i my-api.tar                 # Load it back on another machine
```

## SWARM

```bash
docker swarm init --advertise-addr <manager-IP>
# verify > Swarm: active
docker info
# list of nodes in the swarm
docker node ls
# get the token
docker swarm join-token worker
# run on worker to join
docker swarm join --token <join-token> <manager-IP>:2377
# inspect a node from the manager
docker node inspect <worker-name>

# autolock
docker swarm update --autolock=true
# to unlock
docker swarm unlock
# forgot the key
docker swarm unlock-key

docker node update --availability drain [node name]
docker node update --availability pause [node name]
docker node update --availability active [node name]

# promote/demote to a manager
docker node promote [node name]
docker node demote [node name]

docker swarm leave
docker swarm leave --force

docker node rm [node name 1] [node name 2]
```



<br>

## docker-compose.yml examples

```yaml
services:
  web:
    image: odoo:19.0
    depends_on:
      - db
    ports:
      - "8069:8069"
    volumes:
      - odoo-web-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=odoo_secure_password
  db:
    image: postgres:18
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=odoo_secure_password
      - POSTGRES_USER=odoo
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - odoo-db-data:/var/lib/postgresql/data/pgdata
volumes:
  odoo-web-data:
  odoo-db-data:
```

```yaml
services:
  db:
    image: postgres:18
    container_name: postgres_db
    restart: unless-stopped
    shm_size: 128mb
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql
volumes:
  pgdata:
```

local filestore
```yaml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)
    volumes:
      - odoo_filestore:/var/lib/odoo
      - ./odoo.conf:/etc/odoo/odoo.conf:ro
      - ./extra-addons:/mnt/extra-addons
    environment:
      - ODOO_RC=/etc/odoo/odoo.conf

volumes:
  odoo_filestore:
```


dedicated NFS server
```yaml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)
    volumes:
      - /mnt/odoo_filestore:/var/lib/odoo
      - ./odoo.conf:/etc/odoo/odoo.conf:ro
      - ./extra-addons:/mnt/extra-addons
    environment:
      - ODOO_RC=/etc/odoo/odoo.conf
```

or

```yml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)
    volumes:
      - odoo-filestore:/var/lib/odoo
      - ./odoo.conf:/etc/odoo/odoo.conf:ro
      - ./extra-addons:/mnt/extra-addons
    environment:
      - ODOO_RC=/etc/odoo/odoo.conf
volumes:
  odoo-filestore:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.8.10,nfsvers=4,rw
      device: ":/srv/odoo/filestore"




# Healthcheck so dependents wait for ready, not just started:
db:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U odoo"]
    interval: 5s
    retries: 5
odoo:
  depends_on:
    db: { condition: service_healthy }


```



## Advanced



### BuildKit cache mounts — cache deps across builds

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```
```bash
DOCKER_BUILDKIT=1 docker build .
```

### Secret mounts — no secrets baked into layers

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```



## Dockerfile
```dockerfile
# Distroless / scratch — minimal attack surface
FROM scratch
COPY --from=build /app /app


# Multi-stage builds — slim images, separate build/runtime
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app
FROM gcr.io/distroless/base
COPY --from=build /src/app /app
ENTRYPOINT ["/app"]
```



```bash
# Multi-arch builds — amd64 + arm64 in one push
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t user/app:1.0 --push .

# SBOM + provenance — supply-chain attestation
docker buildx build --sbom=true --provenance=true -t user/app .
docker scout cves user/app          # vuln scan

# Resource limits
docker run --memory=512m --cpus=1.5 --pids-limit=100 app

# Read-only + drop caps — hardening
docker run --read-only --cap-drop=ALL --security-opt=no-new-privileges --tmpfs /tmp app

# pulls files from a dead/exited container. 
docker cp app:/app/out.json ./

# Squash build cache from registry
docker buildx build --cache-from=type=registry,ref=user/app:cache \
  --cache-to=type=registry,ref=user/app:cache,mode=max -t user/app .



```

Live restore / log rotation (/etc/docker/daemon.json)
 — keep containers running during daemon restart; stop logs eating disk. Use on every production host.

```json
{ 
"live-restore": true, 
"log-driver": "json-file", 
"log-opts": {"max-size":"10m","max-file":"3"} 
}
```