

```bash
sudo apt install -y docker.io docker-compose-v2 docker-buildx
```


## Docker Compose YML

```bash
docker compose ps            # see what's running

docker compose up -d --build   # first time (builds the image)
docker compose up -d           # subsequent runs (detached)

docker compose down          # stop and remove containers
docker compose down -v       # same + deletes volumes (careful: wipes filestore)
docker compose stop          # stop containers but don't remove them
docker compose start         # start them back up after stop
docker compose restart       # restart containers
docker compose logs -f odoo  # follow logs
```





docker-compose.yml
```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=odoo
      - POSTGRES_USER=odoo
  odoo:
    image: ..
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

```bash
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
```




Build
```bash
docker build -t quran-api:latest -t quran-api:1.0.0 --no-cache .
```
Run

```bash
docker run --env-file .env -p 8000:8000 --name quran-api quran-api:latest
# run -d detached
docker run -d --env-file .env -p 8000:8000 --name quran-api quran-api:latest
```
other stuff
```bash
# List all images
docker images
# Remove an image
docker rmi quran-api:latest
# Remove all unused images
docker image prune


# lists all container IDs(both running and stopped)
docker ps -aq
# force remove a running container
docker rm -f <the-container-id>
# force remove all containers
docker rm --force $(docker ps -aq)

# remove anonymous volumes
docker system prune -a --volumes
# removes all stopped containers
docker system prune -a

# portable image file
# Save image to a file
docker save -o quran-api.tar quran-api:latest
# Load it back on another machine
docker load -i quran-api.tar
```