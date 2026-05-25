---
name: minia-docker-compose-nginx
description: "Use when building, reviewing, or modifying infrastructure for Docker Compose + Nginx deployments, backend API runtime topology, Nginx gatewaying, Vault-backed secrets, TLS/Certbot, PgBouncer, Redis, observability stack, deterministic deploys, graceful shutdown, scale tiers, or Kubernetes upgrade planning."
---

# Infrastructure: Docker Compose + Nginx

This skill applies the repository wiki standard for VPS-first infrastructure, backend API runtime topology, Nginx gatewaying, secrets, TLS, observability, deterministic deployments, graceful shutdown, scale tiers, and Kubernetes upgrade paths.

Primary source of truth: `wiki/engineering/tech-stacks/infra-docker-compose-nginx.md`.

Related standards:
- `wiki/engineering/tech-stacks/backend-bun-elysia.md`
- `wiki/engineering/tech-stacks/web-react-vite-dashboard.md`
- `wiki/engineering/tech-stacks/web-astro-landing.md`
- `wiki/engineering/tech-stacks/security-web-app-baseline.md`
- `wiki/engineering/tech-stacks/ci-testing-typescript-react.md`

## Required Skill And Docs Loading

Before implementing infrastructure changes, load and apply these skills when available:

- `backend-bun-elysia` when infra touches API, worker, PostgreSQL 18+, PgBouncer, Redis 8+, BullMQ, OpenTelemetry, or backend Nginx routing.
- `web-react-vite-dashboard` when infra touches authenticated dashboard static deployment, Nginx static serving, Certbot, or dashboard Docker images.
- `web-astro-landing` only when comparing landing deployment concerns or coordinating with Cloudflare/static landing deployments.

For setup or configuration-specific work, consult current documentation for Docker Compose, Nginx, Certbot/Let's Encrypt, HashiCorp Vault, PgBouncer, Redis, Prometheus, Grafana, Loki, and Tempo before implementing.

If a skill or documentation source is unavailable, continue using the rules in this skill and state the gap in the final response.

## Stack Invariants

- Infrastructure baseline is Docker Compose + Nginx.
- The default target is VPS-first deployment for backend APIs and approved runtime services.
- Nginx is the only host-facing HTTP service.
- API and worker containers stay internal.
- Static Vite dashboard deployment uses the dashboard-specific static profile, not the generic backend API proxy template.
- Backend API deployment runs separate API and worker processes behind Nginx.
- Deployment procedures must be deterministic and tested in staging.
- Production deploys must prove the new target is healthy before removing the old target.
- Main Nginx gateways use `fholzer/nginx-brotli:<pinned-version>` directly or through a minimal wrapper image.
- Brotli is enabled by default with `NGINX_BROTLI_ENABLED=on`; gzip fallback is enabled with `NGINX_GZIP_ENABLED=on`.
- Production Let's Encrypt issuance and renewal use Certbot sidecars with shared Docker volumes.

## Environment And Secrets

Secrets are tiered by environment:

| Environment | Secrets source | Mechanism |
| --- | --- | --- |
| Local development | `.env` | Docker Compose `env_file` |
| Staging | HashiCorp Vault | Vault Agent or CI injection |
| Production | HashiCorp Vault | Vault Agent or CI injection |

Rules:
- Never hardcode secrets, API keys, DSNs, or database URLs.
- Never bake secrets into Docker images through `ENV` or build-time `ARG`.
- Vault OSS is the mandatory secrets manager for staging and production.
- `.env.example` is committed and documents all required variables.
- Staging and production `DATABASE_URL` points to PgBouncer, not directly to PostgreSQL 18+.
- Baseline env categories include app config, database, JWT/JWKS, CORS, Redis/BullMQ, object storage, OpenTelemetry, and monitoring.

## Backend API Runtime Profile

Standalone Bun + Elysia backend APIs deploy as separate API and worker processes behind Nginx.

Required baseline services:
- `nginx`
- `api`
- `worker`
- `postgres 18+`
- `pgbouncer`
- `redis 8+`
- `otel-collector`
- `prometheus`
- `loki`
- `tempo`
- `grafana`
- `certbot`
- `certbot-renew`

Optional services:
- `minio`
- `mailpit`
- `bull-board`

Baseline Compose shape:

```yaml
services:
  api:
    image: example-api:${TAG}
    expose:
      - "3000"
    stop_grace_period: 30s
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  worker:
    image: example-api:${TAG}
    command: ["bun", "run", "worker"]
    stop_grace_period: 60s
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  nginx:
    build:
      context: ./nginx
    image: example-api-nginx:${TAG}
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - letsencrypt:/etc/letsencrypt:ro
      - certbot-webroot:/var/www/certbot:ro
    environment:
      NGINX_UPSTREAM_HOST: api
      NGINX_UPSTREAM_PORT: 3000
      NGINX_SERVER_NAME: api.example.com
      NGINX_BROTLI_ENABLED: "on"
      NGINX_GZIP_ENABLED: "on"
    depends_on:
      - api

  certbot:
    image: certbot/certbot:<pinned-version>
    volumes:
      - letsencrypt:/etc/letsencrypt
      - certbot-webroot:/var/www/certbot

  certbot-renew:
    image: certbot/certbot:<pinned-version>
    volumes:
      - letsencrypt:/etc/letsencrypt
      - certbot-webroot:/var/www/certbot

volumes:
  letsencrypt:
  certbot-webroot:
```

Rules:
- `api` and `worker` use the same application image with different commands.
- `api` handles HTTP traffic only.
- `worker` processes BullMQ jobs only.
- Do not run BullMQ workers inside the API process in production.
- `api` and `worker` both connect to PgBouncer, Redis 8+, object storage, and OpenTelemetry Collector.
- PostgreSQL 18+ is the baseline durable database service.
- Redis 8+ is mandatory for BullMQ, rate limiting, permission cache, and short-lived coordination.
- PgBouncer runs in transaction mode and connects to PostgreSQL 18+.
- The Nginx gateway image must be based on `fholzer/nginx-brotli:<pinned-version>` with Brotli enabled by default.
- Certbot sidecars own Let's Encrypt issuance and renewal; do not install Certbot into Nginx or API images.
- `api` `stop_grace_period` is at least `30s`.
- `worker` `stop_grace_period` is at least `60s`.
- Production media storage should prefer managed S3-compatible storage.
- Self-hosted MinIO requires backup and restore procedures.
- API containers use `expose`, not host `ports`.
- Containers use `security_opt: no-new-privileges:true` where supported.
- Containers drop capabilities by default.

## Nginx Backend API Template

The backend API Nginx template is for JSON APIs, not static dashboard SPAs. It does not include CSP nonce propagation, robots/sitemap handling, static asset caching, or SSR cache rules.

Key rules:
- Nginx applies baseline non-CSP security headers to API responses.
- Backend API CSP nonce handling is omitted by default because JSON APIs do not serve HTML.
- `/health` may be public to Nginx or load balancers.
- `/ready` is internal-only.
- `/metrics` is internal-only.
- `/docs` and `/docs/json` are blocked by baseline public Nginx unless the project intentionally auth-gates them.
- `limit_req` is coarse per-IP edge protection only.
- Elysia/Redis owns product rate limits.
- `NGINX_CLIENT_MAX_BODY_SIZE` defaults to `1m`.
- Media uses presigned object-storage uploads.
- Set upstream timeouts such as `proxy_connect_timeout 5s`, `proxy_send_timeout 30s`, and `proxy_read_timeout 30s`.
- Nginx runtime variables include `NGINX_SERVER_NAME`, `NGINX_UPSTREAM_HOST=api`, `NGINX_UPSTREAM_PORT=3000`, Brotli/Gzip toggles, worker/connection limits, body size, rate limit RPS, and TLS/HSTS switch.
- Port `80` must serve `/.well-known/acme-challenge/` from the shared Certbot webroot for ACME HTTP-01 validation.

## Static SPA Dashboard Profile

Static Vite dashboards do not use the generic proxy-to-API template. They serve `dist/` directly from Nginx and follow `web-react-vite-dashboard`.

Dashboard invariants:
- Production deployment uses Docker Compose.
- Nginx serves Vite `dist/` directly from `/usr/share/nginx/html`.
- No Bun, Node, Vite preview, custom static server, or internal dashboard app server runs in production runtime.
- TLS terminates at Nginx with Let's Encrypt certificates in Docker volumes.
- Certbot sidecars handle issuance and renewal.
- Static hashed assets use immutable cache headers.
- `index.html` uses no-cache or must-revalidate.

## Deterministic Deployments

A deployment script that checks only `http://localhost/ready` through Nginx is insufficient because the old instance can make that health check pass while the new instance is unhealthy.

Acceptable strategies:
- Blue/green services such as `api_blue` and `api_green`, switching Nginx only after the target color is healthy.
- Container-specific readiness checks against the newly-created container before old containers are removed.
- Kubernetes rolling updates at Tier 3.

Rules:
- Never run `docker compose restart` as the production deploy procedure.
- Never run direct `docker compose up` as the production deploy procedure.
- Use the project deployment script and test it in staging.
- The script must prove the new container or target is ready before removing the old target.
- API and worker processes must handle SIGTERM and drain cleanly.

## Graceful Shutdown

API shutdown sequence:
1. `api` receives SIGTERM.
2. `isShuttingDown = true`.
3. `/ready` returns `503`.
4. Nginx stops routing new requests.
5. API drains in-flight requests.
6. Active DB transactions commit or rollback.
7. SQL, Redis, BullMQ, logs, and telemetry close before `stop_grace_period` expires.

Worker shutdown sequence:
1. `worker` receives SIGTERM.
2. Worker stops taking new BullMQ jobs.
3. Active jobs finish or checkpoint within timeout.
4. BullMQ workers and queues close before shared Redis connections close.
5. SQL, Redis, and telemetry close before `stop_grace_period` expires.

## Scale Tiers

| Tier | Concurrency | Orchestration | Notes |
| --- | ---: | --- | --- |
| Tier 0 | 100 | Single VM Docker Compose | MVP/prototype, co-located services |
| Tier 1 | 2K | Docker Compose | Nginx, API, worker, PgBouncer, PostgreSQL 18+, Redis 8+, Grafana stack |
| Tier 2 | 10K | Docker Compose or light K8s | More API instances, workers, DB read replicas, observability stack |
| Tier 3 | 100K | Kubernetes | API HPA, worker HPA, PDB, Redis 8+ Cluster, read replicas, OS tuning, observability stack |

Tier 3 requirements:
- API Deployment with `maxUnavailable: 0`, `minReadySeconds`, readiness/liveness probes, `preStop` sleep, and sufficient `terminationGracePeriodSeconds`.
- Worker Deployment using the same image and `command: ["bun", "run", "worker"]`.
- API and worker HPAs/PDBs sized independently.
- Redis 8+ Cluster for 100K+ concurrency.
- OS file descriptor tuning on every host running API containers.
- Tier 3 architecture routes API traffic to Redis/BullMQ, and workers consume from Redis/BullMQ.
- API pods do not call worker pods directly.

## Security Baseline

- Every HTTP surface includes applicable security headers at the server, gateway, or middleware layer.
- Backend JSON APIs that do not serve HTML do not require nonce CSP in the API Nginx template.
- Backend JSON APIs still require non-CSP security headers.
- If a backend endpoint serves HTML, CSP applies to that response.
- Production CSP must not contain wildcard `*` sources.
- Production `script-src` must not use `unsafe-inline` unless reviewed.
- `connect-src` lists only approved API, monitoring, analytics, storage, or websocket endpoints actually used.
- CORS allowlists are explicit and environment-specific.
- Do not use wildcard origins with credentials.
- Do not reflect arbitrary `Origin` values.
- Preflight responses must not expose internal or debug headers.
- Health/readiness endpoints must not expose env vars, secret names, credentials, internal hostnames, image tags, or detailed config.
- OpenTelemetry spans, metrics, and logs must not include PII, bearer tokens, refresh tokens, API keys, cookies, raw request bodies, card data, emails, phone numbers, or free-text user content unless explicitly allowlisted and scrubbed.
- Metrics avoid high-cardinality labels such as user ID, workspace ID, request ID, resource ID, raw URL path, email, or phone.
- Production source maps are not publicly served by default.
- Pin exact production dependencies and Docker tags.
- Commit and review Bun lockfile changes.

## Verification Checklist

When reviewing or changing infra, verify the relevant items:

- Compose services match the intended runtime profile.
- Host-facing ports are limited to Nginx or explicitly approved public services.
- API containers use internal `expose`, not host `ports`.
- `api` and `worker` use the same image with distinct commands.
- BullMQ workers are not running inside the API process.
- PgBouncer is present and staging/production `DATABASE_URL` values target it.
- PgBouncer is in transaction mode.
- PostgreSQL 18+ is used for durable database services.
- Redis 8+ is present for BullMQ, rate limiting, permission cache, and coordination.
- Object storage is configured when media is used.
- Nginx uses `fholzer/nginx-brotli:<pinned-version>` and Brotli is on by default.
- Self-hosted MinIO has backup and restore procedures.
- Nginx blocks public `/ready`, `/metrics`, `/docs`, and `/docs/json` unless auth-gated by project decision.
- `/health` is safe for public load-balancer checks.
- Nginx API body-size and upstream timeouts match the standard.
- TLS certificates live in volumes, not image layers.
- Certbot issuance and renewal sidecars use shared volumes and renewal reloads Nginx after success.
- Deployment script proves the new target is healthy before removing the old target.
- API and worker graceful shutdown paths are configured and tested.
- Observability services exist where required.
- Logs, spans, and metrics avoid secrets, PII, and high-cardinality labels.
- `.env.example` documents required variables.
- No secrets are baked into images or committed to source.

## Anti-Patterns

Do not introduce these patterns:

- Exposing API container ports to the host.
- Running BullMQ workers inside the API process in production.
- Using Redis as durable source of truth.
- Using stock `nginx`, `nginx:latest`, `fholzer/nginx-brotli:latest`, or disabling Brotli by default.
- Installing Certbot into the API or Nginx image.
- Baking secrets into images with `ENV` or build-time `ARG`.
- Using `.env` as staging or production secrets management.
- Direct `docker compose restart` as a production deploy procedure.
- Direct `docker compose up` as a production deploy procedure.
- Checking readiness only through an old Nginx target.
- Public `/ready`, `/metrics`, or `/docs/json` by default.
- Using the generic backend proxy template for static dashboard deployment.
- Using Nginx `limit_req` as authoritative product rate limiting.
- Mounting production dashboard `dist/` from the host.
- Self-hosted MinIO without backup and restore procedures.
- Wildcard credentialed CORS.
- Public production source maps.
- Health/readiness responses exposing secrets, env vars, internal hostnames, image tags, or detailed config.
- Telemetry labels containing user IDs, workspace IDs, raw URLs, raw IPs, emails, or phone numbers.

## Implementation Workflow

When modifying or creating infrastructure:

1. Identify whether the change affects backend API runtime, dashboard static runtime, Nginx, secrets, TLS, database connectivity, Redis/BullMQ, observability, deployment scripts, graceful shutdown, or scale tier.
2. Load the relevant stack skill before editing application-adjacent infra.
3. Keep host-facing exposure minimal; Nginx is the default public HTTP boundary.
4. Keep API and worker runtime responsibilities separate.
5. Route staging and production database traffic through PgBouncer to PostgreSQL 18+.
6. Keep staging and production secrets in Vault or approved secret injection, not `.env` files or images.
7. Prefer deterministic deploy scripts over ad hoc Compose commands.
8. Ensure the new deployment target is checked directly before removing the old target.
9. Preserve graceful shutdown and stop grace periods.
10. Validate Nginx exposure for health, readiness, metrics, docs, and admin surfaces.
11. Validate TLS/certificate renewal and volume ownership.
12. Add or update operational documentation when deployment or recovery procedures change.
13. Run the smallest relevant verification commands available in the project.
14. In the final response, report which infra gates were verified and which were unavailable.
