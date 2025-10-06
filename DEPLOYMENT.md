# Deployment Guide

This document describes how to deploy Axiom Market Matrix in production and staging environments using Docker and Docker Compose. It covers environment configuration, secrets management, networking, observability, scaling, and operational playbooks.

## 1. Environments

- development: Local workstation with hot-reload, verbose logging
- staging: Pre-production for integration and load testing
- production: Highly available, monitored, and restricted-access

## 2. Prerequisites

- Docker 24+ and Docker Compose v2+
- Domain names and DNS for public endpoints (e.g., api.example.com)
- TLS certificates (Let's Encrypt or managed CA)
- API keys for data providers (Alpha Vantage, Binance, Polygon)
- PostgreSQL and Redis volumes sized per retention policy

## 3. Configuration (.env)

Create a .env file at the repository root. Example:

ENVIRONMENT=production
LOG_LEVEL=INFO
SECRET_KEY=generate-a-secure-random-key

# Database
POSTGRES_DB=axiom_trading
POSTGRES_USER=axiom
POSTGRES_PASSWORD=change-this

# Redis
REDIS_PASSWORD=change-this

# API Keys
ALPHA_VANTAGE_API_KEY=... 
BINANCE_API_KEY=...
BINANCE_API_SECRET=...
POLYGON_API_KEY=...

# Trading
TRADING_MODE=paper
MAX_POSITION_SIZE=10000
RISK_PER_TRADE=0.02

# Grafana admin
GRAFANA_USER=admin
GRAFANA_PASSWORD=admin

Never commit .env with secrets to version control.

## 4. Build and Run

docker compose build --no-cache

docker compose up -d

Check service health:

- App: http://localhost:8000/health
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

Logs:

docker compose logs -f app

docker compose logs -f worker

## 5. Reverse Proxy (optional, production)

Use Traefik or Nginx in front of the app for TLS termination and routing.

Traefik example labels (add under app service in docker-compose.yml):

labels:
  - traefik.enable=true
  - traefik.http.routers.axiom.rule=Host(`api.example.com`)
  - traefik.http.routers.axiom.entrypoints=websecure
  - traefik.http.routers.axiom.tls.certresolver=letsencrypt
  - traefik.http.services.axiom.loadbalancer.server.port=8000

## 6. Database Migrations

Run Alembic migrations on first deploy and when models change:

alembic upgrade head

You can add an init container or a startup command in Compose:

command: bash -lc "alembic upgrade head && uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --workers 4"

## 7. Observability

- Prometheus scrapes metrics from app at /metrics (enable in app)
- Grafana connects to Prometheus and uses provisioned dashboards
- Structured logging (JSON) recommended in production

Example FastAPI metrics setup:

from prometheus_client import Counter, Histogram
REQUEST_COUNT = Counter("axiom_requests_total", "Total requests", ["method", "path", "status"])
REQUEST_LATENCY = Histogram("axiom_request_latency_seconds", "Request latency", ["method", "path"])

## 8. Security Hardening

- Run containers as non-root (already configured)
- Use read-only filesystem for app container when possible
- Set resource limits in Compose (CPU/memory)
- Rotate API keys and secrets regularly
- Enforce HTTPS with strong ciphers
- Network policies: only expose necessary ports
- Back up volumes regularly (Postgres, Grafana, Prometheus)

Compose resource limits example:

deploy:
  resources:
    limits:
      cpus: "2.0"
      memory: 4G
    reservations:
      cpus: "1.0"
      memory: 2G

## 9. Scaling

- Horizontal scaling with multiple app replicas behind a reverse proxy
- Worker can be scaled independently for strategy execution

Example (with docker-compose --profile prod):

app:
  deploy:
    replicas: 3

worker:
  deploy:
    replicas: 2

Ensure idempotent operations and externalized state (DB/Redis).

## 10. Backups and Disaster Recovery

- Nightly Postgres backups (pg_dump) stored offsite
- Redis snapshotting enabled (AOF)
- Restore drills performed quarterly
- Export Grafana dashboards and Prometheus data as part of backup

Sample Postgres backup command:

PGPASSWORD=$POSTGRES_PASSWORD pg_dump -h localhost -U $POSTGRES_USER -d $POSTGRES_DB \
  -Fc -f /backups/axiom_$(date +%F).dump

## 11. Zero-Downtime Deployments

- Use rolling updates with a load balancer
- Health checks must pass before routing traffic
- Blue/Green or Canary strategies for major changes

## 12. Compliance and Monitoring

- Access logs retained per policy
- Alerting on error rates, latency, CPU/memory, queue depth
- Audit trail for deployments and configuration changes

## 13. Troubleshooting

- App not starting: check DB connectivity and migrations
- High latency: profile endpoints, check DB indices, increase replicas
- Data gaps: verify data provider API quotas and retry policies
- Worker stalls: ensure idempotent tasks and visibility timeouts

## 14. Kubernetes (future option)

For large-scale operation, migrate to Kubernetes:

- Use StatefulSets for Postgres (or managed DB)
- Deployments for app and worker
- Ingress with TLS termination
- Prometheus Operator and Grafana
- External Secrets for secret management

## 15. Change Management

- Versioned releases with semantic versioning
- Changelogs and release notes
- Staged rollouts from staging to production

## 16. Post-Deployment Validation

- Run smoke tests against key endpoints
- Validate metrics ingestion and dashboards
- Verify trade simulation in paper mode
- Confirm alerting channels receive test alerts

## 17. Decommissioning

- Drain traffic and stop new jobs
- Archive data per retention policy
- Remove secrets and DNS records
- Destroy infrastructure after validation

Appendix: Ports

- App: 8000
- Postgres: 5432
- Redis: 6379
- Prometheus: 9090
- Grafana: 3000
