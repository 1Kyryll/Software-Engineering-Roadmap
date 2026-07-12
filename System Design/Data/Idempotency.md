---
tags: [system-design, reliability, database]
---

# Idempotency

An operation is idempotent when doing it N times has the same effect as doing it once. In distributed systems **retries are unavoidable** (network blips, timeouts, back-button, at-least-once queues), so any operation with side effects must be made safe to repeat. I've implemented this at three different layers.

## Layer 1 — HTTP: Idempotency-Key on checkout (E-Commerce)

[ADR 003](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/003-idempotency-on-order-creation.md): every checkout carries a client-generated `Idempotency-Key` (UUID). It's enforced as a **unique constraint** on the reservations/orders tables — the database, not application logic, is the arbiter. A retried request returns the original order instead of creating a second one.

Implementation in [place_order.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/place_order.go):

```go
// replay? return the existing order
if existing, err := s.repo.GetOrderByIdempotencyKey(ctx, idempotencyKey); err == nil {
    recordOutcome(ctx, "replay")
    return existing, nil
}
```

Subtle detail worth quoting in interviews — **derived keys for sub-operations**. A multi-item checkout creates one reservation per product, each needing its own idempotency key that must be *stable across retries*. Solution: derive them deterministically (UUIDv5) from the parent key:

```go
// deriveReservationKey computes a deterministic UUID v5 per (parent, product),
// so retries with the same parent key produce the same reservation rows.
func deriveReservationKey(parent, productID uuid.UUID) uuid.UUID {
    return uuid.NewSHA1(uuid.NameSpaceURL, []byte(parent.String()+":"+productID.String()))
}
```

Rejected alternatives (ADR 003): trusting the client not to retry (fails on every network blip), deduplicating by user+product+time-window (silently merges two intentional purchases).

## Layer 2 — Queue consumers: idempotent writes (Restaurant, Kafka)

Kafka delivers **at-least-once** — consumers *will* see duplicates (rebalances, redelivery after crash-before-commit). The kitchen service makes ticket creation a no-op on replay with a single SQL clause ([kitchen.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/kitchen/services/kitchen.go)):

```go
err := s.queries.CreateTicketsIdempotent(ctx, ...)  // INSERT ... ON CONFLICT DO NOTHING
```

`ON CONFLICT DO NOTHING` against a unique key on `order_id` = idempotent consumer, no read-before-write race.

## Layer 3 — Workers: idempotent jobs (OneTube)

The transcode worker's design rule: "**The worker is idempotent — reprocessing a job must be safe.**" Concretely: it *overwrites* the same S3 keys (`hls/{video_id}/...`) and *upserts* `video_assets` rows. A job that crashes halfway and gets redelivered by RabbitMQ just redoes the work and converges to the same final state. Overwrite/upsert semantics instead of append semantics is the whole trick. ([transcoder.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/transcode/services/transcoder.go))

The outbox publisher relies on the same property downstream: it guarantees **at-least-once** delivery, so consumers must be idempotent "which is normal anyway" ([ADR 005](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/005-async-fanout-transactional-outbox.md)).

## The general recipe
1. Identify the natural **idempotency key** (client UUID, order_id, S3 key).
2. Enforce it with a **unique constraint / upsert / overwrite** — atomic at the storage layer, never check-then-act in app code.
3. On conflict, **return the original result** (HTTP) or **ack and move on** (queues).

## Related
- [[Database Concurrency and Atomicity]]
- [[Message Queues and Background Workers]] — why at-least-once forces this
- [[Transactional Outbox]]
- [[Data Index]]
