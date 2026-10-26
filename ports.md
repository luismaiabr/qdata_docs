# DEV_ENV5 Infrastructure and Network Reference

This document is the source of truth for network topology, exposed ports, and traffic paths across the three stacks in this workspace:

- `qData/qData` (qData API and streaming backend)
- `dashboard` (stats frontend reverse proxy + Angular dev host)
- `nginxmainproxy` (public edge/ingress)

All values below reflect the current checked-in Docker and Nginx configuration.

## 1. Topology at a Glance

```text
Internet
  |
  +--> nginxmainproxy (edge ingress)
	  - uptoken.cloud
	  - stats.uptoken.cloud
	  |
	  +--> qData-nginx (inner proxy, qData stack)
	  |     +--> qData-app (FastAPI :8000, gRPC :50051)
	  |     +--> qData-envoy (:8080, gRPC-Web translation)
	  |
	  +--> dashboard-nginx (stats domain proxy)
		  +--> host.docker.internal:4200 (Angular dev server)
```

Shared docker network used for cross-stack routing: `uptoken_network`.

## 2. Public Ingress (Host-Level Exposed Ports)

These ports are published by `nginxmainproxy` and are the intended public entry points.

| Host Port | TLS | Protocol | Hostname(s) | Routed To | Purpose |
|---|---|---|---|---|---|
| `80` | No | HTTP | `uptoken.cloud`, `stats.uptoken.cloud` | Redirect/ACME | HTTP challenge + redirect to HTTPS |
| `443` | Yes | HTTPS | `uptoken.cloud` | `qData-nginx:443` | REST API and docs |
| `443` | Yes | HTTPS | `stats.uptoken.cloud` | `dashboard-nginx:443` | Stats dashboard |
| `8443` | Yes | HTTPS + HTTP/2 | `uptoken.cloud` | `qData-nginx:8443` | gRPC-Web endpoint for browsers |
| `50052` | Yes | gRPC over HTTP/2 | `uptoken.cloud` | `qData-nginx:50052` | Native gRPC streaming endpoint |

Notes:

- Port `443` is virtual-host routed by SNI/Host header between `uptoken.cloud` and `stats.uptoken.cloud`.
- `8443` is not native gRPC. It is dedicated to gRPC-Web traffic.

## 3. qData Stack Network Contract

### 3.1 Internal Services and Listening Ports

| Service | Container | Listen Port(s) | Role |
|---|---|---|---|
| FastAPI + gRPC app | `qData-app` | `8000`, `50051` | REST and native gRPC business logic |
| Inner Nginx | `qData-nginx` | `80`, `443`, `50052`, `8443` | TLS termination, protocol routing, CORS headers |
| Envoy | `qData-envoy` | `8080` | gRPC-Web to native gRPC translation |
| Certbot | `qData-certbot` | n/a | Certificate issuance/renewal |

### 3.2 Host Port Publishing in qData Compose

Currently `qData-app` publishes:

- `8000:8000`
- `50051:50051`

These are marked as optional direct access in compose comments. Public production clients should use edge ingress (`443`, `8443`, `50052`) through `nginxmainproxy`.

### 3.3 qData Routing Rules (Effective)

| Entry | Next Hop | Final Backend |
|---|---|---|
| `qData-nginx:443` | HTTP proxy | `qData-app:8000` |
| `qData-nginx:50052` | `grpc_pass` | `qData-app:50051` |
| `qData-nginx:8443` | HTTP proxy | `qData-envoy:8080` then `qData-app:50051` |

## 4. Dashboard Stack Network Contract

| Service | Container | Listen Port(s) | Role |
|---|---|---|---|
| Dashboard Nginx | `dashboard-nginx` | `80`, `${FRONT_END_EXTERNAL_PORT:-443}`, `${BACKEND_GRPC_PORT:-8443}` | TLS termination for `stats.uptoken.cloud`, frontend reverse proxy |
| Dashboard Certbot | `dashboard-certbot` | n/a | Cert renewal loop |

Current template behavior:

- `stats.uptoken.cloud:443` proxies to `host.docker.internal:4200` (Angular dev server).
- A gRPC passthrough block exists on `${BACKEND_GRPC_PORT}` and currently targets `host.docker.internal:50051`.

Important:

- `dashboard` compose does not publish host ports directly; traffic reaches it via `nginxmainproxy` on `uptoken_network`.
- `start_all.sh` starts Angular separately via `dashboard/run_dash.sh`, which serves on local port `4200`.

## 5. End-to-End Traffic Flows

### 5.1 REST API (Production)

1. Client calls `https://uptoken.cloud/...` on host port `443`.
2. `nginxmainproxy` forwards to `qData-nginx:443`.
3. `qData-nginx` proxies to `qData-app:8000`.

### 5.2 Native gRPC (Production)

1. Client connects to `uptoken.cloud:50052` with TLS.
2. `nginxmainproxy` forwards gRPC to `qData-nginx:50052`.
3. `qData-nginx` forwards to `qData-app:50051`.

### 5.3 gRPC-Web (Browser)

1. Browser client connects to `https://uptoken.cloud:8443`.
2. `nginxmainproxy` forwards to `qData-nginx:8443`.
3. `qData-nginx` forwards to `qData-envoy:8080`.
4. Envoy translates gRPC-Web and forwards native gRPC to `qData-app:50051`.

### 5.4 Stats Frontend

1. Browser opens `https://stats.uptoken.cloud` on host port `443`.
2. `nginxmainproxy` routes by hostname to `dashboard-nginx:443`.
3. `dashboard-nginx` proxies to `host.docker.internal:4200`.

## 6. Security and TLS Boundaries

- Edge TLS certificates are served by `nginxmainproxy` for both domains.
- qData inner Nginx also uses certificates in its own stack.
- `proxy_ssl_verify off` is currently used in edge-to-inner HTTPS proxy hops.
- Long-lived stream timeouts are set to 3600s for gRPC/gRPC-Web forwarding.

## 7. Operations and Validation

### 7.1 Start Order

`start_all.sh` creates `uptoken_network`, then starts:

1. `qData/qData`
2. `dashboard`
3. `nginxmainproxy`
4. Angular dev server (`dashboard/run_dash.sh`)

### 7.2 Quick Port Checks

```bash
# Host listeners
ss -ltnp | grep -E ':80|:443|:8443|:50052|:4200|:8000|:50051'

# Containers and published ports
docker ps --format 'table {{.Names}}\t{{.Ports}}'

# Verify edge routes
curl -I https://uptoken.cloud
curl -I https://stats.uptoken.cloud
```

### 7.3 gRPC Reachability Checks

```bash
# Native gRPC (TLS)
grpcurl -insecure uptoken.cloud:50052 list

# gRPC-Web endpoint should be used by grpc-web clients, not grpcurl native mode
curl -vk https://uptoken.cloud:8443
```

## 8. Canonical Endpoint Matrix

Use these endpoints for clients unless you are doing local debugging:

| Use Case | Endpoint |
|---|---|
| REST API | `https://uptoken.cloud` |
| Swagger/OpenAPI | `https://uptoken.cloud/docs` |
| Native gRPC (TLS) | `uptoken.cloud:50052` |
| Browser gRPC-Web | `https://uptoken.cloud:8443` |
| Stats dashboard | `https://stats.uptoken.cloud` |

For local direct debugging only:

- `http://localhost:8000` (direct app REST)
- `localhost:50051` (direct app gRPC insecure)
- `http://localhost:4200` (Angular dev server)

