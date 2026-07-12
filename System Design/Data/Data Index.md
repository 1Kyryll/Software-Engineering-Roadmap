---
tags:
  - index
  - system-design
  - "#database"
---

# Data Index

Database correctness and modeling: the largest cluster, anchored by the E-Commerce ADRs.

Up: [[System Design MOC]]

## Correctness under concurrency
- [[Database Concurrency and Atomicity]] — atomic conditional decrement + reservations; the flagship pattern
- [[Locking]] — the ladder: atomic write → FOR UPDATE / SKIP LOCKED → optimistic → advisory → distributed
- [[Transaction Isolation Levels]] — anomalies, Postgres specifics, why my checkout runs at Read Committed
- [[Idempotency]] — unique keys, derived keys, ON CONFLICT, idempotent workers
- [[Transactional Outbox]] — atomic "write + publish" via an outbox table (bridge to [[Messaging Index]])

## Modeling decisions (one-way doors)
- [[Primary Keys - UUIDv7]] — vs bigserial, UUIDv4, ULID, Snowflake
- [[Storing Money in Databases]] — NUMERIC + currency, never float
- [[Denormalization and Invariants]] — materialized values + reconciliation as a metric
- [[Pagination - Cursor vs Offset]] — keyset pagination with composite cursors

## Operating the database
- [[Postgres in Production]] — pooling budgets, migrations, sqlc
- [[SQL vs NoSQL]] — Postgres ×4 vs MongoDB ×1, defended

## Connects to
- [[Caching Index]] — caching is denormalization with a TTL; invalidation is the same consistency problem
- [[Messaging Index]] — the outbox feeds Kafka; consumers write back idempotently
- [[Operations Index]] — invariant checks surface as Prometheus gauges
