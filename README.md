# Production-Ready Frappe/ERPNext v15 Docker Compose Stack

This repository contains a high-availability, microservice-architected deployment stack for **Frappe/ERPNext v15**. It is explicitly optimized for IoT, computer vision integration, and smart home automation workflows, separating high-frequency stateful events from business logic execution.

## 🚀 Key Architectural Highlights

* **Decoupled WebSocket Layer:** Runs `erpnext-websocket` (`WORKER_TYPE: websocket`) as an isolated container on port `9000`. This isolates real-time IoT/OpenCV telemetry events from Gunicorn HTTP worker threads, mitigating MariaDB thread starvation.
* **Database Optimization for Frappe v15:** Fixed table-agnostic migration bugs with MariaDB `10.11` by explicitly tweaking strict engines (`--innodb-read-only-compressed=OFF`, `--innodb-strict-mode=OFF`) and packet buffers (`--max-allowed-packet=64M`).
* **Zero-Host Configuration Frontend:** Utilizes the official `frappe/erpnext:v15` dynamic entrypoint for Nginx (`nginx-entrypoint.sh`). Static assets are seamlessly mounted and served via an internal Docker volume, eliminating broken symlinks (`Device or resource busy`) on the host system.
* **Deadlock-Free Orchestration:** Services depend on the `backend` using `condition: service_started` instead of `service_healthy`. This approach breaks initialization deadlocks before the database schema is generated.

## 📦 System Architecture

```text
                    [ Client Request ]
                            │
                            ▼
                     ┌─────────────┐
                     │  erp-nginx  │
                     └──────┬──────┘
                            │
             ┌──────────────┴──────────────┐
             ▼                             ▼
       /socket.io                      / (HTTP)
     ┌─────────────┐               ┌─────────────┐
     │  websocket  │               │   backend   │
     │ (Node.js)   │               │ (Gunicorn)  │
     └─────────────┘               └──────┬──────┘
                                           │
                                   ┌───────┴───────┐
                                   ▼               ▼
                             ┌───────────┐   ┌───────────┐
                             │  MariaDB  │   │   Redis   │
                             └───────────┘   └───────────┘
```

## 🛠️ Quick Start

### 1. Clone & Environment Setup
```bash
git clone <your-repository-url>
cd erp-stack-v2/docker
cp .env.example .env
# Edit .env and change credentials!
```

### 2. Launch Infrastructure
```bash
docker compose up -d
sleep 15
```

### 3. Initialize Bench & Frappe Site
Execute these automated wrapper commands to set upstream routing and deploy the schema:
```bash
# Configure internal DNS mapping
docker compose exec backend bench set-config -g db_host mariadb
docker compose exec backend bench set-config -g redis_cache redis://redis-cache:6379
docker compose exec backend bench set-config -g redis_queue redis://redis-queue:6379
docker compose exec backend bench set-config -g redis_socketio redis://redis-queue:6379

# Create Site & Database Scheme
docker compose exec backend bench new-site visiontech \
  --mariadb-root-password "your_root_password_from_env" \
  --admin-password "your_admin_password_from_env" \
  --mariadb-user-host-login-scope "%" \
  --force

# Set Default Multi-tenant Routing
docker compose exec backend bench use visiontech
docker compose exec backend bench --site visiontech enable-scheduler
```

## 📊 Verification
Verify that everything is operational:
```bash
curl -I http://localhost/login
# Should return HTTP/1.1 200 OK
```