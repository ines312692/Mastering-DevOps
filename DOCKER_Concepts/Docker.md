# Docker Interview Questions and Answers

## Table of Contents

1. [Docker Fundamentals](#docker-fundamentals)
2. [Dockerfile Best Practices](#dockerfile-best-practices)
3. [Docker Networking](#docker-networking)
4. [Docker Volumes and Storage](#docker-volumes-and-storage)
5. [Docker Compose](#docker-compose)
6. [Docker Security](#docker-security)
7. [Troubleshooting and Debugging](#troubleshooting-and-debugging)

---

## Docker Fundamentals

### Q1: What is Docker and how does it differ from virtual machines?

**Answer:**

Docker is a containerization platform that packages applications with their dependencies into lightweight, portable containers.

| Docker Containers | Virtual Machines |
|-------------------|------------------|
| Share host OS kernel | Full OS per VM |
| Lightweight (MBs) | Heavy (GBs) |
| Start in seconds | Start in minutes |
| Less isolation | Strong isolation |
| Process-level virtualization | Hardware-level virtualization |

```
Virtual Machines:          Containers:
+-------+-------+          +-------+-------+
| App A | App B |          | App A | App B |
+-------+-------+          +-------+-------+
| Bins  | Bins  |          | Bins  | Bins  |
+-------+-------+          +-------+-------+
| Guest | Guest |          |   Container   |
|  OS   |  OS   |          |    Runtime    |
+-------+-------+          +---------------+
|   Hypervisor  |          |    Host OS    |
+---------------+          +---------------+
|   Host OS     |          |   Hardware    |
+---------------+          +---------------+
|   Hardware    |
+---------------+
```

---

### Q2: Explain Docker architecture components.

**Answer:**

```
+------------------+
|   Docker Client  |  (docker CLI)
+--------+---------+
         |
         | REST API
         v
+--------+---------+
|   Docker Daemon  |  (dockerd)
+--------+---------+
         |
    +----+----+
    |         |
    v         v
+-------+ +----------+
|Images | |Containers|
+-------+ +----------+
    |
    v
+----------+
| Registry |  (Docker Hub, ECR, etc.)
+----------+
```

**Components:**

| Component | Description |
|-----------|-------------|
| Docker Client | CLI tool to interact with Docker |
| Docker Daemon | Background service managing containers |
| Docker Images | Read-only templates for containers |
| Containers | Running instances of images |
| Registry | Storage for Docker images |
| Docker Network | Networking between containers |
| Docker Volume | Persistent data storage |

---

### Q3: What is the difference between `CMD` and `ENTRYPOINT`?

**Answer:**

| ENTRYPOINT | CMD |
|------------|-----|
| Defines the executable that always runs | Provides default arguments |
| Harder to override (need --entrypoint) | Easily overridden at runtime |
| Makes container behave like a binary | Flexible defaults |

```dockerfile
# Example 1: CMD only
FROM python:3.11
CMD ["python", "app.py"]
# Can override: docker run myimage python other.py

# Example 2: ENTRYPOINT only
FROM python:3.11
ENTRYPOINT ["python"]
# Always runs python: docker run myimage app.py

# Example 3: Combined (recommended)
FROM python:3.11
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
# Results in: python app.py --port 8080
# Override args: docker run myimage --port 3000
# Override entry: docker run --entrypoint /bin/bash myimage
```

**Shell vs Exec form:**

```dockerfile
# Exec form (preferred) - runs directly
CMD ["python", "app.py"]
ENTRYPOINT ["python", "app.py"]

# Shell form - runs in /bin/sh -c
CMD python app.py
ENTRYPOINT python app.py
```

---

### Q4: Explain Docker image layers and caching.

**Answer:**

Each Dockerfile instruction creates a layer. Layers are cached and reused.

```dockerfile
FROM python:3.11-slim          # Layer 1 (base image)
WORKDIR /app                   # Layer 2 (metadata)
COPY requirements.txt .        # Layer 3 (small file)
RUN pip install -r requirements.txt  # Layer 4 (cached if requirements unchanged)
COPY . .                       # Layer 5 (invalidated on any code change)
CMD ["python", "app.py"]       # Layer 6 (metadata)
```

**Cache invalidation rules:**
- Any change invalidates that layer AND all subsequent layers
- Order matters: put frequently changing instructions last
- Only ADD, COPY, and RUN create layers with size

**Best practices:**

```dockerfile
# BAD - cache invalidated on any file change
COPY . .
RUN pip install -r requirements.txt

# GOOD - dependencies cached separately
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

**View layers:**

```bash
docker history <image>
docker inspect <image>
```

---

### Q5: What happens when you run `docker run`?

**Answer:**

Step-by-step process:

1. Docker client sends request to daemon
2. Daemon checks if image exists locally
3. If not, pulls image from registry
4. Creates writable container layer on top of image
5. Allocates network interface and IP address
6. Sets up filesystem mounts (volumes, bind mounts)
7. Creates container process with namespaces and cgroups
8. Executes specified command (ENTRYPOINT + CMD)
9. Captures stdout/stderr

```bash
# Detailed breakdown
docker run -d \
  --name mycontainer \      # Container name
  -p 8080:80 \              # Port mapping host:container
  -v data:/app/data \       # Volume mount
  -e DB_HOST=localhost \    # Environment variable
  --network mynet \         # Network
  --restart unless-stopped \ # Restart policy
  myimage:latest            # Image to run
```

---

## Dockerfile Best Practices

### Q6: How do you optimize Docker image size?

**Answer:**

1. **Use minimal base images:**

```dockerfile
# Instead of
FROM python:3.11          # ~900MB

# Use
FROM python:3.11-slim     # ~150MB
FROM python:3.11-alpine   # ~50MB
```

2. **Multi-stage builds:**

```dockerfile
# Build stage
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Runtime stage (minimal image)
FROM openjdk:17-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

3. **Minimize layers and clean up:**

```dockerfile
# BAD - multiple layers, cache not cleaned
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# GOOD - single layer, cache cleaned
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

4. **Use .dockerignore:**

```dockerignore
.git
.gitignore
node_modules
*.md
Dockerfile
.dockerignore
.env
tests/
docs/
```

5. **Order instructions wisely:**

```dockerfile
# Least frequently changed first
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
# Most frequently changed last
COPY . .
CMD ["node", "server.js"]
```

---

### Q7: Explain Docker multi-stage builds in detail.

**Answer:**

Multi-stage builds use multiple FROM statements to separate build and runtime stages.

**Benefits:**
- Smaller final images (no build tools)
- Reduced attack surface
- Faster deployments
- Build dependencies not in production
- Single Dockerfile for entire build process

**Example - Node.js application:**

```dockerfile
# Stage 1: Dependencies
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

USER nextjs
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Example - Go application:**

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Final stage (scratch = empty image)
FROM scratch
COPY --from=builder /app/main /main
EXPOSE 8080
ENTRYPOINT ["/main"]
```

---

### Q8: What is Docker BuildKit and its benefits?

**Answer:**

BuildKit is the next-generation build engine for Docker.

**Enable BuildKit:**

```bash
# Environment variable
export DOCKER_BUILDKIT=1
docker build .

# Or in daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

**Benefits:**

1. **Parallel builds** - independent stages built simultaneously
2. **Better caching** - more intelligent cache management
3. **Build secrets** - mount secrets without leaking to image
4. **SSH forwarding** - access private repos during build
5. **Cache mounts** - persistent package manager caches

**BuildKit features:**

```dockerfile
# syntax=docker/dockerfile:1

# Cache mount for pip (persistent across builds)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache mount for apt
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl

# Secret mount (not stored in image layers)
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/file .

# SSH mount for private repos
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git

# Build with secrets
docker build --secret id=aws,src=$HOME/.aws/credentials .
```

---

### Q9: Explain Docker health checks.

**Answer:**

Health checks let Docker monitor container health and take action.

**Dockerfile HEALTHCHECK:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**Parameters:**

| Option | Default | Description |
|--------|---------|-------------|
| --interval | 30s | Time between checks |
| --timeout | 30s | Max time for check to complete |
| --start-period | 0s | Grace period before checks count |
| --retries | 3 | Consecutive failures before unhealthy |

**Health check types:**

```dockerfile
# HTTP check
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1

# TCP check
HEALTHCHECK CMD nc -z localhost 8080 || exit 1

# Command check
HEALTHCHECK CMD pg_isready -U postgres || exit 1

# Disable inherited health check
HEALTHCHECK NONE
```

**Docker Compose health check:**

```yaml
services:
  web:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**Check health status:**

```bash
docker ps                                    # Shows health status
docker inspect --format='{{.State.Health.Status}}' container-name
docker inspect --format='{{json .State.Health}}' container-name | jq
```

---

## Docker Networking

### Q10: Explain Docker networking modes.

**Answer:**

| Driver | Description | Use Case |
|--------|-------------|----------|
| bridge | Default isolated network | Single host containers |
| host | Uses host network directly | Performance, no isolation |
| none | No networking | Security, offline processing |
| overlay | Multi-host networking | Swarm, Kubernetes |
| macvlan | Assigns MAC address | Legacy apps needing layer 2 |

```bash
# List networks
docker network ls

# Create network
docker network create --driver bridge mynetwork

# Run container on network
docker run --network mynetwork --name app myimage

# Connect existing container
docker network connect mynetwork container-name

# Disconnect container
docker network disconnect mynetwork container-name

# Inspect network
docker network inspect mynetwork

# Remove network
docker network rm mynetwork
```

**Network examples:**

```bash
# Bridge (default)
docker run --network bridge myimage

# Host (container uses host's network stack)
docker run --network host nginx
# Access nginx at localhost:80

# None (no networking)
docker run --network none myimage

# Custom bridge with DNS
docker network create mynet
docker run --network mynet --name db postgres
docker run --network mynet --name app myapp
# app can reach db via hostname "db"
```

---

### Q11: How does DNS resolution work between containers?

**Answer:**

Containers on the same user-defined network can resolve each other by name.

```bash
# Create network
docker network create mynetwork

# Run containers
docker run -d --network mynetwork --name database postgres
docker run -d --network mynetwork --name api myapi

# From api container, can reach database by name
docker exec api ping database
docker exec api curl http://database:5432
```

**DNS configuration:**

```bash
# Custom DNS servers
docker run --dns 8.8.8.8 --dns 8.8.4.4 myimage

# Custom hostname
docker run --hostname myhost myimage

# Add hosts entry
docker run --add-host db.local:192.168.1.100 myimage

# DNS search domain
docker run --dns-search example.com myimage
```

**Docker Compose networking:**

```yaml
version: '3.8'
services:
  api:
    image: myapi
    networks:
      - backend
    # Can reach 'db' by hostname

  db:
    image: postgres
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

---

### Q12: How do you expose container ports?

**Answer:**

```bash
# Publish port (host:container)
docker run -p 8080:80 nginx

# Publish to specific interface
docker run -p 127.0.0.1:8080:80 nginx

# Publish range of ports
docker run -p 8080-8090:80-90 myimage

# Publish all exposed ports to random host ports
docker run -P nginx

# UDP port
docker run -p 53:53/udp dns-server
```

**Dockerfile EXPOSE:**

```dockerfile
# Documentation only - doesn't actually publish
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
EXPOSE 53/udp
```

**View port mappings:**

```bash
docker port container-name
docker ps  # Shows port mappings in PORTS column
```

---

## Docker Volumes and Storage

### Q13: Explain Docker storage types.

**Answer:**

| Type | Description | Use Case |
|------|-------------|----------|
| Volumes | Managed by Docker | Persistent data, sharing |
| Bind mounts | Map host path | Development, config files |
| tmpfs | In-memory storage | Sensitive data, temp files |

```bash
# Volume (Docker managed)
docker volume create mydata
docker run -v mydata:/app/data myimage

# Bind mount (host directory)
docker run -v /host/path:/container/path myimage
docker run -v $(pwd):/app myimage

# tmpfs (memory only)
docker run --tmpfs /app/temp myimage

# Read-only mount
docker run -v mydata:/app/data:ro myimage
```

**Volume commands:**

```bash
# Create volume
docker volume create myvolume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect myvolume

# Remove volume
docker volume rm myvolume

# Remove unused volumes
docker volume prune

# Copy data to volume
docker run --rm -v myvolume:/data -v $(pwd):/backup \
  alpine cp -r /backup/. /data/
```

---

### Q14: How do you manage persistent data in containers?

**Answer:**

**Named volumes (recommended for production):**

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  postgres-data:
    driver: local
```

**Backup and restore volumes:**

```bash
# Backup volume to tar file
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .

# Restore volume from tar file
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /data
```

**Volume drivers for cloud storage:**

```bash
# AWS EFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=fs-xxx.efs.region.amazonaws.com,nfsvers=4.1 \
  --opt device=:/ \
  efs-volume
```

---

## Docker Compose

### Q15: Explain Docker Compose file structure.

**Answer:**

```yaml
version: '3.8'

services:
  # Web application
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    image: myapp-web:latest
    container_name: web
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    networks:
      - frontend
      - backend
    volumes:
      - ./uploads:/app/uploads
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Database
  db:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache
  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

volumes:
  postgres-data:
  redis-data:
```

---

### Q16: Explain Docker Compose commands.

**Answer:**

```bash
# Start services
docker compose up
docker compose up -d              # Detached mode
docker compose up --build         # Rebuild images
docker compose up --scale web=3   # Scale service

# Stop services
docker compose down
docker compose down -v            # Remove volumes too
docker compose down --rmi all     # Remove images too

# View status
docker compose ps
docker compose top

# Logs
docker compose logs
docker compose logs -f            # Follow logs
docker compose logs web           # Specific service

# Execute command
docker compose exec web sh
docker compose exec db psql -U postgres

# Build images
docker compose build
docker compose build --no-cache

# Pull images
docker compose pull

# Restart services
docker compose restart
docker compose restart web

# Stop without removing
docker compose stop

# Remove stopped containers
docker compose rm

# View config
docker compose config            # Validated config
```

---

## Docker Security

### Q17: Explain Docker security best practices.

**Answer:**

1. **Use non-root user:**

```dockerfile
# Create user and switch
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -D appuser

# Set ownership
COPY --chown=appuser:appgroup . /app

# Switch to non-root
USER appuser
```

2. **Use minimal base images:**

```dockerfile
# Prefer
FROM alpine:3.18
FROM gcr.io/distroless/static-debian11
FROM scratch

# Avoid
FROM ubuntu:latest
```

3. **Don't store secrets in images:**

```dockerfile
# BAD
ENV API_KEY=secret123
COPY secrets.json /app/

# GOOD - use runtime secrets
# docker run -e API_KEY=$API_KEY myimage
# Or use Docker secrets / Vault
```

4. **Scan images for vulnerabilities:**

```bash
# Using Trivy
trivy image myimage:latest

# Using Docker Scout
docker scout cves myimage:latest

# Using Snyk
snyk container test myimage:latest
```

5. **Use specific image tags:**

```dockerfile
# BAD
FROM python:latest

# GOOD
FROM python:3.11.4-slim-bookworm
```

6. **Read-only filesystem:**

```bash
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myimage
```

7. **Drop capabilities:**

```bash
docker run --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  myimage
```

8. **Use security options:**

```bash
docker run \
  --security-opt no-new-privileges:true \
  --security-opt seccomp=default.json \
  myimage
```

---

### Q18: What are Docker security scanning tools?

**Answer:**

| Tool | Type | Features |
|------|------|----------|
| Trivy | Open source | CVE scanning, IaC, secrets |
| Snyk | Commercial | CVE, code, dependencies |
| Clair | Open source | Static analysis |
| Docker Scout | Docker native | CVE, recommendations |
| Anchore | Open source | Policy-based scanning |

**Trivy example:**

```bash
# Scan image
trivy image myimage:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myimage:latest

# Scan filesystem
trivy fs .

# Scan in CI (exit code on findings)
trivy image --exit-code 1 --severity HIGH,CRITICAL myimage:latest

# Output formats
trivy image -f json -o results.json myimage:latest
trivy image -f sarif -o results.sarif myimage:latest
```

**CI integration:**

```yaml
# GitHub Actions
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myimage:latest
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
```

---

## Troubleshooting and Debugging

### Q19: How do you debug a running container?

**Answer:**

```bash
# Execute command in running container
docker exec -it container-name /bin/bash
docker exec -it container-name /bin/sh  # Alpine

# View logs
docker logs container-name
docker logs -f container-name           # Follow
docker logs --tail 100 container-name   # Last 100 lines
docker logs --since 1h container-name   # Last hour
docker logs --timestamps container-name # With timestamps

# Inspect container details
docker inspect container-name
docker inspect -f '{{.State.Status}}' container-name
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-name

# View resource usage
docker stats container-name

# View processes
docker top container-name

# Copy files from container
docker cp container-name:/app/logs ./logs

# View filesystem changes
docker diff container-name
```

---

### Q20: How do you debug a container that won't start?

**Answer:**

```bash
# Check container logs
docker logs container-name

# Check events
docker events --since 1h

# Inspect container for error
docker inspect container-name | jq '.[0].State'

# Override entrypoint for debugging
docker run -it --entrypoint /bin/sh myimage

# Run with different command
docker run -it myimage /bin/sh

# Check if image works
docker run --rm myimage echo "test"

# Debug with network tools
docker run -it --rm --network container:mycontainer \
  nicolaka/netshoot

# Check resource limits
docker inspect -f '{{.HostConfig.Memory}}' container-name
```

**Common issues and solutions:**

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| OOMKilled | `docker inspect` shows OOMKilled=true | Increase memory limit |
| Exit code 137 | SIGKILL (usually OOM) | Increase memory |
| Exit code 1 | Application error | Check logs |
| Exit code 127 | Command not found | Check CMD/ENTRYPOINT |
| Exit code 126 | Permission denied | Check file permissions |

---

### Q21: How do you optimize Docker container performance?

**Answer:**

1. **Resource limits:**

```bash
docker run -d \
  --memory=512m \
  --memory-swap=1g \
  --cpus=1.5 \
  --cpu-shares=1024 \
  myimage
```

2. **Storage driver optimization:**

```bash
# Check storage driver
docker info | grep "Storage Driver"

# Use overlay2 (recommended)
# In /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```

3. **Logging configuration:**

```bash
# Limit log size
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myimage
```

4. **Network optimization:**

```bash
# Use host network for performance
docker run --network host myimage

# Or optimize bridge network
docker network create --driver bridge \
  --opt com.docker.network.driver.mtu=9000 \
  fastnet
```

5. **Build cache optimization:**

```bash
# Use BuildKit
export DOCKER_BUILDKIT=1

# Use cache mounts
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## Summary

| Topic | Key Concepts |
|-------|--------------|
| Images | Layers, caching, multi-stage builds |
| Containers | Lifecycle, resource limits, health checks |
| Networking | Bridge, host, overlay, DNS resolution |
| Storage | Volumes, bind mounts, tmpfs |
| Security | Non-root, scanning, secrets management |
| Compose | Multi-container orchestration |
| Debugging | Logs, exec, inspect, stats |

**Essential commands:**

```bash
# Image management
docker build, docker pull, docker push, docker images, docker rmi

# Container management
docker run, docker start, docker stop, docker rm, docker ps

# Debugging
docker logs, docker exec, docker inspect, docker stats

# Networking
docker network create, docker network connect

# Volumes
docker volume create, docker volume ls, docker volume rm

# Cleanup
docker system prune, docker image prune, docker volume prune
```