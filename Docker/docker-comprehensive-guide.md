# 🐳 Docker Comprehensive Guide for Senior DevOps Engineers
*Advanced Containerization, Architecture, and Enterprise Patterns*

## Table of Contents
1. [Docker Architecture & Fundamentals](#docker-architecture--fundamentals)
2. [Container Lifecycle & Process Management](#container-lifecycle--process-management)
3. [Image Management & Optimization](#image-management--optimization)
4. [Networking & Service Discovery](#networking--service-discovery)
5. [Storage & Data Management](#storage--data-management)
6. [Security & Hardening](#security--hardening)
7. [Multi-Stage Builds & Build Optimization](#multi-stage-builds--build-optimization)
8. [Docker Compose & Orchestration](#docker-compose--orchestration)
9. [Registry Management & Distribution](#registry-management--distribution)
10. [Monitoring & Observability](#monitoring--observability)
11. [Production Deployment Patterns](#production-deployment-patterns)
12. [Troubleshooting & Performance Tuning](#troubleshooting--performance-tuning)

---

## 🔹 Docker Architecture & Fundamentals

### Container Technology Foundation

**Understanding Containerization:**
Containers are not virtual machines. They're isolated processes that share the host kernel while maintaining separate user spaces, file systems, and network stacks.

**Linux Kernel Features:**
Docker leverages several Linux kernel features to provide isolation:

**Namespaces:**
- **PID Namespace**: Process isolation - containers see only their own processes
- **Network Namespace**: Network stack isolation - separate network interfaces, routing tables
- **Mount Namespace**: File system isolation - separate mount points and file system views
- **UTS Namespace**: Hostname isolation - containers can have different hostnames
- **IPC Namespace**: Inter-process communication isolation - separate message queues, semaphores
- **User Namespace**: User ID isolation - root inside container != root on host

**Control Groups (cgroups):**
Resource limitation and accounting:
- **CPU**: CPU time allocation and throttling
- **Memory**: Memory usage limits and OOM handling
- **Block I/O**: Disk I/O bandwidth control
- **Network**: Network bandwidth control (with tc)
- **Device**: Device access control

**Union File Systems:**
Layered file system that enables:
- **Copy-on-Write**: Efficient storage by sharing read-only layers
- **Layer Reuse**: Multiple containers can share base layers
- **Incremental Updates**: Only changed layers need to be transferred

### Docker Daemon Architecture

**Docker Engine Components:**
- **dockerd**: Main daemon process managing containers, images, networks, volumes
- **containerd**: High-level container runtime managing container lifecycle
- **runc**: Low-level container runtime implementing OCI specification
- **docker-proxy**: Handles port forwarding for published ports

**Client-Server Architecture:**
```bash
# Docker client communicates with daemon via REST API
docker version  # Shows client and server versions
docker system info  # Detailed daemon information
docker system events  # Real-time events from daemon
```

**Container Runtime Interface (CRI):**
Docker implements the Open Container Initiative (OCI) specifications:
- **Runtime Spec**: How to run containers
- **Image Spec**: How to build and distribute images
- **Distribution Spec**: How to distribute images via registries

### Process Model Understanding

**Container vs Process:**
A container is essentially a process (or process tree) with enhanced isolation. Understanding this is crucial for:
- **Resource Management**: Containers inherit process limitations
- **Signal Handling**: Containers respond to Unix signals
- **Exit Codes**: Container exit codes reflect main process exit codes
- **PID 1 Problem**: Main process must handle child processes properly

**Init Systems in Containers:**
```dockerfile
# Problem: Application doesn't handle signals properly
FROM ubuntu:20.04
CMD ["my-app"]

# Solution: Use proper init system
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["my-app"]
```

---

## 🔹 Container Lifecycle & Process Management

### Container States and Transitions

**Container Lifecycle States:**
- **Created**: Container exists but not started
- **Running**: Container is executing
- **Paused**: Container processes are suspended
- **Restarting**: Container is being restarted
- **Exited**: Container has stopped (successfully or with error)
- **Dead**: Container removal failed

**State Transitions:**
```bash
# Create container without starting
docker create --name myapp nginx

# Start created container
docker start myapp

# Pause running container (freeze processes)
docker pause myapp

# Unpause container
docker unpause myapp

# Stop container (SIGTERM then SIGKILL)
docker stop myapp

# Kill container immediately (SIGKILL)
docker kill myapp

# Remove stopped container
docker rm myapp
```

### Signal Handling and Graceful Shutdown

**Signal Propagation:**
Understanding how signals reach containerized applications is critical for graceful shutdowns:

```bash
# Docker stop sends SIGTERM to PID 1, waits 10s, then SIGKILL
docker stop --time=30 myapp

# Docker kill sends specified signal
docker kill --signal=SIGUSR1 myapp
```

**Proper Signal Handling in Applications:**
```python
# Python application with proper signal handling
import signal
import sys
import time
import threading

class GracefulShutdown:
    def __init__(self):
        self.shutdown = False
        signal.signal(signal.SIGTERM, self._exit_gracefully)
        signal.signal(signal.SIGINT, self._exit_gracefully)
    
    def _exit_gracefully(self, signum, frame):
        print(f"Received signal {signum}, shutting down gracefully...")
        self.shutdown = True
    
    def run(self):
        while not self.shutdown:
            # Application logic here
            print("Working...")
            time.sleep(1)
        
        print("Cleanup completed, exiting...")
        sys.exit(0)

if __name__ == "__main__":
    app = GracefulShutdown()
    app.run()
```

### Resource Management and Limits

**Memory Management:**
```bash
# Set memory limits
docker run --memory=512m --memory-swap=1g nginx

# Memory reservation (soft limit)
docker run --memory=1g --memory-reservation=512m nginx

# OOM kill disable (dangerous in production)
docker run --oom-kill-disable --memory=512m nginx
```

**CPU Management:**
```bash
# CPU shares (relative weight)
docker run --cpu-shares=512 nginx

# CPU quota (hard limit)
docker run --cpu-quota=50000 --cpu-period=100000 nginx  # 50% of one CPU

# CPU affinity
docker run --cpuset-cpus=0,1 nginx  # Use only CPU 0 and 1
```

**I/O Management:**
```bash
# Block I/O weight
docker run --blkio-weight=500 nginx

# Device read/write limits
docker run --device-read-bps=/dev/sda:1mb nginx
docker run --device-write-bps=/dev/sda:1mb nginx
```

### Health Checks and Monitoring

**Container Health Checks:**
```dockerfile
# Dockerfile health check
FROM nginx:alpine
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

```bash
# Runtime health check
docker run -d --name web \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  nginx
```

**Advanced Health Check Patterns:**
```bash
# Multi-service health check
#!/bin/bash
# health-check.sh
set -e

# Check web server
curl -f http://localhost:8080/health || exit 1

# Check database connection
nc -z database 5432 || exit 1

# Check external dependencies
curl -f http://external-api/health || exit 1

echo "All health checks passed"
```

---

## 🔹 Image Management & Optimization

### Image Architecture and Layers

**Understanding Image Layers:**
Docker images are composed of read-only layers stacked on top of each other. Each instruction in a Dockerfile creates a new layer.

**Layer Caching Strategy:**
```dockerfile
# Inefficient - cache invalidated on any code change
FROM node:16-alpine
COPY . /app
WORKDIR /app
RUN npm install
CMD ["npm", "start"]

# Efficient - dependencies cached separately
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["npm", "start"]
```

**Layer Inspection:**
```bash
# Inspect image layers
docker history nginx:alpine

# Detailed layer information
docker inspect nginx:alpine

# Show layer sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Image Optimization Techniques

**Base Image Selection:**
```dockerfile
# Full OS (large, more tools)
FROM ubuntu:20.04

# Minimal OS (smaller, fewer tools)
FROM alpine:3.14

# Distroless (smallest, no shell)
FROM gcr.io/distroless/java:11

# Scratch (empty, for static binaries)
FROM scratch
```

**Multi-Stage Build Optimization:**
```dockerfile
# Build stage
FROM golang:1.19-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Production stage
FROM alpine:3.14
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

**Layer Optimization Strategies:**
```dockerfile
# Combine RUN instructions to reduce layers
RUN apt-get update && \
    apt-get install -y \
        curl \
        vim \
        git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Use .dockerignore to exclude unnecessary files
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.nyc_output
```

### Image Security Scanning

**Vulnerability Scanning:**
```bash
# Scan image for vulnerabilities
docker scan nginx:alpine

# Scan with specific severity
docker scan --severity=high nginx:alpine

# Generate SBOM (Software Bill of Materials)
docker sbom nginx:alpine
```

**Security Best Practices:**
```dockerfile
# Run as non-root user
FROM alpine:3.14
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser

# Use specific versions
FROM node:16.14.2-alpine3.14

# Minimize attack surface
FROM alpine:3.14
RUN apk add --no-cache nodejs npm && \
    rm -rf /var/cache/apk/*
```

---

## 🔹 Networking & Service Discovery

### Docker Networking Models

**Network Drivers:**
- **bridge**: Default network driver for standalone containers
- **host**: Remove network isolation, use host's network directly
- **overlay**: Multi-host networking for swarm services
- **macvlan**: Assign MAC address to container, appears as physical device
- **none**: Disable networking

**Bridge Network Deep Dive:**
```bash
# Create custom bridge network
docker network create --driver bridge \
  --subnet=172.20.0.0/16 \
  --ip-range=172.20.240.0/20 \
  --gateway=172.20.0.1 \
  mynetwork

# Inspect network configuration
docker network inspect mynetwork

# Connect container to network
docker run -d --name web --network mynetwork nginx
```

**Network Isolation and Security:**
```bash
# Create isolated networks for different tiers
docker network create frontend
docker network create backend
docker network create database

# Web server on frontend and backend networks
docker run -d --name web \
  --network frontend \
  nginx

docker network connect backend web

# Database only on backend network
docker run -d --name db \
  --network database \
  postgres:13
```

### Service Discovery Patterns

**DNS-based Service Discovery:**
```bash
# Containers can reach each other by name within same network
docker network create app-network

docker run -d --name database \
  --network app-network \
  postgres:13

docker run -d --name api \
  --network app-network \
  -e DATABASE_URL=postgresql://user:pass@database:5432/db \
  myapi:latest
```

**External Service Discovery:**
```dockerfile
# Application with service discovery
FROM alpine:3.14
RUN apk add --no-cache curl jq
COPY service-discovery.sh /usr/local/bin/
COPY app /usr/local/bin/
CMD ["/usr/local/bin/service-discovery.sh"]
```

```bash
#!/bin/bash
# service-discovery.sh
set -e

# Discover services from external registry
SERVICES=$(curl -s http://consul:8500/v1/catalog/services | jq -r 'keys[]')

for service in $SERVICES; do
    SERVICE_INFO=$(curl -s "http://consul:8500/v1/catalog/service/$service")
    SERVICE_IP=$(echo $SERVICE_INFO | jq -r '.[0].ServiceAddress')
    SERVICE_PORT=$(echo $SERVICE_INFO | jq -r '.[0].ServicePort')
    
    echo "$SERVICE_IP:$SERVICE_PORT" > "/etc/services/$service"
done

# Start main application
exec /usr/local/bin/app
```

### Load Balancing and Proxy Patterns

**Nginx as Reverse Proxy:**
```nginx
# nginx.conf
upstream backend {
    server api1:8080;
    server api2:8080;
    server api3:8080;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**HAProxy Configuration:**
```haproxy
# haproxy.cfg
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 web1:8080 check
    server web2 web2:8080 check
    server web3 web3:8080 check
```

---

## 🔹 Storage & Data Management

### Volume Types and Use Cases

**Volume Types:**
- **Named Volumes**: Managed by Docker, persistent across container restarts
- **Bind Mounts**: Direct host filesystem mounts
- **tmpfs Mounts**: In-memory storage, lost on container stop

**Named Volumes:**
```bash
# Create named volume
docker volume create mydata

# Use volume in container
docker run -d --name app \
  -v mydata:/app/data \
  myapp:latest

# Inspect volume
docker volume inspect mydata

# Backup volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .
```

**Bind Mounts for Development:**
```bash
# Development with live code reloading
docker run -d --name dev-app \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/config:/app/config \
  -p 3000:3000 \
  node:16-alpine npm run dev
```

### Data Persistence Patterns

**Database Persistence:**
```yaml
# docker-compose.yml for database
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

volumes:
  postgres_data:
    driver: local
```

**Backup and Restore Strategies:**
```bash
# Database backup
docker exec postgres pg_dump -U user myapp > backup.sql

# Database restore
docker exec -i postgres psql -U user myapp < backup.sql

# Volume backup with versioning
#!/bin/bash
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

docker run --rm \
  -v mydata:/data:ro \
  -v $BACKUP_DIR:/backup \
  alpine sh -c "tar czf /backup/mydata_$DATE.tar.gz -C /data ."

# Keep only last 7 backups
find $BACKUP_DIR -name "mydata_*.tar.gz" -mtime +7 -delete
```

### Storage Drivers and Performance

**Storage Driver Selection:**
```bash
# Check current storage driver
docker info | grep "Storage Driver"

# Configure storage driver (daemon.json)
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

**Performance Optimization:**
```bash
# Use tmpfs for temporary data
docker run -d --name app \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  myapp:latest

# Optimize for I/O intensive workloads
docker run -d --name database \
  --device=/dev/sdb:/dev/sdb \
  --volume-driver=local \
  -v /mnt/ssd:/var/lib/postgresql/data \
  postgres:13
```

---

## 🔹 Security & Hardening

### Container Security Fundamentals

**Principle of Least Privilege:**
```dockerfile
# Create non-root user
FROM alpine:3.14
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Set ownership of application files
COPY --chown=appuser:appgroup app /app/
USER appuser
WORKDIR /app
CMD ["./app"]
```

**Capability Management:**
```bash
# Drop all capabilities, add only required ones
docker run -d --name secure-app \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --user 1001:1001 \
  myapp:latest

# Run with read-only root filesystem
docker run -d --name readonly-app \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myapp:latest
```

### Security Scanning and Compliance

**Image Vulnerability Assessment:**
```bash
# Comprehensive security scan
docker scan --file Dockerfile --exclude-base myapp:latest

# Generate security report
docker scan --json myapp:latest > security-report.json

# Check for secrets in image
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:latest
```

**Runtime Security:**
```bash
# Security profiles with AppArmor
docker run -d --name secure-app \
  --security-opt apparmor:docker-default \
  myapp:latest

# SELinux labels
docker run -d --name selinux-app \
  --security-opt label:type:container_runtime_t \
  myapp:latest

# Seccomp profiles
docker run -d --name restricted-app \
  --security-opt seccomp:restricted.json \
  myapp:latest
```

### Secrets Management

**Docker Secrets (Swarm Mode):**
```bash
# Create secret
echo "mysecretpassword" | docker secret create db_password -

# Use secret in service
docker service create --name myapp \
  --secret db_password \
  myapp:latest
```

**External Secrets Management:**
```dockerfile
# Integration with HashiCorp Vault
FROM alpine:3.14
RUN apk add --no-cache curl jq
COPY vault-init.sh /usr/local/bin/
COPY app /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/vault-init.sh"]
```

```bash
#!/bin/bash
# vault-init.sh
set -e

# Authenticate with Vault
VAULT_TOKEN=$(curl -s -X POST \
  -d '{"role_id":"'$VAULT_ROLE_ID'","secret_id":"'$VAULT_SECRET_ID'"}' \
  $VAULT_ADDR/v1/auth/approle/login | jq -r '.auth.client_token')

# Retrieve secrets
DB_PASSWORD=$(curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/secret/data/myapp | jq -r '.data.data.db_password')

# Export as environment variable
export DB_PASSWORD

# Start application
exec /usr/local/bin/app
```

---

## 🔹 Multi-Stage Builds & Build Optimization

### Advanced Multi-Stage Patterns

**Build Optimization Strategy:**
```dockerfile
# Multi-stage build with dependency caching
FROM node:16-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:16-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine AS runtime
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package*.json ./
EXPOSE 3000
CMD ["npm", "start"]
```

**Language-Specific Optimizations:**

**Go Applications:**
```dockerfile
FROM golang:1.19-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM scratch
COPY --from=builder /app/main /
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
CMD ["/main"]
```

**Java Applications:**
```dockerfile
FROM openjdk:11-jdk-slim AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build Context Optimization

**Efficient Build Context:**
```dockerignore
# .dockerignore
node_modules
npm-debug.log*
.git
.gitignore
README.md
.env
.nyc_output
coverage
.nyc_output
.cache
.vscode
.idea
*.swp
*.swo
*~
```

**Build Cache Strategies:**
```bash
# Use BuildKit for advanced caching
export DOCKER_BUILDKIT=1

# Build with cache mount
docker build --build-arg BUILDKIT_INLINE_CACHE=1 \
  --cache-from myapp:cache \
  -t myapp:latest .

# Push cache layers
docker push myapp:cache
```

### Parallel Builds and CI/CD Integration

**Parallel Multi-Stage Builds:**
```dockerfile
FROM alpine:3.14 AS base
RUN apk add --no-cache ca-certificates

FROM golang:1.19-alpine AS backend-builder
WORKDIR /app
COPY backend/ .
RUN go build -o backend .

FROM node:16-alpine AS frontend-builder
WORKDIR /app
COPY frontend/ .
RUN npm ci && npm run build

FROM base AS final
COPY --from=backend-builder /app/backend /usr/local/bin/
COPY --from=frontend-builder /app/dist /var/www/html/
EXPOSE 8080
CMD ["backend"]
```

---

## 🔹 Docker Compose & Orchestration

### Advanced Compose Patterns

**Environment-Specific Configurations:**
```yaml
# docker-compose.yml (base)
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
    depends_on:
      - database
  
  database:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

```yaml
# docker-compose.override.yml (development)
version: '3.8'
services:
  web:
    build:
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
  
  database:
    ports:
      - "5432:5432"
```

**Service Dependencies and Health Checks:**
```yaml
version: '3.8'
services:
  web:
    build: .
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  
  database:
    image: postgres:13
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  redis:
    image: redis:6-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

### Scaling and Load Balancing

**Service Scaling:**
```bash
# Scale specific service
docker-compose up --scale web=3 --scale worker=2

# Scale with load balancer
docker-compose -f docker-compose.yml -f docker-compose.scale.yml up
```

```yaml
# docker-compose.scale.yml
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - web
  
  web:
    ports: []  # Remove direct port mapping
```

**Configuration Management:**
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    configs:
      - source: app_config
        target: /app/config.yml
        mode: 0440
    secrets:
      - db_password
      - api_key

configs:
  app_config:
    file: ./config/app.yml

secrets:
  db_password:
    external: true
  api_key:
    external: true
```

---

## 🔹 Registry Management & Distribution

### Private Registry Setup

**Docker Registry Configuration:**
```yaml
# docker-compose.registry.yml
version: '3.8'
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - registry_data:/data
      - ./auth:/auth
    restart: unless-stopped

volumes:
  registry_data:
```

**Registry Authentication:**
```bash
# Create htpasswd file
docker run --rm --entrypoint htpasswd \
  httpd:2 -Bbn admin password > auth/htpasswd

# Login to private registry
docker login localhost:5000
```

### Image Distribution Strategies

**Multi-Architecture Images:**
```bash
# Build for multiple architectures
docker buildx create --name multiarch --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myregistry.com/myapp:latest --push .
```

**Image Promotion Pipeline:**
```bash
#!/bin/bash
# promote-image.sh
set -e

SOURCE_TAG="myapp:dev-${BUILD_NUMBER}"
STAGING_TAG="myapp:staging-${BUILD_NUMBER}"
PROD_TAG="myapp:prod-${BUILD_NUMBER}"

# Pull from development
docker pull $SOURCE_TAG

# Run security scan
docker scan $SOURCE_TAG

# Tag for staging
docker tag $SOURCE_TAG $STAGING_TAG
docker push $STAGING_TAG

# Deploy to staging and run tests
deploy_to_staging $STAGING_TAG
run_integration_tests

# Promote to production if tests pass
if [ $? -eq 0 ]; then
    docker tag $SOURCE_TAG $PROD_TAG
    docker push $PROD_TAG
    echo "Image promoted to production: $PROD_TAG"
fi
```

### Content Trust and Security

**Docker Content Trust:**
```bash
# Enable content trust
export DOCKER_CONTENT_TRUST=1

# Sign and push image
docker push myregistry.com/myapp:latest

# Verify signed image
docker pull myregistry.com/myapp:latest
```

**Image Signing with Notary:**
```bash
# Initialize repository
notary init myregistry.com/myapp

# Add signing key
notary key generate myregistry.com/myapp

# Sign specific tag
notary add myregistry.com/myapp latest \
  sha256:abc123... --publish
```

---

## 🔹 Monitoring & Observability

### Container Metrics and Logging

**Metrics Collection:**
```bash
# Container resource usage
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Detailed container inspection
docker inspect --format='{{.State.Health.Status}}' myapp
```

**Centralized Logging:**
```yaml
# docker-compose.logging.yml
version: '3.8'
services:
  app:
    image: myapp:latest
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: myapp
  
  fluentd:
    image: fluent/fluentd:v1.14
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf
      - fluentd_logs:/var/log/fluentd
  
  elasticsearch:
    image: elasticsearch:7.15.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

volumes:
  fluentd_logs:
```

### Application Performance Monitoring

**Prometheus Integration:**
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    labels:
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8080"
      - "prometheus.io/path=/metrics"
  
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

volumes:
  prometheus_data:
```

**Distributed Tracing:**
```yaml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_ZIPKIN_HTTP_PORT=9411
  
  app:
    image: myapp:latest
    environment:
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
    depends_on:
      - jaeger
```

---

## 🔹 Production Deployment Patterns

### Blue-Green Deployment

**Blue-Green with Docker Compose:**
```bash
#!/bin/bash
# blue-green-deploy.sh
set -e

CURRENT_COLOR=$(docker-compose -f docker-compose.yml ps -q web | wc -l)
if [ $CURRENT_COLOR -eq 0 ]; then
    NEW_COLOR="blue"
    OLD_COLOR="green"
else
    CURRENT_COLOR_NAME=$(docker-compose -f docker-compose.yml config | grep -A 5 "web:" | grep "image" | cut -d: -f3)
    if [[ $CURRENT_COLOR_NAME == *"blue"* ]]; then
        NEW_COLOR="green"
        OLD_COLOR="blue"
    else
        NEW_COLOR="blue"
        OLD_COLOR="green"
    fi
fi

echo "Deploying to $NEW_COLOR environment"

# Deploy new version
docker-compose -f docker-compose.$NEW_COLOR.yml up -d

# Health check
sleep 30
if curl -f http://localhost:8081/health; then
    echo "Health check passed, switching traffic"
    
    # Update load balancer
    docker-compose -f docker-compose.lb.yml up -d
    
    # Stop old version
    docker-compose -f docker-compose.$OLD_COLOR.yml down
    
    echo "Deployment completed successfully"
else
    echo "Health check failed, rolling back"
    docker-compose -f docker-compose.$NEW_COLOR.yml down
    exit 1
fi
```

### Rolling Updates

**Rolling Update Strategy:**
```bash
#!/bin/bash
# rolling-update.sh
set -e

NEW_IMAGE=$1
SERVICE_NAME="web"
REPLICAS=3

echo "Starting rolling update to $NEW_IMAGE"

for i in $(seq 1 $REPLICAS); do
    echo "Updating replica $i/$REPLICAS"
    
    # Update one replica
    docker-compose up -d --scale $SERVICE_NAME=$REPLICAS --no-recreate
    
    # Wait for health check
    sleep 30
    
    # Verify deployment
    if ! curl -f http://localhost:8080/health; then
        echo "Health check failed, rolling back"
        docker-compose up -d --scale $SERVICE_NAME=$REPLICAS
        exit 1
    fi
done

echo "Rolling update completed successfully"
```

### Canary Deployment

**Canary with Traffic Splitting:**
```nginx
# nginx-canary.conf
upstream backend_stable {
    server web-stable:8080;
}

upstream backend_canary {
    server web-canary:8080;
}

split_clients $remote_addr $variant {
    10%     canary;
    *       stable;
}

server {
    listen 80;
    
    location / {
        if ($variant = "canary") {
            proxy_pass http://backend_canary;
        }
        proxy_pass http://backend_stable;
    }
}
```

---

## 🔹 Troubleshooting & Performance Tuning

### Debugging Container Issues

**Container Debugging Techniques:**
```bash
# Inspect container processes
docker exec myapp ps aux

# Check container logs with timestamps
docker logs -t --since="2h" myapp

# Monitor container resources in real-time
docker stats myapp

# Execute shell in running container
docker exec -it myapp /bin/sh

# Debug networking issues
docker exec myapp netstat -tulpn
docker exec myapp nslookup database
```

**System-level Debugging:**
```bash
# Check Docker daemon logs
journalctl -u docker.service -f

# Inspect Docker system information
docker system df  # Disk usage
docker system events  # Real-time events
docker system prune  # Clean up unused resources

# Network debugging
docker network ls
docker network inspect bridge
```

### Performance Optimization

**Resource Optimization:**
```bash
# Optimize for CPU-intensive workloads
docker run -d --name cpu-app \
  --cpus="2.0" \
  --cpu-shares=1024 \
  myapp:latest

# Optimize for memory-intensive workloads
docker run -d --name memory-app \
  --memory=4g \
  --memory-swap=6g \
  --oom-kill-disable=false \
  myapp:latest

# Optimize for I/O intensive workloads
docker run -d --name io-app \
  --device-read-bps=/dev/sda:100mb \
  --device-write-bps=/dev/sda:100mb \
  myapp:latest
```

**Container Startup Optimization:**
```dockerfile
# Optimize container startup time
FROM alpine:3.14

# Use multi-stage builds to reduce image size
FROM node:16-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM alpine:3.14
RUN apk add --no-cache nodejs npm
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Use exec form for faster startup
CMD ["node", "server.js"]
```

### Common Issues and Solutions

**Memory Issues:**
```bash
# Diagnose OOM kills
dmesg | grep -i "killed process"

# Monitor memory usage patterns
docker stats --format "table {{.Container}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Analyze memory leaks
docker exec myapp cat /proc/meminfo
```

**Network Issues:**
```bash
# Test container connectivity
docker exec web ping database
docker exec web telnet database 5432

# Check port bindings
docker port myapp

# Inspect network configuration
docker exec myapp ip route show
docker exec myapp iptables -L
```

**Storage Issues:**
```bash
# Check disk usage
docker system df -v

# Clean up unused resources
docker system prune -a --volumes

# Monitor I/O performance
docker exec myapp iostat -x 1
```

This comprehensive Docker guide provides the deep understanding and practical knowledge essential for senior DevOps engineers managing containerized applications in production environments. The focus is on architectural concepts, enterprise patterns, and operational excellence rather than basic implementation.
