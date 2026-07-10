

## 🐳 Docker Cheatsheet

[💡 Core Concepts](#concepts) [📥 Install](#install) [🏗️
Dockerfile](#dockerfile) [📦 Images](#images) [🚀
Containers](#containers) [🐳 Compose](#compose) [💾 Volumes](#volumes)
[🌐 Networks](#networks) [☁️ Registry](#registry) [⚙️ Resources &
Limits](#resources) [🏗️ Buildx](#buildx) [🐝 Swarm (basics)](#swarm) [🧹
Cleanup](#cleanup) [🔍 Debug](#debug) [📝 .dockerignore](#dockerignore)
[⚡ Quick Ref](#quickref)


# 🐳 Docker --- The Complete Cheatsheet

Every command, every flag, every use case. One page. No hunting through
random blogs.\
[v2.0]{.tag} [Complete]{.tag .green} [Beginner-Friendly]{.tag .orange}

## 💡 Core Concepts (30 Seconds)

Docker packages your app + everything it needs into a **container** that
runs identically everywhere --- your laptop, a server, the cloud. No
\"works on my machine\" nonsense.

#### 🖼️ Image

A read-only blueprint / recipe. Built from a Dockerfile. Stored in
layers.

#### 📦 Container

A running instance of an image. Isolated process with its own
filesystem.

#### 📝 Dockerfile

Text file with build instructions. Think of it as a shell script for
building images.

#### 💾 Volume

Persistent storage that survives container deletion. A shared fridge for
your data.

#### 🌐 Network

Virtual wiring --- how containers find and talk to each other.

#### ☁️ Registry

Image storage & distribution. Docker Hub is the default public one.
:::
:::


  Concept      Real-World Analogy                     CLI Object
  ------------ -------------------------------------- --------------------
  Image        Recipe (read-only)                     `docker image`
  Container    Dish made from the recipe              `docker container`
  Dockerfile   Written recipe instructions            `docker build`
  Volume       Shared fridge (data survives dishes)   `docker volume`
  Network      Kitchen wiring                         `docker network`
  Registry     Cookbook library (Docker Hub)          `docker pull/push`
:::

### 🔧 Docker Architecture

    # Docker uses a client-server architecture:

    # DOCKER CLIENT (docker CLI)
    #     ↓  sends commands via REST API
    # DOCKER DAEMON (dockerd)
    #     ↓  manages images, containers, networks, volumes
    # CONTAINERD (containerd)
    #     ↓  manages container lifecycle
    # RUNSC (runc)
    #     ↓  actually creates & runs the container
    # CONTAINER

## 📥 Installation

### Linux (Ubuntu/Debian --- recommended way)

    # Official convenience script
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh

    # Add your user to docker group (no sudo needed after relogin)
    sudo usermod -aG docker $USER
    newgrp docker  # or log out and back in

    # Verify
    docker --version
    docker run hello-world

### macOS

    # Download Docker Desktop from:
    # https://www.docker.com/products/docker-desktop/
    # Or via Homebrew:
    brew install --cask docker

### Windows

    # Download Docker Desktop from docker.com
    # Requires WSL 2 backend for best performance
    wsl --install  # Enable WSL 2 first if needed

## 🏗️ Dockerfile --- Complete Reference

A Dockerfile is a plain text file named `Dockerfile` (no extension).
Each instruction creates a layer.

### Complete Dockerfile Example

    # ── Production-grade Node.js Dockerfile ──
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

### Every Dockerfile Instruction


  Instruction     Purpose                               Example
  --------------- ------------------------------------- -------------------------------------------------
  `FROM`          Base image (must be first)            `FROM node:20-alpine`
  `FROM ... AS`   Name a build stage                    `FROM golang:1.21 AS builder`
  `RUN`           Execute commands during build         `RUN apt-get update && apt-get install -y curl`
  `COPY`          Copy files from host to image         `COPY . /app`
  `COPY --from`   Copy from another stage               `COPY --from=builder /app/bin /app/bin`
  `ADD`           Copy + auto-extract tar/zip           `ADD archive.tar.gz /app`
  `WORKDIR`       Set working directory                 `WORKDIR /app`
  `ENV`           Set environment variable              `ENV NODE_ENV=production`
  `ARG`           Build-time variable only              `ARG VERSION=1.0`
  `EXPOSE`        Document port (doesn\'t publish it)   `EXPOSE 8080`
  `VOLUME`        Create a mount point                  `VOLUME /data`
  `USER`          Switch user                           `USER node`
  `CMD`           Default command (overridable)         `CMD ["node", "server.js"]`
  `ENTRYPOINT`    Fixed command (CMD becomes args)      `ENTRYPOINT ["python", "app.py"]`
  `HEALTHCHECK`   Container health check                `HEALTHCHECK CMD curl -f localhost:3000/health`
  `ONBUILD`       Trigger for child images              `ONBUILD COPY . /app`
  `STOPSIGNAL`    Signal to stop container              `STOPSIGNAL SIGTERM`
  `SHELL`         Change default shell                  `SHELL ["/bin/bash", "-c"]`
  `LABEL`         Add metadata                          `LABEL maintainer="samir@example.com"`
:::

### CMD vs ENTRYPOINT (critical distinction)

    # CMD: default command, fully overridable
    CMD ["echo", "hello"]
    # docker run myimg           → "hello"
    # docker run myimg world     → "world" (CMD replaced entirely)

    # ENTRYPOINT: fixed command, CMD is appended as arguments
    ENTRYPOINT ["echo"]
    CMD ["hello"]
    # docker run myimg           → "hello" (ENTRYPOINT + CMD)
    # docker run myimg world     → "world" (ENTRYPOINT + override)

    # ENTRYPOINT + CMD together (best practice)
    ENTRYPOINT ["docker-entrypoint.sh"]
    CMD ["node", "server.js"]
    # docker run myimg              → docker-entrypoint.sh node server.js
    # docker run myimg npm test     → docker-entrypoint.sh npm test

    # Override ENTRYPOINT at runtime:
    docker run --entrypoint /bin/bash myimg

### HEALTHCHECK

    # Options: --interval=DURATION (default 30s)
    #          --timeout=DURATION   (default 30s)
    #          --start-period=DURATION (grace period)
    #          --retries=N          (default 3)

    HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
      CMD curl -f http://localhost:3000/health || exit 1

    # NONE = disable inherited healthcheck
    HEALTHCHECK NONE

### Dockerfile Best Practices


  ✅ Do                                                                ❌ Don\'t
  -------------------------------------------------------------------- ------------------------------------------------
  Pin specific tags: `node:20-alpine`                                  Use `:latest` in production
  Multi-stage builds for small images                                  Ship build tools in final image
  COPY package.json → RUN install → COPY rest                          COPY everything then install (kills cache)
  Combine RUN commands: `RUN a && b && c`                              Separate RUN per package (bloats layers)
  Use `.dockerignore` file                                             COPY node_modules, .git, logs
  Run as non-root USER                                                 Run containers as root in production
  Use `COPY` (explicit)                                                Use `ADD` unless you need tar extraction
  Clean up in same RUN: `apt install && rm -rf /var/lib/apt/lists/*`   Clean in separate RUN (layer still holds data)

## 📦 Image Commands

### Building Images

    # Basic build
    docker build -t myapp:latest .                    # -t = tag, . = context dir
    docker build -t myapp:v1.0 -t myapp:latest .      # multiple tags

    # Build with build args
    docker build --build-arg VERSION=2.0 -t myapp:2.0 .

    # Build from specific Dockerfile
    docker build -f Dockerfile.prod -t myapp:prod .

    # No cache (force full rebuild)
    docker build --no-cache -t myapp:latest .

    # Build and tag for registry
    docker build -t samir/myapp:v1.0 .
    docker build -t ghcr.io/samir/myapp:v1.0 .        # GitHub Container Registry

### Listing & Inspecting

    docker images                                      # list all images
    docker image ls                                    # same thing
    docker images -a                                   # include intermediate layers
    docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
    docker image ls -f dangling=true                   # show untagged (<none>) images only
    docker image ls -f "reference=myapp*"              # filter by name

    docker image inspect nginx:latest                  # full JSON metadata
    docker image inspect -f '{{.Os}}' nginx:latest     # extract one field
    docker history nginx:latest                        # see layer history & sizes
    docker history --no-trunc nginx:latest             # full command for each layer

### Tagging & Renaming

    docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
    docker tag myapp:latest samir/myapp:v1.0
    docker tag abc123 samir/myapp:latest               # tag by image ID

### Removing Images

    docker rmi nginx:latest                            # remove by name:tag
    docker rmi abc123def456                            # remove by image ID
    docker rmi -f abc123                               # force remove (even if in use)
    docker image prune                                 # remove dangling (<none>) images
    docker image prune -a                              # remove ALL unused images
    docker image prune -a --filter "until=24h"         # unused older than 24h

### Saving & Loading (offline transfer)

    docker save -o myimage.tar myapp:latest            # export image as tar
    docker save myapp:latest | gzip > myimage.tar.gz   # compressed
    docker load -i myimage.tar                         # import from tar
    docker load < myimage.tar.gz                       # import compressed

    # NOTE: save/load preserves layers & metadata. For flat filesystem use export.

### Commit (create image from container)

    docker commit mycontainer myimage:v2               # snapshot container → image
    docker commit -m "added nginx config" -a "Samir" mycontainer myimage:v2
    # ⚠️ Prefer Dockerfiles over commit — commits are opaque and unreproducible.

### Import/Export (flat filesystem)

    docker export mycontainer > mycontainer.tar        # flat FS export (no layers/history)
    docker import mycontainer.tar myimage:flat         # import flat FS as image
    # export/import loses metadata. Use save/load to preserve layers & history.

## 🚀 Container Commands

### docker run --- The Swiss Army Knife

    # ── Basic patterns ──
    docker run nginx                                  # run, attach, Ctrl+C to stop
    docker run -d nginx                               # detached (background)
    docker run --name myweb nginx                     # give it a name
    docker run --rm nginx                             # auto-remove on stop
    docker run -it ubuntu bash                        # interactive + TTY

    # ── Port mapping (-p HOST:CONTAINER) ──
    docker run -p 8080:80 nginx                       # host 8080 → container 80
    docker run -p 3000:3000 -p 9229:9229 myapp        # multiple ports
    docker run -p 127.0.0.1:8080:80 nginx             # only localhost
    docker run -P nginx                               # publish all EXPOSE'd ports to random host ports

    # ── Volumes (-v / --mount) ──
    docker run -v /host/path:/container/path nginx    # bind mount
    docker run -v myvolume:/data nginx                # named volume
    docker run -v /container/path nginx               # anonymous volume
    docker run --mount type=bind,src=/host,dst=/app nginx   # verbose syntax
    docker run --mount type=volume,src=mydata,dst=/data nginx

    # ── Environment variables (-e / --env-file) ──
    docker run -e MY_VAR=hello nginx
    docker run -e DB_HOST=db -e DB_PORT=5432 nginx    # multiple
    docker run --env-file .env.prod myapp             # load from file

    # ── Resource limits ──
    docker run --memory="256m" nginx                  # RAM limit
    docker run --memory="256m" --memory-swap="512m" nginx
    docker run --cpus="1.5" nginx                     # 1.5 CPU cores
    docker run --cpuset-cpus="0-2" nginx              # pin to CPU cores 0,1,2
    docker run --shm-size="256m" nginx                # /dev/shm size (important for browsers, DBs)

    # ── Restart policies ──
    docker run --restart=no nginx                     # default: never restart
    docker run --restart=always nginx                 # always restart (even after docker restart)
    docker run --restart=unless-stopped nginx         # restart unless explicitly stopped
    docker run --restart=on-failure nginx             # restart only on non-zero exit
    docker run --restart=on-failure:5 nginx           # max 5 retries

    # ── Other useful flags ──
    docker run --init nginx                           # use tini init (handles zombie processes)
    docker run --dns 8.8.8.8 nginx                    # custom DNS
    docker run --add-host myhost:192.168.1.5 nginx    # add /etc/hosts entry
    docker run -h myhostname nginx                    # set hostname
    docker run --privileged nginx                     # full host access (⚠️ security risk)
    docker run --cap-add=NET_ADMIN nginx              # add specific capability
    docker run --cap-drop=ALL nginx                   # drop all, add back selectively
    docker run --read-only nginx                      # read-only root filesystem
    docker run --tmpfs /tmp:rw,noexec,nosuid nginx    # tmpfs mount
    docker run -u 1000:1000 nginx                     # run as specific user:group
    docker run --log-driver json-file --log-opt max-size=10m nginx

::: note
**💡 The all-in-one example:**\
`docker run -d --name myapp --restart=unless-stopped -p 3000:3000 -v $(pwd):/app -e NODE_ENV=prod --memory="512m" --cpus="1" myapp:latest`
:::

### Lifecycle Management

    # ── Starting & Stopping ──
    docker start myweb                                # start a stopped container
    docker stop myweb                                 # graceful stop (SIGTERM → wait → SIGKILL)
    docker stop -t 30 myweb                           # wait 30s before SIGKILL
    docker kill myweb                                 # force kill (SIGKILL immediately)
    docker kill -s SIGTERM myweb                      # kill with specific signal
    docker restart myweb                              # stop + start
    docker restart -t 5 myweb                         # with timeout
    docker pause myweb                                # freeze (SIGSTOP)
    docker unpause myweb                              # unfreeze (SIGCONT)

    # ── Removing ──
    docker rm myweb                                   # remove stopped container
    docker rm -f myweb                                # force remove (running or not)
    docker rm -v myweb                                # also remove anonymous volumes
    docker rm $(docker ps -aq)                        # remove ALL containers
    docker rm $(docker ps -aq -f status=exited)       # remove all stopped
    docker container prune                            # remove all stopped containers
    docker container prune --filter "until=24h"       # stopped > 24h ago

    # ── Rename ──
    docker rename myweb mynewweb

### Executing Commands in Running Containers

    docker exec myweb ls /app                         # run single command
    docker exec -it myweb bash                        # interactive shell
    docker exec -it myweb sh                          # for alpine (no bash)
    docker exec -e VAR=value myweb env                # pass env var
    docker exec -u root myweb whoami                  # run as different user
    docker exec -w /tmp myweb pwd                     # run from specific working dir

### Attaching & Waiting

    docker attach myweb                               # attach to container's stdio
    docker attach --no-stdin myweb                    # no stdin
    docker wait myweb                                 # block until container exits, prints exit code

### Listing & Formatting

    docker ps                                         # running containers
    docker ps -a                                      # all (including stopped)
    docker ps -q                                      # just IDs
    docker ps -s                                      # include size info
    docker ps -l                                      # latest created (any state)
    docker ps -n 5                                    # last 5 containers
    docker ps -f "name=myweb"                         # filter by name
    docker ps -f "status=exited"                      # filter by status
    docker ps -f "label=env=prod"                     # filter by label

    # Custom format (Go templates)
    docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
    docker ps --format "{{.ID}}: {{.Command}}"        # custom columns

### Logs

    docker logs myweb                                 # all logs
    docker logs -f myweb                              # follow (like tail -f)
    docker logs --tail 100 myweb                      # last 100 lines
    docker logs --since 30m myweb                     # last 30 minutes
    docker logs --until 2024-01-01T00:00:00 myweb     # until timestamp
    docker logs -t myweb                              # include timestamps
    docker logs --details myweb                       # show extra log attributes

### Inspection & Monitoring

    docker inspect myweb                              # full JSON config
    docker inspect -f '{{.NetworkSettings.IPAddress}}' myweb
    docker inspect -f '{{.State.Status}}' myweb
    docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myweb

    docker top myweb                                  # running processes inside container
    docker stats                                      # live CPU/MEM/IO for all containers
    docker stats --no-stream                         # one-shot snapshot
    docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}" myweb

    docker port myweb                                 # show port mappings
    docker diff myweb                                 # files changed since container started

    docker events                                     # real-time Docker events stream
    docker events --filter "type=container"           # only container events
    docker events --filter "event=start" --filter "event=stop"

### Copying Files (host ↔ container)

    docker cp myfile.txt myweb:/app/file.txt          # host → container
    docker cp myweb:/app/log.txt ./log.txt            # container → host
    docker cp myweb:/app/data/ ./data/                # copy entire directory

### Update Running Container Config

    # Update resource limits, restart policy on the fly
    docker update --memory="512m" --cpus="2" myweb
    docker update --restart=always myweb
    # ⚠️ Cannot update ports, volumes, or env vars on a running container.
    #    For those, stop → create new → start.
:::

::: {#compose .section .section}
## 🐳 Docker Compose --- Complete Reference

For multi-container apps. Define everything in `docker-compose.yml` (or
`compose.yaml`).

### Complete compose.yaml Example

    name: myproject                     # project name (v2.23+)

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
            condition: service_healthy   # wait for healthy, not just started
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
          interval: 30s
          timeout: 5s
          retries: 3
        restart: unless-stopped
        volumes:
          - ./frontend/src:/app/src      # live reload in dev
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
          - full                          # only start with --profile full

      worker:
        build: ./worker
        command: python worker.py
        depends_on:
          - api
          - db
        deploy:
          replicas: 2                    # run 2 instances (swarm only in v3)

      db:
        image: postgres:16-alpine
        environment:
          POSTGRES_USER: myapp
          POSTGRES_PASSWORD: ${DB_PASSWORD}   # from .env file or shell
          POSTGRES_DB: myapp
        volumes:
          - pgdata:/var/lib/postgresql/data
          - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # auto-run on first start
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
        internal: true                    # no external connectivity

### All Compose CLI Commands

    # ── Core commands ──
    docker compose up                                   # start all services (foreground)
    docker compose up -d                                # background
    docker compose up -d --build                        # rebuild images then start
    docker compose up -d --force-recreate               # recreate containers even if unchanged
    docker compose up -d --scale web=3                  # scale (v2 compose)
    docker compose up --profile full                    # include profiled services
    docker compose up web api                           # start specific services only

    docker compose down                                 # stop and remove containers & networks
    docker compose down -v                              # also remove volumes (⚠️ deletes data!)
    docker compose down --rmi all                       # also remove images
    docker compose down --remove-orphans                # remove containers not in current compose file

    # ── Lifecycle ──
    docker compose start                                # start stopped services
    docker compose stop                                 # stop (don't remove)
    docker compose stop -t 30                           # with timeout
    docker compose restart                              # restart all
    docker compose restart web                          # restart one service
    docker compose pause                                # freeze all
    docker compose unpause                              # unfreeze
    docker compose kill                                 # force kill all

    # ── Viewing ──
    docker compose ps                                   # list compose services
    docker compose ps -a                                # include stopped
    docker compose ps --format json                     # JSON output
    docker compose images                               # images used by compose services
    docker compose top                                  # processes running in each service
    docker compose port web 3000                        # show published port

    # ── Logs ──
    docker compose logs                                 # all services
    docker compose logs -f web                          # follow specific service
    docker compose logs --tail=100 web                  # last 100 lines
    docker compose logs --since 10m api                 # time-based

    # ── Exec ──
    docker compose exec web bash                        # shell in running service
    docker compose exec -T web ls /app                  # no TTY (for scripts)
    docker compose exec -u root web bash                # as different user
    docker compose exec -e FOO=bar web env              # with env var
    docker compose run web npm test                     # run one-off command (new container)
    docker compose run --rm web npm test                # auto-remove after
    docker compose run --service-ports web bash         # with port mappings

    # ── Build ──
    docker compose build                                # build all services
    docker compose build --no-cache web                 # force rebuild one
    docker compose build --build-arg ENV=prod web       # with build args

    # ── Config & Validation ──
    docker compose config                               # show resolved config (with variable substitution)
    docker compose config --services                    # list service names only
    docker compose config --volumes                     # list volumes

    # ── Other ──
    docker compose pull                                 # pull latest images for all services
    docker compose push                                 # push built images to registry
    docker compose cp myfile.txt web:/app/file.txt      # copy files
    docker compose events                               # real-time events for project
    docker compose watch                                # watch for file changes and sync (v2.22+)

### Compose Environment Variables

    # Priority (highest to lowest):
    # 1. Shell environment variables
    # 2. --env-file on command line
    # 3. env_file in compose file (service-level overrides project-level)
    # 4. environment: in compose file
    # 5. .env file (in project directory)

    # .env file example:
    DB_PASSWORD=supersecret
    API_KEY=sk-abc123
    COMPOSE_PROJECT_NAME=myproject

    # Usage in compose.yaml:
    environment:
      - PASSWORD=${DB_PASSWORD}
      - KEY=${API_KEY:-default_value}        # default if not set
      - REQUIRED=${API_KEY:?err}             # error if not set
:::

::: {#volumes .section .section}
## 💾 Volumes & Storage

### Volume Types


  Type               Managed By   Location                     Use Case
  ------------------ ------------ ---------------------------- -----------------------------------------------------
  Named Volume       Docker       `/var/lib/docker/volumes/`   Production data persistence
  Bind Mount         You          Any host path                Development (live reload, sharing config)
  Anonymous Volume   Docker       Random dir                   Ephemeral (deleted with container unless `-v` flag)
  tmpfs              Kernel       RAM                          Sensitive temp data, no disk writes
:::

### Volume Commands

    # ── Create & Inspect ──
    docker volume create mydata
    docker volume create --driver local --opt type=nfs --opt device=:/nfs/path nfsvol
    docker volume ls
    docker volume inspect mydata                       # see mountpoint, labels, driver
    docker volume ls -f dangling=true                  # unused volumes only

    # ── Remove ──
    docker volume rm mydata
    docker volume rm $(docker volume ls -q)            # remove ALL (⚠️)
    docker volume prune                                # remove all unused local volumes
    docker volume prune -a                             # all unused (including named)
    docker volume prune --filter "label!=keep"

### Usage Patterns

    # Named volume (Docker manages the path)
    docker run -v mydata:/app/data myapp

    # Bind mount (you pick the host path)
    docker run -v /home/samir/project:/app myapp
    docker run -v $(pwd):/app myapp                    # current directory
    docker run -v /host/config:/app/config:ro myapp    # read-only bind mount

    # Anonymous volume (auto-generated)
    docker run -v /app/data myapp                      # no host path given

    # tmpfs (in-memory)
    docker run --tmpfs /tmp:rw,size=64m myapp
    docker run --tmpfs /run:rw,noexec,nosuid,size=32m myapp

    # --mount (verbose, recommended for production)
    docker run --mount type=bind,src=$(pwd)/config,dst=/app/config,readonly myapp
    docker run --mount type=volume,src=mydata,dst=/data,volume-driver=local myapp
    docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=64m myapp

### Backup & Restore Volumes

    # Backup a volume
    docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
      tar czf /backup/mydata-backup.tar.gz -C /data .

    # Restore a volume
    docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
      tar xzf /backup/mydata-backup.tar.gz -C /data
:::

::: {#networks .section .section}
## 🌐 Networks

### Network Drivers


  Driver      Behavior                                                              Use Case
  ----------- --------------------------------------------------------------------- --------------------------------------
  `bridge`    Default. Containers on same bridge can talk. DNS by container name.   Single-host apps
  `host`      Container shares host\'s network stack directly.                      Max performance, no isolation needed
  `none`      No networking at all.                                                 Security isolation
  `overlay`   Multi-host networking (Swarm).                                        Distributed apps across nodes
  `macvlan`   Container gets its own MAC/IP on physical network.                    Legacy apps expecting real IPs
  `ipvlan`    Like macvlan but shares MAC. Lower overhead.                          When MAC limits are a problem
:::

### Network Commands

    # ── Create & Manage ──
    docker network create mynet
    docker network create --driver bridge --subnet=172.28.0.0/16 --gateway=172.28.0.1 mynet
    docker network create --internal mynet              # no external connectivity
    docker network create --attachable mynet            # for overlay: allow standalone containers to attach
    docker network ls
    docker network ls -f driver=bridge
    docker network inspect mynet
    docker network rm mynet
    docker network prune                                # remove all unused

    # ── Connect & Disconnect ──
    docker network connect mynet myweb
    docker network connect --ip 172.28.0.100 mynet myweb
    docker network connect --alias db mynet myweb       # extra DNS alias
    docker network disconnect mynet myweb

    # ── Using Networks ──
    docker run --network mynet nginx
    docker run --network host nginx                     # share host network
    docker run --network none nginx                     # no networking

    # ── DNS: containers on same network resolve by name ──
    docker run -d --name db --network mynet postgres
    docker run -d --name web --network mynet myapp
    docker exec web ping db                             # ← resolves to db container's IP
:::

::: {#registry .section .section}
## ☁️ Registry Operations (Push, Pull, Login)

### Docker Hub

    # ── Login/Logout ──
    docker login                                        # interactive
    docker login -u samir                               # specify username
    docker login --password-stdin < password.txt        # CI/CD (stdin)
    docker logout

    # ── Push/Pull ──
    docker pull nginx:latest
    docker pull ubuntu:22.04                            # specific version
    docker pull python:3.12-slim                        # slim variant
    docker pull alpine                                  # :latest is default
    docker pull --all-tags samir/myapp                  # pull all tags
    docker pull --platform linux/amd64 nginx            # specific architecture

    docker push samir/myapp:v1.0
    docker push --all-tags samir/myapp                  # push all local tags

    # ── Search ──
    docker search nginx
    docker search --limit 25 python
    docker search --filter is-official=true ubuntu
    docker search --filter stars=100 node

### Other Registries

    # ── GitHub Container Registry (ghcr.io) ──
    echo $CR_PAT | docker login ghcr.io -u samir --password-stdin
    docker tag myapp ghcr.io/samir/myapp:v1.0
    docker push ghcr.io/samir/myapp:v1.0

    # ── AWS ECR ──
    aws ecr get-login-password --region us-east-1 | \
      docker login -u AWS --password-stdin ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
    docker tag myapp:latest ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
    docker push ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

    # ── Google Artifact Registry ──
    gcloud auth configure-docker us-central1-docker.pkg.dev
    docker push us-central1-docker.pkg.dev/PROJECT/repo/myapp:v1.0

    # ── Azure Container Registry (ACR) ──
    az acr login --name myregistry
    docker push myregistry.azurecr.io/myapp:v1.0

    # ── Self-hosted / Private Registry ──
    docker login registry.example.com
    docker tag myapp registry.example.com/myapp:v1.0
    docker push registry.example.com/myapp:v1.0

### Image Transfer Without Registry

    # Save → scp → load (offline / air-gapped)
    docker save myapp:latest -o myapp.tar
    scp myapp.tar user@server:/tmp/
    ssh user@server "docker load -i /tmp/myapp.tar"

    # Pipe directly (one-liner via SSH)
    docker save myapp:latest | gzip | ssh user@server "gunzip | docker load"
:::

::: {#resources .section .section}
## ⚙️ Resource Limits & Constraints

::: note
Resource limits prevent one container from hogging the entire machine.
:::

    # ── Memory ──
    docker run --memory="512m" nginx                    # hard limit
    docker run --memory="512m" --memory-reservation="256m" nginx  # soft limit
    docker run --memory="512m" --memory-swap="1g" nginx # total (mem + swap) limit
    docker run --memory="512m" --memory-swap="0" nginx  # disable swap
    docker run --memory-swappiness=10 nginx             # 0-100, lower = less swap
    docker run --oom-kill-disable nginx                 # don't kill on OOM (⚠️ risky)

    # ── CPU ──
    docker run --cpus="1.5" nginx                       # max 1.5 cores
    docker run --cpus=".5" nginx                        # half a core
    docker run --cpu-shares=512 nginx                   # relative weight (default 1024)
    docker run --cpuset-cpus="0-3" nginx                # pin to cores 0,1,2,3
    docker run --cpuset-cpus="0,2" nginx                # pin to cores 0 and 2
    docker run --cpu-period=100000 --cpu-quota=50000 nginx  # 50% of a CPU

    # ── Disk I/O ──
    docker run --blkio-weight=500 nginx                 # relative weight (10-1000)
    docker run --device-read-bps=/dev/sda:1mb nginx     # read limit
    docker run --device-write-bps=/dev/sda:1mb nginx    # write limit

    # ── Other limits ──
    docker run --pids-limit=100 nginx                   # max processes
    docker run --ulimit nofile=1024:2048 nginx          # file descriptor limit (soft:hard)
    docker run --shm-size="256m" nginx                  # /dev/shm size

    # ── Update running container ──
    docker update --memory="1g" --cpus="2" myweb
:::

::: {#buildx .section .section}
## 🏗️ Docker Buildx --- Multi-Platform Builds

    # ── Create a builder ──
    docker buildx create --name mybuilder --use
    docker buildx create --name mybuilder --driver docker-container --use  # advanced driver
    docker buildx ls
    docker buildx inspect mybuilder
    docker buildx use mybuilder

    # ── Multi-platform build ──
    docker buildx build --platform linux/amd64,linux/arm64 -t samir/myapp:latest .
    docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t samir/myapp:v1.0 .

    # ── Build + Push (all-in-one) ──
    docker buildx build --platform linux/amd64,linux/arm64 \
      -t samir/myapp:latest --push .

    # ── Build without pushing ──
    docker buildx build --platform linux/amd64,linux/arm64 \
      -t samir/myapp:latest -o type=docker .            # load into local docker

    # ── Cache ──
    docker buildx build --cache-to type=registry,ref=samir/myapp:cache \
      --cache-from type=registry,ref=samir/myapp:cache -t samir/myapp:latest .

    # ── Cleanup ──
    docker buildx rm mybuilder
    docker buildx prune                                 # clean build cache
:::

::: {#swarm .section .section}
## 🐝 Docker Swarm (Basics)

    # ── Init & Join ──
    docker swarm init                                   # create swarm (on manager)
    docker swarm init --advertise-addr 192.168.1.10
    docker swarm join --token SWMTKN-... 192.168.1.10:2377  # worker joins
    docker swarm join-token manager                     # get manager join token
    docker swarm join-token worker                      # get worker join token
    docker swarm leave                                  # leave swarm
    docker swarm leave --force                          # force leave (manager)

    # ── Nodes ──
    docker node ls
    docker node inspect node1
    docker node update --availability drain node1       # take node out of service
    docker node update --label-add env=prod node1
    docker node promote node2                           # worker → manager
    docker node demote node1                            # manager → worker

    # ── Services (swarm-level containers) ──
    docker service create --name web --replicas 3 -p 80:80 nginx
    docker service ls
    docker service ps web
    docker service inspect web
    docker service logs -f web
    docker service scale web=5
    docker service update --image nginx:alpine web      # rolling update
    docker service update --force web                   # force re-deploy
    docker service rollback web
    docker service rm web

    # ── Stacks (compose in swarm) ──
    docker stack deploy -c docker-compose.yml mystack
    docker stack ls
    docker stack services mystack
    docker stack ps mystack
    docker stack rm mystack
:::

::: {#cleanup .section .section}
## 🧹 Cleanup --- Free Up Disk Space

    # ── Check disk usage ──
    docker system df                                    # summary
    docker system df -v                                 # verbose (per-image, per-container, per-volume)

    # ── Prune by category ──
    docker container prune                              # stopped containers
    docker image prune                                  # dangling images (<none>)
    docker image prune -a                               # ALL unused images
    docker volume prune                                 # unused volumes
    docker network prune                                # unused networks
    docker builder prune                                # build cache

    # ── Prune with filters ──
    docker container prune --filter "until=24h"
    docker image prune -a --filter "until=7d"
    docker image prune -a --filter "label=env=dev"

    # ── Nuclear options ──
    docker system prune                                 # containers + networks + dangling images + build cache
    docker system prune -a                              # + ALL unused images
    docker system prune -a --volumes                    # + volumes (⚠️⚠️ data loss!)
    docker system prune -a -f                           # skip confirmation prompt

    # ── Manual all-in-one (if you need exact control) ──
    docker stop $(docker ps -aq)                        # stop everything
    docker rm $(docker ps -aq)                          # remove everything
    docker rmi $(docker images -q)                      # remove every image
    docker volume rm $(docker volume ls -q)             # remove every volume
    docker network rm $(docker network ls -q)           # remove every network

## 🔍 Debugging & Troubleshooting

### Common Debug Workflows

    # "Why won't my container start?"
    docker ps -a                          # is it exited? crashed?
    docker logs mycontainer               # check the output
    docker logs --tail 50 mycontainer     # last 50 lines
    docker inspect mycontainer            # full config (State.ExitCode, State.Error)

    # "Get a shell inside to poke around"
    docker exec -it mycontainer bash
    docker exec -it mycontainer sh         # alpine images

    # "Container exits immediately — debug it"
    docker run -it --entrypoint /bin/bash broken-image
    docker run -it --entrypoint /bin/sh broken-image
    docker commit broken-container debug-image && docker run -it debug-image bash

    # "What's using all my disk space?"
    docker system df -v                   # detailed breakdown
    du -sh /var/lib/docker/               # direct disk usage

    # "Is my port mapping working?"
    docker port mycontainer
    ss -tlnp | grep                 # check if host is listening
    curl localhost:PORT                   # quick test

    # "What processes are running inside?"
    docker top mycontainer

    # "Live resource usage"
    docker stats                          # CPU, memory, I/O, pids
    docker stats --no-stream              # one-shot

    # "What files changed inside?"
    docker diff mycontainer               # A=added, C=changed, D=deleted

    # "See Docker events in real-time"
    docker events                         # all events
    docker events --filter type=container --filter event=die  # only container deaths

    # "System info"
    docker info                           # daemon config, storage driver, registries
    docker version                        # client + server version details

### Common Issues & Fixes


  Symptom                           Likely Cause                      Fix
  --------------------------------- --------------------------------- ----------------------------------------------------------
  Container exits immediately       No foreground process             Make sure CMD/ENTRYPOINT is a foreground process
  Port already in use               Another process on host port      `lsof -i :PORT`, change host port mapping
  \"Permission denied\" on volume   UID/GID mismatch                  `docker run -u $(id -u):$(id -g)`
  Can\'t connect to localhost       Container has its own localhost   Use `host.docker.internal` (Mac/Win) or `--network host`
  Container can\'t reach internet   DNS or network issue              `docker run --dns 8.8.8.8`, restart docker daemon
  OOM killed                        Memory limit too low              `docker run --memory="1g"` or check app for memory leaks
  Build is slow                     Bad layer ordering                COPY package.json → RUN install → COPY rest
  Image too large                   No multi-stage build              Use multi-stage builds, alpine base images

## 📝 .dockerignore --- Exclude Files from Build Context

    # .dockerignore — placed next to Dockerfile
    # Speeds up builds and keeps secrets out of images

    # Version control
    .git
    .gitignore
    .svn

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
    *.so
    *.dylib

    # IDE & Editor
    .vscode
    .idea
    *.swp
    *.swo
    *~

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
    temp/
    *.tmp

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

## ⚡ Quick Reference Card



    # ── Images ──────────────────────────────────
    docker pull nginx:latest                # pull image
    docker build -t myapp:v1 .              # build from Dockerfile
    docker images                           # list images
    docker rmi myapp:v1                     # remove image

    # ── Containers ─────────────────────────────
    docker run -d --name app -p 3000:3000 myapp   # start
    docker ps -a                                  # list all
    docker stop app && docker rm app               # stop + remove
    docker exec -it app bash                       # shell inside
    docker logs -f app                             # follow logs

    # ── Compose ────────────────────────────────
    docker compose up -d                      # start project
    docker compose down                        # stop project
    docker compose logs -f web                # service logs
    docker compose exec web bash              # shell in service

    # ── Cleanup ────────────────────────────────
    docker system prune -a                    # clean everything unused
    docker system df                          # disk usage

    # ── Debug ──────────────────────────────────
    docker inspect container_name             # full config
    docker stats                              # resource usage
    docker events                             # event stream
