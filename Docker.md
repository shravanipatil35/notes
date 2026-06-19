# Docker — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Docker vs Virtual Machines](#2-docker-vs-virtual-machines)
3. [Docker Architecture](#3-docker-architecture)
4. [Installation](#4-installation)
5. [Core Concepts](#5-core-concepts)
6. [Basic Docker Commands](#6-basic-docker-commands)
7. [Dockerfile Deep Dive](#7-dockerfile-deep-dive)
8. [Image Layers & Caching](#8-image-layers--caching)
9. [Multi-Stage Builds](#9-multi-stage-builds)
10. [Volumes & Bind Mounts](#10-volumes--bind-mounts)
11. [Docker Networking](#11-docker-networking)
12. [Docker Compose](#12-docker-compose)
13. [Environment Variables & Secrets](#13-environment-variables--secrets)
14. [Docker Registry & Docker Hub](#14-docker-registry--docker-hub)
15. [Resource Constraints](#15-resource-constraints)
16. [Docker Security Best Practices](#16-docker-security-best-practices)
17. [Logging & Monitoring](#17-logging--monitoring)
18. [Docker Swarm](#18-docker-swarm)
19. [Docker vs Kubernetes](#19-docker-vs-kubernetes)
20. [Advanced Internals](#20-advanced-internals)
21. [Docker in CI/CD](#21-docker-in-cicd)
22. [Troubleshooting & Debugging](#22-troubleshooting--debugging)
23. [Best Practices Summary](#23-best-practices-summary)
24. [Docker Cheat Sheet](#24-docker-cheat-sheet)
25. [Interview Questions & Answers](#25-interview-questions--answers)

---

## 1. Introduction

**Docker** is an open-source platform that uses **OS-level virtualization** to package an application and all its dependencies (code, runtime, libraries, system tools, settings) into a single unit called a **container**. Containers run consistently across any environment — your laptop, a test server, or production — solving the classic "it works on my machine" problem.

Docker was released in 2013 by Solomon Hykes (dotCloud). It's built on Linux kernel features like **namespaces** and **cgroups**, and uses a **union filesystem** for efficient image layering.

**Why Docker?**
- Consistency across dev/test/prod environments
- Lightweight compared to VMs (shares host OS kernel)
- Fast startup (seconds, not minutes)
- Easy to version, share, and roll back (images are versioned artifacts)
- Enables microservices architecture
- Simplifies CI/CD pipelines

---

## 2. Docker vs Virtual Machines

| Aspect | Docker Containers | Virtual Machines |
|---|---|---|
| Virtualization level | OS-level (process isolation) | Hardware-level |
| OS | Shares host OS kernel | Each VM has full guest OS |
| Size | MBs | GBs |
| Boot time | Seconds | Minutes |
| Performance | Near-native | Overhead due to hypervisor |
| Isolation | Process-level (weaker) | Full isolation (stronger) |
| Portability | Highly portable (image-based) | Less portable (large images) |
| Use case | Microservices, CI/CD, scaling | Running different OS, strong isolation needs |

**Key insight:** A VM virtualizes hardware and runs a full OS via a **hypervisor** (e.g., VMware, KVM, Hyper-V). A container virtualizes the OS and runs as an isolated process on the host kernel, managed by a **container runtime** (e.g., containerd, runc).

---

## 3. Docker Architecture

Docker uses a **client-server architecture**:

```
┌─────────────┐     REST API (HTTP)     ┌─────────────────────┐
│ Docker CLI  │ ───────────────────────▶ │   Docker Daemon      │
│ (client)    │                          │   (dockerd)           │
└─────────────┘                          │                       │
                                          │  - Manages images     │
                                          │  - Manages containers │
                                          │  - Manages networks   │
                                          │  - Manages volumes    │
                                          └──────────┬────────────┘
                                                      │
                                          ┌───────────▼───────────┐
                                          │   containerd            │
                                          │  (container lifecycle)  │
                                          └───────────┬───────────┘
                                                      │
                                          ┌───────────▼───────────┐
                                          │      runc                │
                                          │ (OCI runtime — creates  │
                                          │  actual container via   │
                                          │  namespaces & cgroups)  │
                                          └────────────────────────┘
```

**Components:**
- **Docker Client (CLI)** — the `docker` command you type; sends commands to the daemon via REST API.
- **Docker Daemon (`dockerd`)** — background service that builds, runs, and manages containers, images, networks, volumes.
- **containerd** — a high-level container runtime that manages container lifecycle (start/stop/pause), image pulls, storage.
- **runc** — low-level OCI-compliant runtime that actually creates containers using Linux namespaces/cgroups.
- **Docker Registry** — stores and distributes images (Docker Hub is the default public registry; you can run private ones like Harbor, ECR, GCR, ACR).
- **Docker Objects** — Images, Containers, Networks, Volumes.

---

## 4. Installation

**Linux (Ubuntu example):**
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**Verify:**
```bash
docker --version
docker run hello-world
```

**Windows/Mac:** Use **Docker Desktop** (uses a lightweight Linux VM under the hood since the kernel features Docker needs are Linux-only).

**Post-install (Linux) — run docker without sudo:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 5. Core Concepts

| Term | Definition |
|---|---|
| **Image** | A read-only template with instructions for creating a container (app code + dependencies + runtime). Built in layers. |
| **Container** | A running (or stopped) instance of an image — an isolated, executable process. |
| **Dockerfile** | A text file with instructions to build an image. |
| **Registry** | A storage/distribution system for images (e.g., Docker Hub, ECR, GCR, private registry). |
| **Repository** | A collection of related images with different tags (e.g., `nginx:1.25`, `nginx:latest`). |
| **Volume** | Persistent storage managed by Docker, independent of container lifecycle. |
| **Network** | Virtual networking that lets containers communicate. |
| **Docker Compose** | Tool to define and run multi-container applications via YAML. |
| **Tag** | A label for an image version, e.g., `myapp:1.0`. |

---

## 6. Basic Docker Commands

### Images
```bash
docker pull nginx:latest          # download image from registry
docker images                     # list local images
docker rmi <image_id>             # remove an image
docker build -t myapp:1.0 .       # build image from Dockerfile in current dir
docker tag myapp:1.0 myrepo/myapp:1.0   # tag an image
docker push myrepo/myapp:1.0      # push to registry
docker history <image>            # show layer history
docker inspect <image>            # detailed metadata
```

### Containers
```bash
docker run -d --name web -p 8080:80 nginx     # run detached, map port
docker run -it ubuntu bash                    # run interactively
docker ps                                     # list running containers
docker ps -a                                  # list all containers (incl. stopped)
docker stop <container>                       # graceful stop (SIGTERM)
docker kill <container>                       # force stop (SIGKILL)
docker start <container>                      # start stopped container
docker restart <container>                    # restart
docker rm <container>                         # remove container
docker rm -f <container>                      # force remove (running)
docker logs <container>                       # view logs
docker logs -f <container>                    # follow logs (tail)
docker exec -it <container> bash              # shell into running container
docker inspect <container>                    # detailed JSON info
docker stats                                  # live resource usage
docker top <container>                        # processes inside container
docker cp file.txt <container>:/app/          # copy file into container
docker diff <container>                       # show filesystem changes
```

### System / Cleanup
```bash
docker system df              # disk usage
docker system prune           # remove unused data (stopped containers, dangling images, build cache)
docker system prune -a        # also remove unused images (not just dangling)
docker volume prune           # remove unused volumes
docker network prune          # remove unused networks
docker container prune        # remove all stopped containers
```

**Common `docker run` flags:**
| Flag | Purpose |
|---|---|
| `-d` | detached mode (background) |
| `-it` | interactive + TTY |
| `-p host:container` | port mapping |
| `-v host:container` | volume/bind mount |
| `--name` | assign container name |
| `-e KEY=VALUE` | set environment variable |
| `--rm` | auto-remove container on exit |
| `--network` | connect to a specific network |
| `--restart` | restart policy (`no`, `always`, `on-failure`, `unless-stopped`) |
| `--memory` / `--cpus` | resource limits |

---

## 7. Dockerfile Deep Dive

A **Dockerfile** is a script of instructions to build an image.

```dockerfile
# Base image
FROM node:20-alpine

# Metadata
LABEL maintainer="you@example.com"

# Set working directory inside container
WORKDIR /app

# Copy dependency files first (for layer caching)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy rest of the app
COPY . .

# Set environment variable
ENV NODE_ENV=production

# Expose port (documentation only, doesn't actually publish)
EXPOSE 3000

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1

# Default command when container starts
CMD ["node", "server.js"]
```

### Key Instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image (must be first instruction) |
| `WORKDIR` | Sets working directory for subsequent instructions |
| `COPY` | Copies files from host into image |
| `ADD` | Like COPY, but also supports URL download & auto-extracts tar archives (prefer COPY unless you need these features) |
| `RUN` | Executes a command at **build time**, creates a new layer |
| `CMD` | Default command at **container runtime** (can be overridden by `docker run` args) |
| `ENTRYPOINT` | Fixed command at runtime (harder to override; often combined with CMD for default args) |
| `ENV` | Sets environment variable (persists in image & container) |
| `ARG` | Build-time-only variable (not available at runtime) |
| `EXPOSE` | Documents the port the app listens on |
| `VOLUME` | Creates a mount point for persistent/external storage |
| `USER` | Sets user for subsequent instructions/runtime (avoid running as root) |
| `LABEL` | Adds metadata key-value pairs |
| `HEALTHCHECK` | Defines how Docker tests container health |
| `ONBUILD` | Trigger instruction executed when this image is used as a base for another build |

### CMD vs ENTRYPOINT
```dockerfile
ENTRYPOINT ["python3"]
CMD ["app.py"]
# docker run myimage           -> runs: python3 app.py
# docker run myimage test.py   -> runs: python3 test.py (CMD overridden)
```
- `CMD` alone: easily overridden entirely by `docker run` arguments.
- `ENTRYPOINT` alone: arguments to `docker run` are **appended** to it.
- Combined: `ENTRYPOINT` is the fixed executable, `CMD` supplies default arguments that can be replaced.

### ARG vs ENV
```dockerfile
ARG VERSION=1.0          # only available during build
ENV APP_VERSION=$VERSION # bake into image, available at runtime too
```
```bash
docker build --build-arg VERSION=2.0 -t myapp .
```

### .dockerignore
Just like `.gitignore` — exclude files from the build context (speeds up builds, avoids leaking secrets):
```
node_modules
.git
*.log
.env
Dockerfile
README.md
```

---

## 8. Image Layers & Caching

Every instruction in a Dockerfile (mainly `RUN`, `COPY`, `ADD`) creates a new **read-only layer**. Layers are cached and reused across builds — Docker only rebuilds from the first changed layer onward.

**Layer caching strategy (important for interviews):**
```dockerfile
# BAD — any code change invalidates the dependency install cache
COPY . .
RUN npm install

# GOOD — dependency layer only rebuilds when package.json changes
COPY package*.json ./
RUN npm install
COPY . .
```

**Union Filesystem (UnionFS / OverlayFS):** Docker stacks layers on top of each other and presents them as a single unified filesystem. The top-most layer of a running container is a thin **writable layer**; all layers below are read-only and shared between containers using the same image — this is what makes containers lightweight and fast to start.

```bash
docker image inspect myimage --format '{{.RootFS.Layers}}'
docker history myimage   # see each layer's size and command
```

To minimize image size:
- Use minimal base images (`alpine`, `distroless`)
- Combine `RUN` commands with `&&` to reduce layer count
- Clean up package manager caches in the same `RUN` layer
- Use multi-stage builds (next section)

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

---

## 9. Multi-Stage Builds

Used to keep final images small by separating **build-time** dependencies from **runtime** dependencies.

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# Stage 2: Runtime (only the binary is copied — no Go toolchain in final image)
FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/myapp .
ENTRYPOINT ["./myapp"]
```

**Benefits:**
- Drastically smaller final images (e.g., 900MB → 15MB)
- No build tools/source code/compilers in production image → smaller attack surface
- One Dockerfile, no need for separate build scripts

You can also copy from external images: `COPY --from=nginx:latest /etc/nginx/nginx.conf .`

---

## 10. Volumes & Bind Mounts

Containers are ephemeral — when removed, the writable layer is lost. To persist data, use:

| Type | Description | Managed by | Use case |
|---|---|---|---|
| **Volumes** | Stored in Docker-managed area (`/var/lib/docker/volumes/`) | Docker | Databases, persistent app data — **preferred** |
| **Bind mounts** | Maps a specific host path into the container | You (host filesystem) | Local dev, sharing source code/config |
| **tmpfs mounts** | Stored in host memory only, never written to disk | Docker (RAM) | Sensitive temp data |

```bash
# Named volume
docker volume create mydata
docker run -d -v mydata:/var/lib/mysql mysql

# Bind mount
docker run -d -v /home/user/app:/app node

# Newer --mount syntax (more explicit, recommended)
docker run -d --mount type=volume,source=mydata,target=/var/lib/mysql mysql
docker run -d --mount type=bind,source=/home/user/app,target=/app node

# tmpfs
docker run -d --mount type=tmpfs,target=/app/cache myapp

# Volume management
docker volume ls
docker volume inspect mydata
docker volume rm mydata
```

**Key interview point:** Volumes are the preferred mechanism because they're decoupled from the host filesystem structure, easier to back up/migrate, work on both Linux and Windows containers, and can be managed via Docker CLI/API. Bind mounts depend on host directory structure existing.

---

## 11. Docker Networking

Docker has several built-in network drivers:

| Driver | Description |
|---|---|
| **bridge** (default) | Private internal network on the host; containers get their own IP; used for standalone containers on a single host |
| **host** | Container shares the host's network namespace directly (no isolation, no port mapping needed) |
| **none** | No networking at all |
| **overlay** | Connects containers across multiple Docker hosts (used in Swarm/multi-host setups) |
| **macvlan** | Assigns a MAC address to a container, making it appear as a physical device on the network |

```bash
docker network ls
docker network create mynet
docker network create --driver bridge --subnet 172.20.0.0/16 mynet
docker run -d --network mynet --name app1 myapp
docker network inspect mynet
docker network connect mynet existing_container
docker network disconnect mynet existing_container
docker network rm mynet
```

**Default bridge vs user-defined bridge:**
- Default `bridge` network: containers can only reach each other by IP, not by name (no automatic DNS).
- **User-defined bridge networks** provide automatic **DNS resolution by container name** — this is why Compose-created networks let services talk to each other using service names as hostnames.

```bash
# On a user-defined network, this works:
docker run -d --network mynet --name db mysql
docker run -it --network mynet myapp ping db   # resolves by name
```

**Port publishing:**
```bash
docker run -p 8080:80 nginx        # host:container — bind to all host interfaces
docker run -p 127.0.0.1:8080:80 nginx   # bind only to localhost
docker run -P nginx                 # publish all EXPOSEd ports to random host ports
```

---

## 12. Docker Compose

Compose defines multi-container applications declaratively in YAML — essential for local dev environments and integration testing.

```yaml
# docker-compose.yml
version: "3.9"

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src
    networks:
      - app-net
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    networks:
      - app-net

volumes:
  db-data:

networks:
  app-net:
    driver: bridge
```

**Commands:**
```bash
docker compose up -d            # start all services in background
docker compose down             # stop & remove containers, networks
docker compose down -v          # also remove volumes
docker compose ps               # list services
docker compose logs -f web      # follow logs for a service
docker compose build            # rebuild images
docker compose exec web bash    # shell into a running service
docker compose restart web
docker compose config           # validate & view resolved config
```

**Notes:**
- `depends_on` controls **startup order only**, not readiness — use healthchecks for true "wait until ready" behavior.
- Compose automatically creates a network so services can reach each other by service name.
- `docker-compose` (old standalone binary) is deprecated in favor of `docker compose` (the integrated CLI plugin, V2).

---

## 13. Environment Variables & Secrets

```bash
docker run -e DB_HOST=localhost -e DB_PORT=5432 myapp
docker run --env-file .env myapp
```

```yaml
# .env file
DB_HOST=localhost
DB_PASS=supersecret
```

**Important — secrets should NOT be:**
- Hardcoded in Dockerfiles (`ENV PASSWORD=123` bakes it permanently into image layers/history — visible via `docker history`)
- Passed as build args for sensitive data (also cached in layer history)

**Better approaches:**
- **Docker Secrets** (Swarm mode) — encrypted, mounted as files in `/run/secrets/`, never in env vars or image layers
- **BuildKit secret mounts** for build-time secrets that don't persist in the image:
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm install
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=npmrc,src=$HOME/.npmrc .
```
- External secret managers (Vault, AWS Secrets Manager, Kubernetes Secrets) injected at runtime

---

## 14. Docker Registry & Docker Hub

- **Docker Hub** — default public registry (`docker.io`)
- **Private registries** — AWS ECR, Google GCR/Artifact Registry, Azure ACR, GitHub Container Registry (GHCR), self-hosted (Harbor, `registry:2` image)

```bash
docker login
docker login myregistry.example.com

docker tag myapp:1.0 myregistry.example.com/myapp:1.0
docker push myregistry.example.com/myapp:1.0
docker pull myregistry.example.com/myapp:1.0

# Run your own local registry
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp
docker push localhost:5000/myapp
```

**Image naming convention:**
```
[registry-host[:port]/]namespace/repository[:tag]
e.g. myregistry.example.com:5000/myteam/myapp:1.0
```
If no registry is specified, Docker Hub is assumed. If no tag is specified, `latest` is assumed (best practice: **never rely on `latest` in production** — always pin explicit versions for reproducibility).

---

## 15. Resource Constraints

```bash
docker run -d --memory="512m" --memory-swap="1g" myapp
docker run -d --cpus="1.5" myapp
docker run -d --cpu-shares=512 myapp        # relative weight vs other containers
docker run -d --memory="512m" --oom-kill-disable myapp   # careful: can cause host instability
```

In Compose:
```yaml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

Without limits, a single container can consume all host resources and starve others — always set limits in production/shared environments.

---

## 16. Docker Security Best Practices

1. **Don't run containers as root** — use `USER` in Dockerfile.
2. **Use minimal base images** (`alpine`, `distroless`, `scratch`) to reduce attack surface.
3. **Scan images for vulnerabilities**: `docker scout` (built-in), Trivy, Snyk, Clair.
4. **Use multi-stage builds** to exclude build tools/secrets from final image.
5. **Pin specific image versions/digests**, never blindly trust `latest`.
6. **Don't store secrets in images** (env vars, ARG, or layers — all inspectable via `docker history`/`docker inspect`).
7. **Limit container capabilities**: `docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE`.
8. **Use read-only filesystems where possible**: `docker run --read-only`.
9. **Avoid `--privileged` mode** unless absolutely necessary (it disables most isolation).
10. **Set resource limits** to prevent DoS via resource exhaustion.
11. **Use trusted/signed images** — Docker Content Trust (`DOCKER_CONTENT_TRUST=1`).
12. **Keep Docker Engine and base images patched/updated.**
13. **Use network segmentation** — don't put unrelated containers on the same network.
14. **Avoid mounting the Docker socket** (`/var/run/docker.sock`) into containers unless required — it grants root-equivalent host access.

```bash
docker scout cves myimage          # vulnerability scan
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE -p 80:80 myapp
docker run --read-only --tmpfs /tmp myapp
docker run --security-opt=no-new-privileges myapp
```

---

## 17. Logging & Monitoring

**Logging drivers** (configure which backend container logs go to):
```bash
docker run --log-driver=json-file --log-opt max-size=10m --log-opt max-file=3 myapp
```
Common drivers: `json-file` (default), `syslog`, `journald`, `fluentd`, `awslogs`, `gelf`, `splunk`.

```bash
docker logs -f --tail 100 mycontainer
docker logs --since 10m mycontainer
```

**Monitoring:**
```bash
docker stats                 # live CPU/mem/network/IO per container
docker events                # stream of real-time Docker events
docker inspect --format='{{.State.Health.Status}}' mycontainer
```

For production-grade monitoring: **cAdvisor**, **Prometheus + Grafana**, **Datadog**, **ELK/EFK stack** for centralized log aggregation.

---

## 18. Docker Swarm

Docker's **native orchestration** tool (built into the Docker Engine) for clustering multiple Docker hosts and running services at scale.

```bash
docker swarm init                                  # initialize a swarm (becomes manager)
docker swarm join-token worker                     # get token for workers to join
docker swarm join --token <token> <manager-ip>:2377

docker node ls                                      # list nodes in the swarm

docker service create --name web --replicas 3 -p 80:80 nginx
docker service ls
docker service scale web=5
docker service update --image nginx:1.26 web        # rolling update
docker service rm web

docker stack deploy -c docker-compose.yml mystack    # deploy a Compose file as a stack
docker stack services mystack
docker stack rm mystack
```

**Key Swarm concepts:**
- **Manager nodes** — maintain cluster state, schedule services (use Raft consensus)
- **Worker nodes** — execute tasks/containers
- **Services** — desired-state definition of tasks to run (replicated or global mode)
- **Tasks** — a single container instance of a service running on a node
- Built-in **load balancing** and **service discovery** via internal DNS
- **Overlay networks** connect containers across multiple hosts

---

## 19. Docker vs Kubernetes

| Aspect | Docker / Docker Swarm | Kubernetes |
|---|---|---|
| Complexity | Simple, easy to learn | Steep learning curve |
| Setup | Built into Docker Engine | Separate installation (kubeadm, managed services) |
| Scaling | Good for small-medium clusters | Built for large-scale, complex deployments |
| Auto-healing | Basic | Advanced (self-healing, auto-restart, rescheduling) |
| Load balancing | Built-in, simple | More configurable (Services, Ingress) |
| Ecosystem | Smaller | Huge ecosystem (Helm, Operators, CRDs, service mesh) |
| Config | docker-compose.yml / stack files | YAML manifests (Deployments, Pods, Services, etc.) |

**Note:** Kubernetes does **not** use Docker as its container runtime anymore (deprecated `dockershim` in 2022) — it uses **containerd** or **CRI-O** directly via the Container Runtime Interface (CRI). Docker is still widely used for **building** images and **local development**, while Kubernetes is used for **production orchestration**.

---

## 20. Advanced Internals

### Namespaces (isolation)
Linux kernel feature that isolates what a process can *see*:
- **PID** — process ID isolation (container sees its own process tree)
- **NET** — network stack isolation (own IP, ports, routing table)
- **MNT** — mount point isolation (own filesystem view)
- **UTS** — hostname/domain isolation
- **IPC** — inter-process communication isolation
- **USER** — user/group ID mapping isolation

### Control Groups — cgroups (resource limiting)
Kernel feature that limits and accounts for resource usage (CPU, memory, disk I/O, network) **per process group** — this is how `--memory` and `--cpus` flags are enforced.

### Union/Overlay Filesystem
Combines multiple directories (layers) into a single view. Docker's default storage driver on Linux is typically **overlay2**, which stacks:
- Lower layers (read-only image layers)
- Upper layer (writable container layer)
- Merged view (what the process actually sees)

### OCI (Open Container Initiative)
Industry standard specs that Docker complies with:
- **Image spec** — defines image format
- **Runtime spec** — defines how to run a container (implemented by `runc`)

This standardization is why images built with Docker can run on other OCI-compliant runtimes (containerd, CRI-O, Podman).

### Container lifecycle (under the hood)
```
docker run → dockerd receives request → containerd pulls image (if needed)
→ containerd creates container spec → runc creates namespaces & cgroups
→ runc execs the container process → containerd supervises it (shim process)
```

### BuildKit
Modern build engine (default since Docker 23+) offering: parallel layer builds, better caching, build secrets, SSH forwarding during build, and reduced build context overhead.
```bash
DOCKER_BUILDKIT=1 docker build .
# or enable permanently in daemon.json: {"features": {"buildkit": true}}
```

---

## 21. Docker in CI/CD

Typical pipeline pattern:
```yaml
# Example: GitHub Actions snippet
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Run tests inside container
  run: docker run --rm myapp:${{ github.sha }} npm test

- name: Push to registry
  run: |
    echo "${{ secrets.DOCKER_PASS }}" | docker login -u user --password-stdin
    docker tag myapp:${{ github.sha }} myrepo/myapp:latest
    docker push myrepo/myapp:latest
```

**Common practices:**
- Build once, promote the same image across environments (dev → staging → prod) rather than rebuilding per stage
- Tag images with commit SHA / build number for traceability, plus a movable tag like `latest` or `stable`
- Use layer caching (`--cache-from`) to speed up CI builds
- Scan images for vulnerabilities as a pipeline gate
- Use multi-stage builds to keep CI artifacts small

---

## 22. Troubleshooting & Debugging

```bash
docker logs <container>                     # check application logs first
docker inspect <container>                  # check config, mounts, network, env
docker exec -it <container> sh               # get a shell inside (use sh if bash unavailable, e.g. alpine)
docker events                                 # watch real-time daemon events
docker port <container>                      # check port mappings
docker top <container>                       # check running processes
docker stats <container>                     # check if hitting resource limits

# Container exits immediately?
docker run myimage                           # run in foreground (not -d) to see the error directly
docker logs <container>                      # check exit reason

# "No space left on device"
docker system df
docker system prune -a --volumes

# Networking issues
docker network inspect <network>
docker exec -it <container> ping <other_container_name>
```

**Common exit codes:**
| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General application error |
| 137 | Killed (SIGKILL) — often OOM killed |
| 139 | Segmentation fault |
| 143 | Graceful termination (SIGTERM) |

---

## 23. Best Practices Summary

- One process/concern per container
- Use small, official, minimal base images
- Always pin explicit image versions, never float on `latest` in prod
- Use multi-stage builds
- Order Dockerfile instructions to maximize cache hits (least-frequently-changed first)
- Use `.dockerignore`
- Never store secrets in images
- Run as non-root user
- Set health checks
- Set resource limits
- Use named volumes for persistent data
- Keep images stateless — externalize state to volumes/databases
- Use `docker compose` for local multi-service dev
- Clean up unused resources regularly (`docker system prune`)
- Log to stdout/stderr (let Docker/log drivers handle aggregation) rather than writing log files inside the container

---

## 24. Docker Cheat Sheet

```bash
# Images
docker build -t name:tag .
docker images
docker rmi <image>
docker pull <image>
docker push <image>
docker tag src target

# Containers
docker run [options] image
docker ps [-a]
docker start/stop/restart/rm <container>
docker exec -it <container> bash
docker logs [-f] <container>
docker inspect <container>
docker stats
docker cp src dest

# Volumes
docker volume create/ls/inspect/rm <name>

# Networks
docker network create/ls/inspect/rm <name>

# Compose
docker compose up -d / down / ps / logs -f / build / exec

# Swarm
docker swarm init/join
docker service create/ls/scale/update/rm
docker stack deploy/rm

# Cleanup
docker system prune -a --volumes
```

---

## 25. Interview Questions & Answers

**Q1: What is Docker and how is it different from a Virtual Machine?**
A: Docker is a containerization platform that packages an app with its dependencies into a portable container, using OS-level virtualization. Unlike VMs, containers share the host OS kernel rather than running a full guest OS each, making them far more lightweight, faster to start, and more resource-efficient — at the cost of slightly weaker isolation than a VM.

**Q2: What is the difference between an image and a container?**
A: An **image** is a read-only, immutable template (built from a Dockerfile, made of layers) describing how to create a container. A **container** is a running (or stopped) instance of that image, with its own writable layer on top.

**Q3: Explain the difference between CMD and ENTRYPOINT.**
A: `CMD` provides default arguments/command that can be fully overridden by anything passed to `docker run`. `ENTRYPOINT` defines a fixed executable that always runs; arguments passed to `docker run` are appended to it rather than replacing it. They're often combined: `ENTRYPOINT` as the fixed binary, `CMD` as default args.

**Q4: What is a multi-stage build and why use it?**
A: A Dockerfile technique using multiple `FROM` stages — one to build/compile the app (with all build tools) and a final minimal stage that copies only the compiled artifact. This drastically shrinks the final image and removes build tools/source/secrets from the production image, reducing both size and attack surface.

**Q5: How does Docker achieve isolation?**
A: Through Linux kernel **namespaces** (PID, NET, MNT, UTS, IPC, USER) for isolating what a process can see, and **cgroups** for limiting/accounting resource usage (CPU, memory, I/O). The **union filesystem** (typically overlay2) layers the image's read-only layers with a writable container layer.

**Q6: What happens when you run `docker run image`?**
A: Docker CLI sends the request to the daemon via REST API → daemon checks if the image exists locally, pulls it from the registry if not → containerd creates a container from the image → it generates an OCI runtime spec and hands off to runc → runc creates namespaces/cgroups and starts the container process → containerd's shim supervises the running process.

**Q7: What's the difference between `COPY` and `ADD`?**
A: Both copy files into the image. `ADD` has extra features: it can fetch remote URLs and auto-extract local tar archives. Best practice is to prefer `COPY` for clarity and predictability, and only use `ADD` when you specifically need its extra behaviors.

**Q8: How do you persist data in Docker?**
A: Using **volumes** (Docker-managed storage, preferred — portable, easy to back up, works across platforms) or **bind mounts** (maps a specific host directory into the container — useful for local dev). Without either, data in a container's writable layer is lost when the container is removed.

**Q9: How do containers communicate with each other?**
A: Via Docker networks. On a **user-defined bridge network**, Docker provides automatic DNS resolution, so containers can reach each other by container/service name. On the **default bridge network**, only IP-based communication works (no built-in DNS). For multi-host communication, **overlay networks** are used (e.g., in Swarm).

**Q10: What is the difference between `docker stop` and `docker kill`?**
A: `docker stop` sends SIGTERM (graceful shutdown, allows cleanup) and waits a grace period (default 10s) before sending SIGKILL if the process hasn't exited. `docker kill` sends SIGKILL immediately (forceful, no cleanup chance).

**Q11: How do you reduce Docker image size?**
A: Use minimal base images (alpine/distroless), multi-stage builds, combine RUN commands to reduce layers, clean up package manager caches within the same layer, use `.dockerignore` to avoid copying unnecessary files, and avoid installing unnecessary packages.

**Q12: What is Docker Compose used for?**
A: Defining and running multi-container applications declaratively via a YAML file (`docker-compose.yml`), specifying services, networks, volumes, and their relationships — primarily used for local development and testing of multi-service apps.

**Q13: What is the difference between Docker Swarm and Kubernetes?**
A: Both orchestrate containers across multiple hosts. Swarm is built into Docker Engine, simpler to set up and learn, suited for smaller-scale deployments. Kubernetes is more complex but far more powerful/extensible — offering advanced self-healing, auto-scaling, rolling updates, a massive ecosystem (Helm, Operators, service mesh), and is the de facto industry standard for large-scale production orchestration.

**Q14: How do you handle secrets in Docker?**
A: Never bake secrets into Dockerfile `ENV`/`ARG` (they persist in image layers/history). Instead use Docker Secrets (Swarm — mounted as files, encrypted), BuildKit secret mounts for build-time-only secrets, or external secret managers (Vault, AWS Secrets Manager) that inject secrets at runtime.

**Q15: What is the purpose of `.dockerignore`?**
A: Excludes files/directories from the build context sent to the Docker daemon — speeding up builds, reducing image size, and preventing accidental inclusion of sensitive files (like `.env`, `.git`).

**Q16: Explain Docker layer caching and how to optimize it.**
A: Each Dockerfile instruction (RUN/COPY/ADD) creates a cacheable layer. If an instruction and its inputs haven't changed since the last build, Docker reuses the cached layer instead of re-executing it. To optimize, order instructions from least-to-most frequently changing — e.g., copy dependency manifest files and install dependencies before copying full source code, so code changes don't invalidate the dependency-install cache.

**Q17: What is the difference between `docker exec` and `docker attach`?**
A: `docker exec` starts a **new process** inside a running container (e.g., opening a new shell) without affecting the container's main process. `docker attach` connects your terminal to the container's existing **main process** (PID 1) — risky because exiting can stop the container if it's not detached properly (Ctrl+P, Ctrl+Q to detach safely).

**Q18: What is a dangling image?**
A: An image layer that's not tagged and not referenced by any container — typically left behind after rebuilding an image with the same tag (the old layer loses its tag). Clean up with `docker image prune`.

**Q19: How would you debug a container that keeps crashing/restarting?**
A: Check `docker logs <container>` for the application error first; run it in the foreground (without `-d`) to see real-time output; check `docker inspect` for exit code and restart policy; check resource limits via `docker stats` (exit code 137 often indicates OOM kill); verify environment variables and config are correct; try `docker exec` if it stays up briefly, or override the entrypoint with a shell to poke around manually.

**Q20: What's the difference between `docker-compose down` and `docker-compose down -v`?**
A: `down` stops and removes containers and the network created by Compose, but **preserves named volumes** (and their data). `down -v` additionally removes those volumes — wiping persistent data — so it should be used carefully.

**Q21: Can two containers use the same port on the host?**
A: No — only one process can bind to a given host port at a time. You'd map them to different host ports (e.g., `-p 8081:80` and `-p 8082:80`), or put a reverse proxy/load balancer in front.

**Q22: What is the OCI and why does it matter?**
A: The Open Container Initiative defines open standards for container **image format** and **runtime behavior**. Because Docker images comply with the OCI image spec, they can be run by any OCI-compliant runtime (containerd, CRI-O, Podman) — not just Docker — ensuring portability across tools and platforms.

**Q23: How do you limit a container's resource usage?**
A: Using flags like `--memory`, `--memory-swap`, `--cpus`, `--cpu-shares` on `docker run`, or `deploy.resources.limits` in a Compose file — enforced under the hood via Linux **cgroups**.

**Q24: What's the difference between a bridge network and host network mode?**
A: **Bridge** (default) gives the container its own isolated network namespace with a private IP, requiring explicit port mapping to reach it from the host. **Host** mode removes that network isolation entirely — the container shares the host's network namespace directly, so it can bind to host ports without explicit mapping, but loses network isolation (and `-p` mappings have no effect).

**Q25: Why shouldn't you run containers as root?**
A: If an attacker breaks out of a containerized process running as root, and especially if combined with container escape vulnerabilities or excessive Linux capabilities, they have root-level access — increasing the blast radius of a compromise. Running as a non-root `USER` limits what a compromised process can do, following the principle of least privilege.

---

### Final tip for interviews
Be ready to **draw the architecture diagram** (client → daemon → containerd → runc) from memory, explain **namespaces vs cgroups** clearly, walk through a **Dockerfile** instruction by instruction, and explain **why multi-stage builds and layer caching matter** — these come up constantly. Also be ready to compare Docker Compose (single host, dev) vs Swarm/Kubernetes (multi-host, production orchestration).

---
*End of notes.*
