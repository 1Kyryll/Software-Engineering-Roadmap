---
tags: [system-design, operations, docker, ci]
---

# Docker Compose Orchestration (+ CI/CD)

Every one of my projects ships as a compose file; the E-Commerce one is the most production-shaped and worth studying line by line: [docker-compose.yml](https://github.com/1Kyryll/E-Commerce-System/blob/main/docker-compose.yml).

## Patterns in that file

**Startup ordering via healthchecks, not `sleep`.** `depends_on` alone only orders *container start*, not *readiness*. Conditions fix that:

```yaml
depends_on:
  postgres:
    condition: service_healthy              # pg_isready healthcheck passes
  migrate:
    condition: service_completed_successfully  # one-shot ran to completion
```

**One-shot migration job.** A `migrate` service applies schema migrations and exits; every app service waits for it. No service ever sees a half-migrated schema (details in [[Postgres in Production]]).

**Shared env via YAML anchors** — config defined once, DRY across services:

```yaml
x-backend-env: &backend-env
  DATABASE_URL: postgres://...
  REDIS_URL: redis://redis:6379
  JWT_SECRET: ${JWT_SECRET:-dev-only-...}   # env override with a dev default
services:
  order:
    environment:
      <<: *backend-env
      ORDER_GRPC_PORT: "9002"
```

**Service discovery = network DNS.** `CATALOG_ADDR: catalog:9000` — compose network DNS resolves service names, so the gateway config is identical in any environment where those names resolve. Same trick lets Nginx `proxy_pass http://ws:4000` in Lychee-Chat ([[Reverse Proxy with Nginx]]).

**Infra tuned in the compose file**: `postgres -c max_connections=200` sized against the services' pool budget (the cross-reference is written as a comment *in the compose file* — infra config and app config that must agree, documented at the point of agreement).

**Separate compose stack for observability** ([observability/docker-compose.yml](https://github.com/1Kyryll/E-Commerce-System/blob/main/observability/docker-compose.yml)) — LGTM runs independently so the app stack doesn't drag Grafana along on every restart.

Other repos add: multi-stage Dockerfiles (Go build stage → scratch/alpine runtime; Restaurant, OneTube), Kafka in KRaft mode + Kafka UI (Restaurant), MinIO + RabbitMQ with management UI (OneTube), Mongo + Redis with auth (Lychee).

## CI/CD ([.github/workflows](https://github.com/1Kyryll/E-Commerce-System/tree/main/.github/workflows), [pipeline.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/pipeline.md))

On every push, two jobs:
- **Backend**: regenerate proto + sqlc code (generated code is never committed — CI proves the sources still generate), unit tests, then **integration tests against a real Postgres** service container (`go test -tags=integration`)
- **Frontend**: generated-types check (OpenAPI → TS types must be in sync), typecheck, lint, build

The build tag split (`go test ./...` vs `-tags=integration`) keeps the fast unit loop local while CI pays for the real-database run.

## Interview positioning
- Compose = single-host orchestration: networks, healthchecks, restart policies, volumes. My deploy target is Compose-on-Hetzner — honest about scale.
- Know the upgrade path and what it buys: Kubernetes adds multi-node scheduling, self-healing, rolling deploys, HPA autoscaling, Services/Ingress — and an operational tax that a single-host system doesn't justify.
- The healthcheck/one-shot-migration/env-anchor patterns transfer directly to k8s (readiness probes, init containers/Jobs, ConfigMaps).

## Related
- [[Postgres in Production]]
- [[Observability]]
- [[Monolith vs Microservices]]
- [[Operations Index]]
