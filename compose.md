# Docker Compose — Comprehensive Study Notes

> **Level:** Beginner to Intermediate  
> **Topics covered:** Core concepts, YAML syntax, commands, networking, volumes, environment variables, use cases, best practices

---

## Table of Contents

1. [What is Docker Compose?](#1-what-is-docker-compose)
2. [Docker vs Docker Compose](#2-docker-vs-docker-compose)
3. [The docker-compose.yaml File](#3-the-docker-composeyaml-file)
4. [Core YAML Fields Reference](#4-core-yaml-fields-reference)
5. [Essential Commands](#5-essential-commands)
6. [Networking](#6-networking)
7. [Volumes & Data Persistence](#7-volumes--data-persistence)
8. [Environment Variables & Secrets](#8-environment-variables--secrets)
9. [Startup Order & Health Checks](#9-startup-order--health-checks)
10. [Real-World Example: Full Stack App](#10-real-world-example-full-stack-app)
11. [Use Cases](#11-use-cases)
12. [Docker Compose vs Kubernetes vs Docker Swarm](#12-docker-compose-vs-kubernetes-vs-docker-swarm)
13. [Common Mistakes & Gotchas](#13-common-mistakes--gotchas)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. What is Docker Compose?

Docker Compose is a tool for **defining and running multi-container Docker applications** using a single declarative YAML configuration file.

Instead of manually running `docker run` for every service, you describe your entire application stack — services, networks, volumes, and environment variables — in one `docker-compose.yaml` file and start everything with one command.

**Key idea:** One file. One command. Your entire app stack is running.

---

## 2. Docker vs Docker Compose

| Feature | Docker (CLI) | Docker Compose |
|---|---|---|
| Scope | Single container | Multi-container applications |
| Configuration | Command-line flags | YAML file |
| Startup | `docker run ...` per container | `docker compose up` |
| Networking | Manual setup | Auto-created shared network |
| Use case | Running individual images | Local dev, CI/CD, microservices |

### Without Compose (painful)

```bash
docker network create myapp-net

docker run -d --name db \
  --network myapp-net \
  -e POSTGRES_PASSWORD=secret \
  -v db_data:/var/lib/postgresql/data \
  postgres:15

docker run -d --name cache \
  --network myapp-net \
  redis:7

docker run -d --name web \
  --network myapp-net \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://db:5432/myapp \
  myapp:latest
```

### With Compose (clean)

```bash
docker compose up
```

---

## 3. The docker-compose.yaml File

The YAML file has three main top-level sections:

```yaml
version: "3.9"   # Compose file format version

services:        # Container definitions (required)
  ...

volumes:         # Named volume definitions (optional)
  ...

networks:        # Custom network definitions (optional)
  ...
```

### Minimal example

```yaml
version: "3.9"

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
```

### Full annotated example

```yaml
version: "3.9"

services:

  web:                              # Service name (used as hostname on the network)
    build: .                        # Build image from Dockerfile in current directory
    ports:
      - "3000:3000"                 # "HOST_PORT:CONTAINER_PORT"
    environment:
      - NODE_ENV=production
      - PORT=3000
    depends_on:
      - db
      - cache
    volumes:
      - ./src:/usr/src/app/src      # Bind mount: sync local code into container
    restart: unless-stopped         # Auto-restart policy

  db:
    image: postgres:15              # Pull from Docker Hub
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASS} # Reference from .env file
      POSTGRES_DB: myapp
    volumes:
      - db_data:/var/lib/postgresql/data  # Named volume for persistence

  cache:
    image: redis:7
    ports:
      - "6379:6379"                 # Expose to host (optional, mainly for debugging)

volumes:
  db_data:                          # Declare the named volume here
```

---

## 4. Core YAML Fields Reference

### Service-level fields

| Field | Description | Example |
|---|---|---|
| `image` | Pull a pre-built image | `image: postgres:15` |
| `build` | Build from a Dockerfile | `build: .` or `build: ./api` |
| `ports` | Map host port to container port | `- "8080:80"` |
| `expose` | Expose port to other services only (not host) | `- "3000"` |
| `environment` | Set environment variables | `- KEY=value` |
| `env_file` | Load env vars from a file | `env_file: .env` |
| `volumes` | Mount files or directories | `- ./code:/app` |
| `depends_on` | Start after named services | `- db` |
| `restart` | Restart policy | `always`, `unless-stopped`, `on-failure` |
| `command` | Override the default CMD | `command: npm run dev` |
| `networks` | Attach to specific networks | `- frontend` |
| `healthcheck` | Define a readiness check | see section 9 |

### Build sub-fields

```yaml
services:
  api:
    build:
      context: ./api        # Directory with Dockerfile
      dockerfile: Dockerfile.dev  # Custom Dockerfile name
      args:
        - NODE_VERSION=18   # Build-time arguments
```

### Restart policies

| Policy | Behaviour |
|---|---|
| `no` | Never restart (default) |
| `always` | Always restart, even on clean exit |
| `unless-stopped` | Restart unless manually stopped |
| `on-failure` | Only restart on non-zero exit code |

---

## 5. Essential Commands

### Starting and stopping

```bash
# Start all services (foreground, shows logs)
docker compose up

# Start in background (detached mode)
docker compose up -d

# Force rebuild images before starting
docker compose up --build

# Start only specific services
docker compose up web db

# Stop all services (keeps containers and volumes)
docker compose stop

# Stop AND remove containers and networks
docker compose down

# Stop, remove containers, networks, AND volumes (wipes all data)
docker compose down -v

# Restart a specific service
docker compose restart web
```

### Viewing status and logs

```bash
# List all services and their status
docker compose ps

# Stream live logs from all services
docker compose logs -f

# Show logs for one service
docker compose logs -f web

# Show last 50 lines
docker compose logs --tail=50 web
```

### Running commands inside containers

```bash
# Open interactive shell in running container
docker compose exec web sh
docker compose exec web bash

# Run a one-off command (creates a new container)
docker compose run web npm install
docker compose run web npm test

# Run as specific user
docker compose exec --user root web sh
```

### Building and managing images

```bash
# Build (or rebuild) images without starting
docker compose build

# Build a specific service
docker compose build web

# Pull latest images for all services
docker compose pull

# Remove stopped containers
docker compose rm
```

---

## 6. Networking

### Default behaviour

Compose automatically creates a **shared private network** for all services in the same YAML file. Services communicate using **the service name as the hostname**.

```
Your App (docker-compose.yaml)
│
├── web     ←→ connects to "db" and "cache" by name
├── db      ←→ hostname: "db", port: 5432
└── cache   ←→ hostname: "cache", port: 6379
```

### Connecting between services

```javascript
// In your Node.js app — use SERVICE NAME, not localhost
const client = new pg.Client({
  host: "db",        // service name from docker-compose.yaml
  port: 5432,
  database: "myapp"
});

const redis = new Redis({
  host: "cache",     // service name
  port: 6379
});
```

> **Common mistake:** Using `localhost` to connect between services. This won't work — use the service name.

### Custom networks

You can create separate networks to isolate services:

```yaml
version: "3.9"

services:
  web:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend          # Only accessible from backend network

  nginx:
    networks:
      - frontend         # Only accessible from frontend

networks:
  frontend:
  backend:
```

---

## 7. Volumes & Data Persistence

Containers are **ephemeral** — all data is lost when a container is removed. Volumes solve this.

### Two types of volumes

#### Named volumes (managed by Docker)

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data   # named_volume:container_path

volumes:
  db_data:    # Docker manages where this is stored on the host
```

- Survives `docker compose down`
- Deleted only with `docker compose down -v`
- Best for: databases, persistent app data

#### Bind mounts (link to host filesystem)

```yaml
services:
  web:
    volumes:
      - ./src:/usr/src/app/src    # host_path:container_path
      - ./config.json:/app/config.json:ro  # :ro = read-only
```

- Changes on your host immediately reflect in the container
- Best for: local development, live code reloading

### Volume summary

| Type | Managed by | Persists after `down`? | Deleted by `down -v`? | Best for |
|---|---|---|---|---|
| Named volume | Docker | Yes | Yes | Databases, app data |
| Bind mount | You (host path) | Yes | No | Dev code sync |

---

## 8. Environment Variables & Secrets

### Option 1: Inline in YAML

```yaml
services:
  web:
    environment:
      - NODE_ENV=production
      - PORT=3000
```

### Option 2: From a `.env` file (recommended)

Create a `.env` file in the same directory as `docker-compose.yaml`:

```bash
# .env
DB_USER=admin
DB_PASS=supersecret
APP_PORT=3000
```

Reference in YAML with `${VAR_NAME}`:

```yaml
services:
  db:
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
  web:
    ports:
      - "${APP_PORT}:3000"
```

> **Important:** Add `.env` to `.gitignore` to keep secrets out of version control.

### Option 3: Separate env file per service

```yaml
services:
  web:
    env_file:
      - ./web.env
      - ./shared.env
```

---

## 9. Startup Order & Health Checks

### `depends_on` — start order

```yaml
services:
  web:
    depends_on:
      - db
      - cache
```

> **Important limitation:** `depends_on` only waits for the *container to start*, not for the *service inside* to be ready. Postgres might take a few seconds to accept connections after its container starts.

### Health checks — true readiness

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s       # Check every 10 seconds
      timeout: 5s         # Fail if no response in 5s
      retries: 5          # Mark unhealthy after 5 failures
      start_period: 30s   # Grace period on first start

  web:
    depends_on:
      db:
        condition: service_healthy   # Wait for health check to pass
```

---

## 10. Real-World Example: Full Stack App

A complete Node.js + PostgreSQL + Redis + Nginx stack:

```yaml
version: "3.9"

services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://admin:${DB_PASS}@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: myapp
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  db_data:
  redis_data:
```

---

## 11. Use Cases

### Local development
- Spin up your full stack locally with one command
- Use bind mounts so code changes are reflected instantly
- All teammates use identical environments — no "works on my machine" problems

### CI/CD pipelines
- Run integration tests against real services (real Postgres, real Redis)
- Bring up the stack, run tests, tear it down — clean every time
- Much lighter than spinning up a Kubernetes cluster for tests

### Microservices prototyping
- Test how multiple services interact before going to production
- Quickly iterate on service dependencies and configurations

### Demos and onboarding
- New developers get the full app running with `git clone` + `docker compose up`
- No manual installation of databases or services

---

## 12. Docker Compose vs Kubernetes vs Docker Swarm

| | Docker Compose | Docker Swarm | Kubernetes |
|---|---|---|---|
| **Purpose** | Local dev / CI | Production clustering (Docker-native) | Production orchestration (industry standard) |
| **Complexity** | Low | Medium | High |
| **Scaling** | No | Yes | Yes |
| **Auto-healing** | No | Yes | Yes |
| **Load balancing** | No (use Nginx) | Built-in | Built-in |
| **Best for** | Dev & testing | Small-medium prod | Large-scale production |

> **Key takeaway:** Docker Compose is NOT a production orchestrator. The Docker-ecosystem equivalent of Kubernetes is **Docker Swarm**, not Compose.

---

## 13. Common Mistakes & Gotchas

### 1. Using `localhost` between services
```bash
# WRONG
DATABASE_URL=postgres://localhost:5432/myapp

# CORRECT — use the service name
DATABASE_URL=postgres://db:5432/myapp
```

### 2. Assuming `depends_on` means "ready"
`depends_on` only waits for the container process to start. Use health checks with `condition: service_healthy` for true readiness.

### 3. Using `down -v` accidentally
`docker compose down -v` **permanently deletes your database data**. Only use it when you want a clean slate.

### 4. Not rebuilding after Dockerfile changes
```bash
# After changing a Dockerfile, always rebuild:
docker compose up --build

# Plain 'up' uses the cached image and ignores your changes
```

### 5. Forgetting to add `.env` to `.gitignore`
Secrets should never be committed to version control.

### 6. Port conflicts on the host
If port 5432 is already in use on your machine, `- "5432:5432"` will fail. Either stop the local service or use a different host port: `- "5433:5432"`.

### 7. Volume path typos in bind mounts
```yaml
volumes:
  - ./src:/usr/src/app   # Relative path — works from where docker-compose.yaml lives
  - /absolute/path:/app  # Absolute path also works
```

---

## 14. Quick Reference Cheat Sheet

### Commands

```bash
docker compose up              # Start all services (foreground)
docker compose up -d           # Start all services (background)
docker compose up --build      # Rebuild images, then start
docker compose down            # Stop and remove containers + networks
docker compose down -v         # Also remove volumes (DELETES DATA)
docker compose ps              # List services and status
docker compose logs -f         # Stream all logs
docker compose logs -f web     # Stream logs for one service
docker compose exec web sh     # Shell into running container
docker compose run web bash    # Shell in new one-off container
docker compose build           # Build images only
docker compose pull            # Pull latest images
docker compose restart web     # Restart one service
```

### YAML skeleton

```yaml
version: "3.9"
services:
  SERVICE_NAME:
    image: IMAGE:TAG          # OR build: PATH_TO_DOCKERFILE
    ports:
      - "HOST:CONTAINER"
    environment:
      - KEY=value
    env_file: .env
    volumes:
      - host_path:container_path
      - named_volume:container_path
    depends_on:
      - OTHER_SERVICE
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "your-check-command"]
      interval: 10s
      retries: 5

volumes:
  named_volume:

networks:
  custom_network:
```

### Key mental models

- **Services talk by name** — `db`, `cache`, `web` are hostnames on the shared network
- **Volumes persist data** — named volumes survive `down`; only `down -v` deletes them
- **Bind mounts sync code** — for local dev, mount your source directory into the container
- **Compose is for dev/CI** — for production at scale, look at Kubernetes or Docker Swarm
- **`.env` keeps secrets safe** — never hardcode passwords in the YAML file

---

*End of study notes. Good luck!*
