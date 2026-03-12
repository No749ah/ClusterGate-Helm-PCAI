<p align="center">
  <img src="logo.png" alt="ClusterGate Logo" width="80" />
</p>

<h1 align="center">ClusterGate — Helm Chart for HPE PCAI</h1>

<p align="center">
  Kubernetes Routing Gateway Platform — manage and expose internal services over public domains.
</p>

---

## Overview

This Helm chart deploys **ClusterGate** on HPE PCAI (formerly EZUA) using the **Import Framework** button. It includes full Istio VirtualService integration, HPE EZUA labels, and is ready for production use.

## Components

| Component      | Description                     | Default Port |
|----------------|---------------------------------|--------------|
| **Backend**    | Express.js API + Proxy Engine   | 3001         |
| **Frontend**   | Next.js Dashboard               | 3000         |
| **PostgreSQL** | Database (16-alpine)            | 5432         |

## Features (v1.3.0)

- **Route Management** — Create, edit, publish, and deactivate proxy routes with multi-step wizard
- **`/r/` Proxy Prefix** — All proxied traffic is served under `/r/*`, cleanly separated from API (`/api/*`) and frontend (`/*`)
- **WebSocket Proxy** — Native WebSocket upgrade support for WS/WSS routes
- **Load Balancing** — Round-robin, weighted, and failover strategies with multiple targets per route
- **Circuit Breaker** — Automatic failure detection (CLOSED/OPEN/HALF_OPEN) with configurable thresholds
- **Route Groups** — Organize routes with shared path prefixes and inherited defaults
- **Request/Response Transforms** — Set/remove headers, rewrite JSON bodies, map status codes
- **Multi-Tenant Organizations** — Organizations with teams, role-based membership, route scoping
- **Two-Factor Authentication** — TOTP-based 2FA with recovery codes for admin accounts
- **Analytics Dashboard** — Latency trends, error rates, traffic heatmap, slowest routes
- **Database Backup & Restore** — Create, download, restore, and manage backups from the UI (Prisma-based JSON export, no pg_dump needed)
- **API Key Management** — Per-route API keys with expiry and usage tracking
- **Health Checks** — Automated uptime monitoring with cron-based checks every 5 minutes
- **Audit Logging** — Full audit trail for all administrative actions
- **Swagger/OpenAPI** — Interactive API docs at `/api/docs`
- **Notifications** — In-app notification system
- **In-App Updates** — One-click updates that pull new container images and roll out deployments
- **Dark Mode** — Default dark theme with light mode toggle

## Quick Start — PCAI Import

1. Download `clustergate-1.2.1.tgz` from this repo
2. In PCAI, click **Import Framework**
3. Upload the `.tgz` file
4. Configure the values (especially secrets) and deploy

## Quick Start — Manual

```bash
helm install clustergate . \
  --namespace clustergate \
  --create-namespace \
  --set backend.image.repository=ghcr.io/no749ah/clustergate-backend \
  --set frontend.image.repository=ghcr.io/no749ah/clustergate-frontend \
  --set backend.secrets.jwtSecret="$(openssl rand -base64 32)" \
  --set backend.secrets.metricsSecret="$(openssl rand -base64 16)" \
  --set postgres.credentials.password="$(openssl rand -base64 24)"
```

## First-Time Setup

After deployment, visit the frontend URL. On first visit you'll see a **Setup Wizard** — no default credentials exist. The wizard creates your initial admin account. The database starts empty and the seed script only validates DB state.

## Traffic Routing

ClusterGate uses path prefixes to separate traffic:

| Path Pattern | Destination | Purpose |
|-------------|-------------|---------|
| `/api/*`    | Backend     | REST API endpoints |
| `/r/*`      | Backend     | Proxied route traffic |
| `/*`        | Frontend    | Next.js dashboard UI |

All proxy routes are created with the `/r/` prefix (e.g., `/r/my-service`). The route creation form enforces this prefix with a locked UI element.

## PCAI / EZUA Integration

The chart follows HPE PCAI conventions:

- **VirtualService** (`virtualservice.yaml`) — Routes traffic through `istio-system/ezaf-gateway` with TLS and HTTP support
- **HPE EZUA Labels** (`_hpe-ezua.tpl`) — All resources are labeled with `hpe-ezua/app` and `hpe-ezua/type: vendor-service`
- **Security Contexts** — All pods run as non-root with specific UID/GID
- **Resource Limits** — CPU/memory limits on all containers (required for EzLicense)

## UI Pages

| Page | Path | Description |
|------|------|-------------|
| Dashboard | `/` | Overview with key metrics |
| Routes | `/routes` | List, search, filter routes by status/tag |
| Route Detail | `/routes/[id]` | View route config, test panel, health checks |
| Create Route | `/routes/new` | Multi-step route creation wizard |
| Edit Route | `/routes/[id]/edit` | Edit existing route configuration |
| Route Groups | `/groups` | Organize routes with shared prefixes |
| Analytics | `/analytics` | Latency, error rate, traffic heatmap, status distribution |
| Logs | `/logs` | Request logs with filtering |
| Organizations | `/organizations` | Multi-tenant organization management |
| Audit | `/audit` | Audit trail for admin actions |
| Users | `/users` | User management |
| Backups | `/backups` | Database backup & restore |
| Notifications | `/notifications` | In-app notifications |
| Settings | `/settings` | System settings, 2FA setup |
| API Docs | `/api/docs` | Swagger/OpenAPI interactive documentation |

## Configuration

All values are configurable via `values.yaml`:

### PCAI / Istio

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ezua.virtualService.enabled` | Enable Istio VirtualService | `true` |
| `ezua.virtualService.endpoint` | Public hostname | `clustergate.${DOMAIN_NAME}` |
| `ezua.virtualService.istioGateway` | Istio gateway reference | `istio-system/ezaf-gateway` |

### Backend

| Parameter | Description | Default |
|-----------|-------------|---------|
| `backend.replicaCount` | Number of backend replicas | `1` |
| `backend.image.repository` | Backend Docker image | `ghcr.io/no749ah/clustergate-backend` |
| `backend.image.tag` | Image tag | `latest` |
| `backend.resources.limits.cpu` | CPU limit | `1000m` |
| `backend.resources.limits.memory` | Memory limit | `1Gi` |
| `backend.secrets.jwtSecret` | JWT signing secret | `CHANGE_ME_JWT_SECRET` |
| `backend.secrets.metricsSecret` | Metrics endpoint secret | `CHANGE_ME_METRICS_SECRET` |
| `backend.env.LOG_LEVEL` | Log level | `info` |
| `backend.env.PROXY_TIMEOUT` | Proxy timeout (ms) | `30000` |
| `backend.env.LOG_RETENTION_DAYS` | Log retention | `90` |
| `backend.autoscaling.enabled` | Enable HPA | `false` |

### Frontend

| Parameter | Description | Default |
|-----------|-------------|---------|
| `frontend.replicaCount` | Number of frontend replicas | `1` |
| `frontend.image.repository` | Frontend Docker image | `ghcr.io/no749ah/clustergate-frontend` |
| `frontend.image.tag` | Image tag | `latest` |
| `frontend.resources.limits.cpu` | CPU limit | `500m` |
| `frontend.resources.limits.memory` | Memory limit | `512Mi` |

### PostgreSQL

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgres.enabled` | Deploy PostgreSQL | `true` |
| `postgres.persistence.enabled` | Enable persistent storage | `true` |
| `postgres.persistence.size` | PVC size | `20Gi` |
| `postgres.persistence.storageClass` | Storage class (empty = default) | `""` |
| `postgres.credentials.username` | Database user | `clustergate` |
| `postgres.credentials.password` | Database password | `CHANGE_ME_PG_PASSWORD` |
| `postgres.credentials.database` | Database name | `clustergate` |

### Optional Features

| Parameter | Description | Default |
|-----------|-------------|---------|
| `networkPolicy.enabled` | Network isolation policies | `false` |
| `monitoring.serviceMonitor.enabled` | Prometheus ServiceMonitor | `false` |
| `rbac.create` | Create RBAC roles | `true` |
| `serviceAccount.create` | Create ServiceAccount | `true` |

## Architecture

```
                    ┌─────────────────────────────────┐
                    │   Istio Gateway (ezaf-gateway)   │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │       VirtualService              │
                    │  /api/* → backend:3001            │
                    │  /r/*   → backend:3001            │
                    │  /*     → frontend:3000           │
                    └──────────┬──────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
     │   Frontend    │ │   Backend    │ │  PostgreSQL  │
     │  (Next.js)    │ │  (Express)   │ │  (16-alpine) │
     │   :3000       │ │   :3001      │ │   :5432      │
     └───────────────┘ └──────┬───────┘ └──────────────┘
                              │                ▲
                              └────────────────┘
```

## Security Notes

- **Change all default secrets** before deploying to production (`jwtSecret`, `metricsSecret`, `postgres.credentials.password`)
- Use `openssl rand -base64 32` to generate strong secrets
- All pods run as non-root with `allowPrivilegeEscalation: false`
- Network policies available (enable with `networkPolicy.enabled: true`)
- RBAC configured for backend service discovery
- **Two-Factor Authentication** available for all user accounts (TOTP + recovery codes)
- PostgreSQL PVC uses `Retain` reclaim policy — data persists even after chart uninstall

## Version History

| Chart Version | App Version | Notes |
|---------------|-------------|-------|
| 1.3.0         | 1.3.0       | WebSocket proxy, load balancing, circuit breaker, route groups, transforms, multi-tenant |
| 1.2.1         | 1.2.1       | Fix analytics, Prisma-based backups, sidebar categories, paginated logs |
| 1.2.0         | 1.2.0       | `/r/` proxy prefix, 2FA, analytics, backups, Swagger, tag filtering |
| 1.1.2         | 1.1.2       | Rate limiting, uptime tracking, UX improvements |
| 1.1.1         | 1.1.1       | Security hardening, frontend bug fixes |
| 1.1.0         | 1.1.0       | Production build fixes, auth improvements |
| 1.0.0         | 1.0.0       | Initial PCAI-compatible release |
