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

| Component   | Description                          | Default Port |
|-------------|--------------------------------------|--------------|
| **Backend** | Express.js API + Proxy Engine        | 3001         |
| **Frontend**| Next.js Dashboard                    | 3000         |
| **PostgreSQL** | Database (16-alpine)              | 5432         |

## Quick Start — PCAI Import

1. Download `clustergate-1.0.0.tgz` from this repo
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
  --set backend.secrets.jwtSecret="your-secure-jwt-secret" \
  --set backend.secrets.metricsSecret="your-metrics-secret" \
  --set postgres.credentials.password="your-db-password"
```

## PCAI / EZUA Integration

The chart follows HPE PCAI conventions:

- **VirtualService** (`virtualservice.yaml`) — Routes traffic through `istio-system/ezaf-gateway` with TLS and HTTP support
- **HPE EZUA Labels** (`_hpe-ezua.tpl`) — All resources are labeled with `hpe-ezua/app` and `hpe-ezua/type: vendor-service`
- **Security Contexts** — All pods run as non-root with specific UID/GID
- **Resource Limits** — CPU/memory limits on all containers (required for EzLicense)

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
- All pods run as non-root with `allowPrivilegeEscalation: false`
- Network policies available (enable with `networkPolicy.enabled: true`)
- RBAC configured for backend service discovery

## Version

| Chart Version | App Version | Notes |
|---------------|-------------|-------|
| 1.0.0         | 1.0.0       | Initial PCAI-compatible release |
