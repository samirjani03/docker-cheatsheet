# 🐳 Docker Cheatsheet

Every command, every flag, every use case, one page. No hunting through random blogs.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Installation](#installation)
- [Dockerfile](#dockerfile)
- [Images](#images)
- [Containers](#containers)
- [Docker Compose](#docker-compose)
- [Volumes & Storage](#volumes--storage)
- [Networks](#networks)
- [Registry Operations](#registry-operations)
- [Resource Limits](#resource-limits)
- [Buildx (Multi-Platform Builds)](#buildx-multi-platform-builds)
- [Swarm Basics](#swarm-basics)
- [Cleanup](#cleanup)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [.dockerignore](#dockerignore)
- [Quick Reference Card](#quick-reference-card)

---

## Core Concepts

Docker packages your app plus everything it needs into a **container** that runs the same everywhere: your laptop, a server, the cloud. No "works on my machine" excuses.

| Term | What it means |
|---|---|
| **Image** | A read-only blueprint. Built from a Dockerfile, stored in layers. |
| **Container** | A running instance of an image. Isolated process with its own filesystem. |
| **Dockerfile** | Text file with build instructions, basically a shell script for building images. |
| **Volume** | Persistent storage that survives container deletion. |
| **Network** | Virtual wiring, how containers find and talk to each other. |
| **Registry** | Image storage and distribution. Docker Hub is the default public one. |

**Analogy table:**

| Concept | Real-world analogy | CLI object |
|---|---|---|
| Image | Recipe (read-only) | `docker image` |
| Container | Dish made from the recipe | `docker container` |
| Dockerfile | Written recipe instructions | `docker build` |
| Volume | Shared fridge (data survives dishes) | `docker volume` |
| Network | Kitchen wiring | `docker network` |
| Registry | Cookbook library (Docker Hub) | `docker pull` / `docker push` |

**Architecture flow:**

```
DOCKER CLIENT (docker CLI)
    ↓ sends commands via REST API
DOCKER DAEMON (dockerd)
    ↓ manages images, containers, networks, volumes
CONTAINERD
    ↓ manages container lifecycle
RUNC
    ↓ actually creates & runs the container
CONTAINER
```

---

## Installation

### Linux (Ubuntu/Debian)

```bash
# Official convenience script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to the docker group (skip typing sudo every time)
sudo usermod -aG docker $USER
newgrp docker  # or log out and back in

# Verify
docker --version
docker run hello-world
```

### macOS

```bash
# Download Docker Desktop from:
# https://www.docker.com/products/docker-desktop/

# Or via Homebrew:
brew install --cask docker
```

### Windows

```powershell
# Enable WSL 2 first if needed
wsl --install

# Then download Docker Desktop from docker.com
# Requires WSL 2 backend for best performance
```

---

## Dockerfile

A Dockerfile is a plain text file named `Dockerfile` (no extension). Each instruction creates a layer.

### Full production example

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Run (smaller final image)
FROM node:20-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000
ENV NODE_ENV=production
USER app
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Every instruction, explained

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image (must be first) | `FROM node:20-alpine` |
| `FROM ... AS` | Name a build stage | `FROM golang:1.21 AS builder` |
| `RUN` | Execute commands during build | `RUN apt-get update && apt-get install -y curl` |
| `COPY` | Copy files from host to image | `COPY . /app` |
| `COPY --from` | Copy from another stage | `COPY --from=builder /app/bin /app/bin` |
| `ADD` | Copy + auto-extract tar/zip | `ADD archive.tar.gz /app` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `ENV` | Set environment variable | `ENV NODE_ENV=production` |
| `ARG` | Build-time variable only | `ARG VERSION=1.0` |
| `EXPOSE` | Document port (doesn't publish it) | `EXPOSE 8080` |
| `VOLUME` | Create a mount point | `VOLUME /data` |
| `USER` | Switch user | `USER node` |
| `CMD` | Default command (overridable) | `CMD ["node", "server.js"]` |
| `ENTRYPOINT` | Fixed command, CMD becomes args | `ENTRYPOINT ["python", "app.py"]` |
| `HEALTHCHECK` | Container health check | `HEALTHCHECK CMD curl -f localhost:3000/health` |
| `ONBUILD` | Trigger for child images | `ONBUILD COPY . /app` |
| `STOPSIGNAL` | Signal used to stop container | `STOPSIGNAL SIGTERM` |
| `SHELL` | Change default shell | `SHELL ["/bin/bash", "-c"]` |
| `LABEL` | Add metadata | `LABEL maintainer="samir@example.com"` |

### CMD vs ENTRYPOINT (the thing everyone mixes up)

```dockerfile
# CMD: default command, fully overridable
CMD ["echo", "hello"]
# docker run myimg           → "hello"
# docker run myimg world     → "world" (CMD replaced entirely)

# ENTRYPOINT: fixed command, CMD is appended as arguments
ENTRYPOINT ["echo"]
CMD ["hello"]
# docker run myimg           → "hello" (ENTRYPOINT + CMD)
# docker run myimg world     → "world" (ENTRYPOINT + override)

# Best practice: combine both
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
# docker run myimg              → docker-entrypoint.sh node server.js
# docker run myimg npm test     → docker-entrypoint.sh npm test

# Override ENTRYPOINT at runtime
docker run --entrypoint /bin/bash myimg
```

### HEALTHCHECK

```dockerfile
# --interval (default 30s), --timeout (default 30s),
# --start-period (grace period), --retries (default 3)

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Disable an inherited healthcheck
HEALTHCHECK NONE
```

### Dockerfile best practices

| ✅ Do | ❌ Don't |
|---|---|
| Pin specific tags: `node:20-alpine` | Use `:latest` in production |
| Multi-stage builds for small images | Ship build tools in the final image |
| COPY package.json → RUN install → COPY rest | COPY everything then install (kills cache) |
| Combine RUN commands: `RUN a && b && c` | Separate RUN per package (bloats layers) |
| Use a `.dockerignore` file | COPY node_modules, .git, logs |
| Run as non-root USER | Run containers as root in production |
| Use `COPY` (explicit) | Use `ADD` unless you need tar extraction |
| Clean up in the same RUN | Clean in a separate RUN (layer still holds the data) |

---

## Images

### Building

```bash
# Basic build
docker build -t myapp:latest .
docker build -t myapp:v1.0 -t myapp:latest .        # multiple tags

# Build args
docker build --build-arg VERSION=2.0 -t myapp:2.0 .

# Custom Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# No cache (full rebuild)
docker build --no-cache -t myapp:latest .

# Tag for registry
docker build -t samir/myapp:v1.0 .
docker build -t ghcr.io/samir/myapp:v1.0 .          # GitHub Container Registry
```

### Listing & inspecting

```bash
docker images                                        # list all images
docker images -a                                      # include intermediate layers
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
docker image ls -f dangling=true                      # untagged (<none>) images only
docker image ls -f "reference=myapp*"                  # filter by name

docker image inspect nginx:latest                      # full JSON metadata
docker image inspect -f '{{.Os}}' nginx:latest         # extract one field
docker history nginx:latest                             # layer history & sizes
docker history --no-trunc nginx:latest                  # full command per layer
```

### Tagging

```bash
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
docker tag myapp:latest samir/myapp:v1.0
docker tag abc123 samir/myapp:latest                    # tag by image ID
```

### Removing

```bash
docker rmi nginx:latest
docker rmi -f abc123                                     # force remove (even if in use)
docker image prune                                       # remove dangling images
docker image prune -a                                     # remove ALL unused images
docker image prune -a --filter "until=24h"                 # unused older than 24h
```

### Saving & loading (offline transfer)

```bash
docker save -o myimage.tar myapp:latest
docker save myapp:latest | gzip > myimage.tar.gz        # compressed
docker load -i myimage.tar
docker load < myimage.tar.gz

# save/load preserves layers & metadata. For a flat filesystem, use export instead.
```

### Commit (snapshot a container into an image)

```bash
docker commit mycontainer myimage:v2
docker commit -m "added nginx config" -a "Samir" mycontainer myimage:v2
# Prefer Dockerfiles over commit — commits are opaque and hard to reproduce.
```

### Import / Export (flat filesystem)

```bash
docker export mycontainer > mycontainer.tar              # flat FS, no layers/history
docker import mycontainer.tar myimage:flat
# export/import loses metadata. Use save/load to keep layers & history.
```

---

## Containers

### `docker run` — the swiss army knife

```bash
# Basic patterns
docker run nginx                                  # run, attach, Ctrl+C to stop
docker run -d nginx                               # detached (background)
docker run --name myweb nginx                      # give it a name
docker run --rm nginx                               # auto-remove on stop
docker run -it ubuntu bash                          # interactive + TTY

# Port mapping (-p HOST:CONTAINER)
docker run -p 8080:80 nginx
docker run -p 3000:3000 -p 9229:9229 myapp          # multiple ports
docker run -p 127.0.0.1:8080:80 nginx                # localhost only
docker run -P nginx                                   # publish all EXPOSE'd ports randomly

# Volumes
docker run -v /host/path:/container/path nginx       # bind mount
docker run -v myvolume:/data nginx                    # named volume
docker run -v /container/path nginx                    # anonymous volume
docker run --mount type=bind,src=/host,dst=/app nginx  # verbose syntax

# Environment variables
docker run -e MY_VAR=hello nginx
docker run -e DB_HOST=db -e DB_PORT=5432 nginx
docker run --env-file .env.prod myapp

# Resource limits
docker run --memory="256m" nginx
docker run --cpus="1.5" nginx
docker run --cpuset-cpus="0-2" nginx
docker run --shm-size="256m" nginx                     # important for browsers/DBs

# Restart policies
docker run --restart=no nginx                          # default
docker run --restart=always nginx
docker run --restart=unless-stopped nginx
docker run --restart=on-failure:5 nginx                 # max 5 retries

# Other useful flags
docker run --init nginx                                  # handles zombie processes
docker run --dns 8.8.8.8 nginx
docker run --add-host myhost:192.168.1.5 nginx
docker run --privileged nginx                             # full host access, risky
docker run --cap-drop=ALL nginx                            # drop all, add back selectively
docker run --read-only nginx
docker run -u 1000:1000 nginx                              # run as specific user:group
```

> **All-in-one example:**
> `docker run -d --name myapp --restart=unless-stopped -p 3000:3000 -v $(pwd):/app -e NODE_ENV=prod --memory="512m" --cpus="1" myapp:latest`

### Lifecycle

```bash
# Start & stop
docker start myweb
docker stop myweb                                        # graceful (SIGTERM → wait → SIGKILL)
docker stop -t 30 myweb                                    # wait 30s before SIGKILL
docker kill myweb                                            # force kill immediately
docker restart myweb
docker pause myweb                                            # freeze
docker unpause myweb

# Removing
docker rm myweb
docker rm -f myweb                                             # force remove (running or not)
docker rm $(docker ps -aq)                                      # remove ALL containers
docker container prune                                            # remove all stopped containers

# Rename
docker rename myweb mynewweb
```

### Executing commands inside running containers

```bash
docker exec myweb ls /app
docker exec -it myweb bash
docker exec -it myweb sh                                # for alpine (no bash)
docker exec -u root myweb whoami                          # run as different user
docker exec -w /tmp myweb pwd                                # from a specific dir
```

### Listing & formatting

```bash
docker ps                                             # running containers
docker ps -a                                            # all, including stopped
docker ps -f "name=myweb"
docker ps -f "status=exited"
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

### Logs

```bash
docker logs myweb
docker logs -f myweb                                    # follow (like tail -f)
docker logs --tail 100 myweb
docker logs --since 30m myweb
docker logs -t myweb                                     # include timestamps
```

### Inspection & monitoring

```bash
docker inspect myweb
docker inspect -f '{{.NetworkSettings.IPAddress}}' myweb
docker inspect -f '{{.State.Status}}' myweb

docker top myweb                                        # processes inside container
docker stats                                             # live CPU/MEM/IO for all containers
docker stats --no-stream                                    # one-shot snapshot
docker port myweb                                             # port mappings
docker diff myweb                                              # files changed since start
docker events                                                    # real-time Docker events
```

### Copying files (host ↔ container)

```bash
docker cp myfile.txt myweb:/app/file.txt
docker cp myweb:/app/log.txt ./log.txt
docker cp myweb:/app/data/ ./data/
```

### Updating a running container's config

```bash
docker update --memory="512m" --cpus="2" myweb
docker update --restart=always myweb
# Can't update ports, volumes, or env vars on a running container.
# For those: stop → create new → start.
```

---

## Docker Compose

For multi-container apps. Define everything in `docker-compose.yml` (or `compose.yaml`).

### Full example

```yaml
name: myproject

services:
  web:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
      args:
        - NODE_ENV=production
    image: samir/frontend:v1.0
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://api:4000
    env_file:
      - .env.web
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    volumes:
      - ./frontend/src:/app/src
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  api:
    build: ./backend
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    profiles:
      - full

  worker:
    build: ./worker
    command: python worker.py
    depends_on:
      - api
      - db
    deploy:
      replicas: 2

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - cache_data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - web
      - api

volumes:
  pgdata:
  cache_data:

networks:
  frontend:
  backend:
    internal: true
```

### All the CLI commands you'll actually use

```bash
# Core
docker compose up                                    # start (foreground)
docker compose up -d                                  # background
docker compose up -d --build                            # rebuild then start
docker compose up -d --force-recreate                     # recreate even if unchanged
docker compose up -d --scale web=3                            # scale
docker compose up web api                                        # specific services only

docker compose down                                            # stop and remove containers/networks
docker compose down -v                                            # also remove volumes (deletes data!)
docker compose down --rmi all                                        # also remove images

# Lifecycle
docker compose start
docker compose stop
docker compose restart
docker compose restart web                                             # one service

# Viewing
docker compose ps
docker compose ps -a
docker compose images
docker compose top
docker compose port web 3000

# Logs
docker compose logs
docker compose logs -f web
docker compose logs --tail=100 web

# Exec
docker compose exec web bash
docker compose exec -T web ls /app                                      # no TTY, for scripts
docker compose run web npm test                                            # one-off, new container
docker compose run --rm web npm test

# Build
docker compose build
docker compose build --no-cache web

# Config
docker compose config                                                        # resolved config
docker compose config --services                                              # list service names

# Other
docker compose pull
docker compose push
docker compose watch                                                            # sync on file changes (v2.22+)
```

### Environment variable priority (highest to lowest)

1. Shell environment variables
2. `--env-file` on the command line
3. `env_file:` in compose file (service-level overrides project-level)
4. `environment:` in compose file
5. `.env` file in project directory

```bash
# .env file
DB_PASSWORD=supersecret
API_KEY=sk-abc123
COMPOSE_PROJECT_NAME=myproject
```

```yaml
# Usage in compose.yaml
environment:
  - PASSWORD=${DB_PASSWORD}
  - KEY=${API_KEY:-default_value}       # default if not set
  - REQUIRED=${API_KEY:?err}            # error if not set
```

---

## Volumes & Storage

| Type | Managed by | Location | Use case |
|---|---|---|---|
| Named Volume | Docker | `/var/lib/docker/volumes/` | Production data persistence |
| Bind Mount | You | Any host path | Development (live reload, config sharing) |
| Anonymous Volume | Docker | Random dir | Ephemeral, deleted with container unless `-v` flag |
| tmpfs | Kernel | RAM | Sensitive temp data, no disk writes |

```bash
# Create & inspect
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume ls -f dangling=true                        # unused volumes only

# Remove
docker volume rm mydata
docker volume prune                                          # remove all unused local volumes

# Usage patterns
docker run -v mydata:/app/data myapp                          # named volume
docker run -v /home/samir/project:/app myapp                    # bind mount
docker run -v $(pwd):/app myapp                                    # current directory
docker run -v /host/config:/app/config:ro myapp                       # read-only bind mount
docker run -v /app/data myapp                                             # anonymous volume
docker run --tmpfs /tmp:rw,size=64m myapp                                    # in-memory

# --mount (verbose, recommended for production)
docker run --mount type=bind,src=$(pwd)/config,dst=/app/config,readonly myapp
docker run --mount type=volume,src=mydata,dst=/data,volume-driver=local myapp
```

### Backup & restore a volume

```bash
# Backup
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar czf /backup/mydata-backup.tar.gz -C /data .

# Restore
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /data
```

---

## Networks

| Driver | Behavior | Use case |
|---|---|---|
| `bridge` | Default. Containers on same bridge can talk, DNS by container name. | Single-host apps |
| `host` | Container shares host's network stack directly. | Max performance, no isolation needed |
| `none` | No networking at all. | Security isolation |
| `overlay` | Multi-host networking (Swarm). | Distributed apps across nodes |
| `macvlan` | Container gets its own MAC/IP on physical network. | Legacy apps expecting real IPs |
| `ipvlan` | Like macvlan but shares MAC, lower overhead. | When MAC limits are a problem |

```bash
# Create & manage
docker network create mynet
docker network create --driver bridge --subnet=172.28.0.0/16 --gateway=172.28.0.1 mynet
docker network create --internal mynet                    # no external connectivity
docker network ls
docker network inspect mynet
docker network rm mynet
docker network prune

# Connect & disconnect
docker network connect mynet myweb
docker network connect --alias db mynet myweb                # extra DNS alias
docker network disconnect mynet myweb

# Using networks
docker run --network mynet nginx
docker run --network host nginx
docker run --network none nginx

# DNS: containers on the same network resolve by name
docker run -d --name db --network mynet postgres
docker run -d --name web --network mynet myapp
docker exec web ping db                                         # resolves to db's IP
```

---

## Registry Operations

### Docker Hub

```bash
# Login/logout
docker login
docker login -u samir
docker login --password-stdin < password.txt                # CI/CD
docker logout

# Push/pull
docker pull nginx:latest
docker pull python:3.12-slim
docker pull --all-tags samir/myapp
docker pull --platform linux/amd64 nginx                      # specific architecture

docker push samir/myapp:v1.0
docker push --all-tags samir/myapp

# Search
docker search nginx
docker search --filter is-official=true ubuntu
docker search --filter stars=100 node
```

### Other registries

```bash
# GitHub Container Registry
echo $CR_PAT | docker login ghcr.io -u samir --password-stdin
docker tag myapp ghcr.io/samir/myapp:v1.0
docker push ghcr.io/samir/myapp:v1.0

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login -u AWS --password-stdin ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
docker tag myapp:latest ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Google Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev
docker push us-central1-docker.pkg.dev/PROJECT/repo/myapp:v1.0

# Azure Container Registry
az acr login --name myregistry
docker push myregistry.azurecr.io/myapp:v1.0

# Self-hosted / private
docker login registry.example.com
docker tag myapp registry.example.com/myapp:v1.0
docker push registry.example.com/myapp:v1.0
```

### Moving images without a registry

```bash
# Save → scp → load (offline / air-gapped)
docker save myapp:latest -o myapp.tar
scp myapp.tar user@server:/tmp/
ssh user@server "docker load -i /tmp/myapp.tar"

# Or pipe directly over SSH
docker save myapp:latest | gzip | ssh user@server "gunzip | docker load"
```

---

## Resource Limits

Resource limits stop one container from hogging the whole machine.

```bash
# Memory
docker run --memory="512m" nginx                                # hard limit
docker run --memory="512m" --memory-reservation="256m" nginx      # soft limit
docker run --memory="512m" --memory-swap="1g" nginx                 # total (mem+swap) limit
docker run --memory-swappiness=10 nginx                                # 0-100, lower = less swap

# CPU
docker run --cpus="1.5" nginx
docker run --cpu-shares=512 nginx                                        # relative weight (default 1024)
docker run --cpuset-cpus="0-3" nginx                                        # pin to specific cores
docker run --cpu-period=100000 --cpu-quota=50000 nginx                        # 50% of a CPU

# Disk I/O
docker run --blkio-weight=500 nginx
docker run --device-read-bps=/dev/sda:1mb nginx
docker run --device-write-bps=/dev/sda:1mb nginx

# Other limits
docker run --pids-limit=100 nginx
docker run --ulimit nofile=1024:2048 nginx
docker run --shm-size="256m" nginx

# Update on the fly
docker update --memory="1g" --cpus="2" myweb
```

---

## Buildx (Multi-Platform Builds)

```bash
# Create a builder
docker buildx create --name mybuilder --use
docker buildx ls
docker buildx inspect mybuilder

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t samir/myapp:latest .

# Build + push in one shot
docker buildx build --platform linux/amd64,linux/arm64 \
  -t samir/myapp:latest --push .

# Build without pushing (load into local docker)
docker buildx build --platform linux/amd64,linux/arm64 \
  -t samir/myapp:latest -o type=docker .

# Cache
docker buildx build --cache-to type=registry,ref=samir/myapp:cache \
  --cache-from type=registry,ref=samir/myapp:cache -t samir/myapp:latest .

# Cleanup
docker buildx rm mybuilder
docker buildx prune
```

---

## Swarm Basics

```bash
# Init & join
docker swarm init --advertise-addr 192.168.1.10
docker swarm join --token SWMTKN-... 192.168.1.10:2377          # worker joins
docker swarm join-token manager
docker swarm leave --force

# Nodes
docker node ls
docker node update --availability drain node1
docker node promote node2                                            # worker → manager
docker node demote node1                                                # manager → worker

# Services
docker service create --name web --replicas 3 -p 80:80 nginx
docker service ls
docker service scale web=5
docker service update --image nginx:alpine web                          # rolling update
docker service rollback web
docker service rm web

# Stacks (compose in swarm)
docker stack deploy -c docker-compose.yml mystack
docker stack ls
docker stack ps mystack
docker stack rm mystack
```

---

## Cleanup

```bash
# Check disk usage
docker system df                                     # summary
docker system df -v                                     # per-image/container/volume detail

# Prune by category
docker container prune                                     # stopped containers
docker image prune                                             # dangling images
docker image prune -a                                             # ALL unused images
docker volume prune                                                 # unused volumes
docker network prune                                                    # unused networks
docker builder prune                                                        # build cache

# With filters
docker container prune --filter "until=24h"
docker image prune -a --filter "until=7d"

# Nuclear options
docker system prune                                                              # containers + networks + dangling images + cache
docker system prune -a                                                              # + ALL unused images
docker system prune -a --volumes                                                       # + volumes (data loss!)

# Manual all-in-one, if you need exact control
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
docker volume rm $(docker volume ls -q)
docker network rm $(docker network ls -q)
```

---

## Debugging & Troubleshooting

### Common workflows

```bash
# "Why won't my container start?"
docker ps -a
docker logs mycontainer
docker inspect mycontainer                          # check State.ExitCode, State.Error

# "Get a shell inside to poke around"
docker exec -it mycontainer bash
docker exec -it mycontainer sh                        # alpine images

# "Container exits immediately, need to debug it"
docker run -it --entrypoint /bin/bash broken-image

# "What's eating my disk space?"
docker system df -v
du -sh /var/lib/docker/

# "Is my port mapping working?"
docker port mycontainer
curl localhost:PORT

# "What's running inside?"
docker top mycontainer

# "Live resource usage"
docker stats

# "What files changed?"
docker diff mycontainer                                # A=added, C=changed, D=deleted

# "System info"
docker info
docker version
```

### Common issues & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits immediately | No foreground process | Make sure CMD/ENTRYPOINT is a foreground process |
| Port already in use | Another process on host port | `lsof -i :PORT`, change host port mapping |
| "Permission denied" on volume | UID/GID mismatch | `docker run -u $(id -u):$(id -g)` |
| Can't connect to localhost | Container has its own localhost | Use `host.docker.internal` (Mac/Win) or `--network host` |
| Container can't reach internet | DNS or network issue | `docker run --dns 8.8.8.8`, restart docker daemon |
| OOM killed | Memory limit too low | `docker run --memory="1g"` or check for memory leaks |
| Build is slow | Bad layer ordering | COPY package.json → RUN install → COPY rest |
| Image too large | No multi-stage build | Use multi-stage builds, alpine base images |

---

## .dockerignore

Placed next to your Dockerfile. Speeds up builds and keeps secrets out of images.

```
# Version control
.git
.gitignore

# Dependencies
node_modules
vendor
__pycache__
*.pyc
.venv
venv

# Build artifacts
dist
build
*.exe
*.dll

# IDE & editor
.vscode
.idea
*.swp

# OS
.DS_Store
Thumbs.db

# Environment & secrets
.env
.env.*
*.key
*.pem
secrets/

# Logs & temp
*.log
logs/
tmp/

# Docker
Dockerfile
docker-compose.yml
.dockerignore

# Tests & docs (unless needed)
tests/
__tests__/
docs/
*.md

# Use negation (!) to include something excluded above
!README.md
!.env.example
```

---

## Quick Reference Card

```bash
# Images
docker pull nginx:latest
docker build -t myapp:v1 .
docker images
docker rmi myapp:v1

# Containers
docker run -d --name app -p 3000:3000 myapp
docker ps -a
docker stop app && docker rm app
docker exec -it app bash
docker logs -f app

# Compose
docker compose up -d
docker compose down
docker compose logs -f web
docker compose exec web bash

# Cleanup
docker system prune -a
docker system df

# Debug
docker inspect container_name
docker stats
docker events
```

---

*The only Docker reference you'll ever need. Contributions welcome.*
