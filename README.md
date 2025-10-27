# DevOps Intern Stage 2 — Blue/Green Deployment with Nginx Upstreams (Auto-Failover + Manual Toggle)

### Author: Kokoscripts

**Project Type:** DevOps / Cloud Infrastructure

**Tech Stack:** Docker, Docker Compose, Nginx, Node.js

**Objective:** High-availability deployment with automatic failover and manual traffic switching between Blue and Green environments.

---

##  Overview

This project demonstrates a **Blue/Green deployment architecture** implemented with **Docker Compose** and **Nginx** acting as a **reverse proxy and load balancer**.

It deploys **two identical Node.js services**—**Blue** (active) and **Green** (standby)—behind Nginx. The setup ensures **zero downtime** and **automatic failover** during service failures using **health-based upstream failover policies**.

The goal is to:

* Route client traffic to the **active (Blue)** environment by default.
* Automatically switch to **Green** if Blue becomes unhealthy.
* Provide **manual toggle capability** via an environment variable (`ACTIVE_POOL`).
* Maintain **response consistency** with proper HTTP headers.

---

## 🏗️ Architecture

### Components

| Component     | Role                          | Port | Description                                                            |
| ------------- | ----------------------------- | ---- | ---------------------------------------------------------------------- |
| **Nginx**     | Reverse proxy & load balancer | 8080 | Routes requests to Blue/Green based on health status and configuration |
| **App Blue**  | Active Node.js service        | 8081 | Handles traffic by default; exposes health and chaos endpoints         |
| **App Green** | Standby Node.js service       | 8082 | Receives traffic only if Blue fails or manually deactivated            |

### System Flow

```
                  ┌──────────────────────────┐
                  │        CLIENT            │
                  └────────────┬─────────────┘
                               │
                               ▼
                     ┌────────────────┐
                     │     NGINX      │   ← Reverse Proxy
                     └────────────────┘
                        ▲          ▲
                        │          │
        ┌───────────────┘          └───────────────┐
        │                                          │
┌────────────────────┐                    ┌────────────────────┐
│    Blue Service    │                    │    Green Service   │
│ (http://localhost:8081) │            │ (http://localhost:8082) │
└────────────────────┘                    └────────────────────┘
          ↑  ↑                                      ↑  ↑
   /version /chaos/start                     /version /chaos/start
```

---

##  Features

✅ **Zero Downtime Failover:** If Blue fails, Nginx instantly switches to Green.
✅ **Automatic Health Checks:** Nginx monitors `/healthz` endpoints for availability.
✅ **Retry on Failure:** Nginx retries failed/timeout requests on the backup pool.
✅ **Manual Toggle:** Switch active environment using `ACTIVE_POOL` variable.
✅ **Header Preservation:** App headers `X-App-Pool` and `X-Release-Id` are preserved end-to-end.
✅ **Parameterization via `.env`:** Full configuration control for CI/CD.
✅ **Chaos Testing:** Built-in simulation of service failures.

---

##  Project Structure

```
.
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf.template
│   └── default.conf (generated after envsubst)
├── .env
├── README.md
└── scripts/
    └── reload_nginx.sh
```

---

##  Environment Variables (.env)

| Variable           | Description                            | Example                              |
| ------------------ | -------------------------------------- | ------------------------------------ |
| `BLUE_IMAGE`       | Docker image for Blue service          | `registry.hng.tech/blue-app:latest`  |
| `GREEN_IMAGE`      | Docker image for Green service         | `registry.hng.tech/green-app:latest` |
| `ACTIVE_POOL`      | Active environment (`blue` or `green`) | `blue`                               |
| `RELEASE_ID_BLUE`  | Identifier for Blue release            | `v1.0.0`                             |
| `RELEASE_ID_GREEN` | Identifier for Green release           | `v1.0.1`                             |
| `PORT`             | (Optional) Application port            | `8080`                               |

---

##  Docker Compose Configuration

### docker-compose.yml

The setup defines three services:

1. **nginx** – Reverse proxy that routes traffic.
2. **app_blue** – Primary application container.
3. **app_green** – Secondary/backup application container.

All services run on a shared network (`app_net`).

Example snippet:

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx_gateway
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app_blue
      - app_green

  app_blue:
    image: ${BLUE_IMAGE}
    container_name: app_blue
    environment:
      - RELEASE_ID=${RELEASE_ID_BLUE}
      - PORT=8081
    ports:
      - "8081:8081"

  app_green:
    image: ${GREEN_IMAGE}
    container_name: app_green
    environment:
      - RELEASE_ID=${RELEASE_ID_GREEN}
      - PORT=8082
    ports:
      - "8082:8082"

networks:
  default:
    name: app_net
```

---

##  Nginx Configuration Logic

The **Nginx upstream configuration** is dynamically templated from the environment variable `ACTIVE_POOL`.
During container startup, it uses `envsubst` to render `nginx.conf` based on which pool is active.

### nginx.conf.template

```nginx
worker_processes 1;
events { worker_connections 1024; }

http {
    upstream app_backend {
% if ACTIVE_POOL == "blue" %
        server app_blue:8081 max_fails=2 fail_timeout=3s;
        server app_green:8082 backup;
% else %
        server app_green:8082 max_fails=2 fail_timeout=3s;
        server app_blue:8081 backup;
% endif %
    }

    server {
        listen 80;
        location / {
            proxy_pass http://app_backend;
            proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
            proxy_connect_timeout 1s;
            proxy_read_timeout 2s;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass_header X-App-Pool;
            proxy_pass_header X-Release-Id;
        }
    }
}
```

---

## ⚡ Manual Toggle (Switch Active Environment)

To manually switch traffic between **Blue** and **Green** environments:

```bash
export ACTIVE_POOL=green
docker-compose up -d --force-recreate nginx
```

Or reload Nginx dynamically:

```bash
docker exec nginx_gateway nginx -s reload
```

This swaps the active pool without downtime.

---

##  Chaos Testing (Simulate Failure)

Each service exposes endpoints to simulate errors and timeouts:

| Endpoint                    | Method | Description                         |
| --------------------------- | ------ | ----------------------------------- |
| `/chaos/start?mode=error`   | POST   | Simulate internal error (HTTP 500s) |
| `/chaos/start?mode=timeout` | POST   | Simulate timeout                    |
| `/chaos/stop`               | POST   | End chaos mode                      |
| `/healthz`                  | GET    | Return service health               |

### Example

```bash
# Simulate Blue failure
curl -X POST http://localhost:8081/chaos/start?mode=error

# Test automatic failover
curl http://localhost:8080/version

# Stop chaos
curl -X POST http://localhost:8081/chaos/stop
```

---

##  Expected Behavior

| Scenario                     | Expected Output                                 |
| ---------------------------- | ----------------------------------------------- |
| Normal (Blue active)         | `X-App-Pool: blue`, `X-Release-Id: v1.0.0`      |
| Blue failure (chaos started) | `X-App-Pool: green`, `X-Release-Id: v1.0.1`     |
| Recovery (chaos stopped)     | Manual switch or reload returns traffic to Blue |

**No requests should return non-200 responses** even during failover.

---

##  Health Verification Commands

```bash
# Check which pool is active
curl -I http://localhost:8080/version | grep X-App-Pool

# Validate failover logic
watch -n 1 "curl -s -I http://localhost:8080/version | grep X-App-Pool"
```

---

##  CI/CD Integration (Conceptual)

In a real-world CI pipeline:

1. **Build phase** — CI sets environment variables (`BLUE_IMAGE`, `GREEN_IMAGE`, etc.).
2. **Deploy phase** — Compose brings up new version (Green).
3. **Verify phase** — Health checks validate the Green service.
4. **Switch phase** — `ACTIVE_POOL` updates to Green via config templating.
5. **Rollback** — On failure, revert `ACTIVE_POOL` to Blue.

This mimics a **safe, zero-downtime deployment flow**.

---

##  Fail Conditions

The deployment fails if:

* Any `GET /version` request returns non-200.
* Headers `X-App-Pool` or `X-Release-Id` mismatch after chaos.
* No switch occurs within 10 seconds of Blue failure.
* Requests are routed directly to app ports (bypassing Nginx).

---

##  Cleanup

```bash
docker-compose down -v
```

---

## Key DevOps Concepts Demonstrated

* **Blue/Green Deployments** — Parallel environments for risk-free upgrades
* **Nginx Failover & Retry** — Intelligent traffic routing under failure
* **Chaos Engineering** — Fault simulation to test resilience
* **Environment Parameterization** — Config-driven deployments
* **Immutable Infrastructure** — No rebuild of app images
* **CI/CD Readiness** — Pipeline-ready structure

---

## Future Improvements

* Integrate **Prometheus + Grafana** for monitoring request flow.
* Use **Consul or etcd** for dynamic service discovery.
* Extend with **GitHub Actions** for automated testing and config templating.
* Implement **blue-green rollback automation**.

---

## Conclusion

This project models **production-grade deployment resilience**—balancing simplicity (Docker Compose) and reliability (Nginx failover). It ensures **high availability**, **fault tolerance**, and **manual control** through configuration-driven environments.

*"If you can fail safely, you can scale confidently."*

Would you like me to include a **sample `nginx.conf.template` and `reload_nginx.sh` script** inside the README for completeness (so your repo looks fully deploy-ready)?
