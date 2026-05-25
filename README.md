# minia-docker-compose-nginx

OpenCode skill for reviewing, building, and modifying Docker Compose + Nginx infrastructure.

## Purpose

This repository contains the `minia-docker-compose-nginx` skill definition. It captures the infrastructure baseline for VPS-first deployments that use Docker Compose, Nginx, Certbot, PgBouncer, Redis, PostgreSQL, and observability services.

Use this skill when working on:

- Docker Compose deployment topology
- Nginx gateway configuration
- Backend API and worker runtime separation
- TLS and Certbot sidecars
- Vault-backed staging and production secrets
- PgBouncer, PostgreSQL, Redis, and BullMQ infrastructure
- Deterministic deploy scripts and graceful shutdown
- Observability with Prometheus, Grafana, Loki, Tempo, and OpenTelemetry
- Scale-tier planning from single-VM Compose to Kubernetes

## Files

- `SKILL.md`: The skill manifest and full infrastructure guidance.

## Core Rules

- Nginx is the default public HTTP boundary.
- API and worker containers remain internal.
- Production deploys must verify the new target is healthy before removing the old target.
- Staging and production secrets come from Vault or approved secret injection, not committed `.env` files.
- TLS certificates are managed by Certbot sidecars using shared Docker volumes.
- Static Vite dashboards use the dashboard-specific static profile, not the generic backend proxy template.

## Source Of Truth

The skill references the repository wiki standard at:

`wiki/engineering/tech-stacks/infra-docker-compose-nginx.md`

That wiki page is the broader standard; `SKILL.md` is the agent-facing execution guide.
