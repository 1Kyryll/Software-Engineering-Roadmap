---
tags: [system-design, database, postgres]
---

# Postgres in Production — Pooling, Migrations, Type-Safe Queries

The operational plumbing around Postgres that shows up in every one of my Go projects.

## Connection pooling

Opening a Postgres connection is expensive (TCP + TLS + auth + backend process fork), and Postgres has a hard `max_connections` ceiling. Every service therefore holds a **pool** of reusable connections — `pgxpool` in all my Go services.

The part most people miss is **capacity math across services**. From [E-Commerce pool.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/internal/database/pool.go):

```go
// pgx's own default is 4, which is too small for any realistic load;
// 25 gives 4 services × 25 = 100 simultaneous connections in steady state
// (Postgres's default max_connections is 100; docker-compose raises it
// to 200 to leave headroom).
const DefaultMaxConns int32 = 25
```

And the matching side in [docker-compose.yml](https://github.com/1Kyryll/E-Commerce-System/blob/main/docker-compose.yml):

```yaml
command: ["postgres", "-c", "max_connections=200"]
```

Interview point: pool sizes are a **fleet-wide budget**, not a per-service knob. N services × pool size + monitoring/psql sessions must stay under `max_connections`, or services start failing to connect under load — exactly when you can least afford it. (At bigger scale: PgBouncer in transaction mode multiplexes thousands of client connections onto tens of server ones.)

The pool is also **fail-fast**: `NewPool` pings with a 5s timeout so a service with a bad DB URL dies at startup, not on first request.

## Migrations — versioned, ordered, reversible

Schema changes are numbered SQL files applied in order, with up/down pairs:

- E-Commerce & OneTube: `golang-migrate` — [E-Commerce migrations/](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/migrations) (`0001_create_users.up.sql` ... `0006_create_outbox.up.sql`)
- Restaurant: `goose` — same idea, different tool
- Finance-Tracker: Flask-side migrations

Deployment pattern worth naming: in E-Commerce, migrations run as a **one-shot compose service** that must complete before any app service starts:

```yaml
migrate:
  image: migrate/migrate:4
  command: ["-path=/migrations", "-database=postgres://...", "up"]
depends_on:            # on app services:
  migrate:
    condition: service_completed_successfully
```

No app ever sees a half-migrated schema, and migration failures block the deploy loudly.

## Type-safe SQL with sqlc

I write plain SQL; [sqlc](https://sqlc.dev) generates typed Go functions from it at build time. Used in E-Commerce, OneTube, and Restaurant ([sqlc.yaml](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/sqlc.yaml), queries in [backend/queries/](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/queries)):

```sql
-- name: ListExpiredActiveReservations :many
SELECT id, product_id, user_id, quantity, expires_at
  FROM reservations
 WHERE status = 'active' AND expires_at < now()
 ORDER BY expires_at ASC
 LIMIT $1;
```

→ generates `func (q *Queries) ListExpiredActiveReservations(ctx, limit) ([]Row, error)`.

Why this over an ORM: the SQL is exactly what runs (no N+1 surprises, no query-builder ceiling for CTEs like the atomic decrement), yet parameters and result columns are compile-time checked against the actual schema. Generated code is gitignored and rebuilt in CI — SQL files are the source of truth. A thin hand-written `repo/` layer wraps the generated queries and maps rows to domain types ([catalog/repo/repo.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/catalog/repo/repo.go)), keeping sqlc types out of business logic.

## Related
- [[Database Concurrency and Atomicity]]
- [[Primary Keys - UUIDv7]]
- [[Docker Compose Orchestration]]
- [[Data Index]]
