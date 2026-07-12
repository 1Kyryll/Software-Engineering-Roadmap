---
tags: [system-design, database, messaging, reliability]
---

# Transactional Outbox

**Problem (dual-write):** after committing an order to the database you also need to notify downstream systems (email, analytics, kitchen...). Writing to the DB *and* publishing to a broker are two systems — if the second write fails after the first commits, the order exists but nobody hears about it. There is no transaction spanning Postgres and Kafka.

**Solution:** write the event into an `outbox` **table in the same DB transaction** as the business row. A separate relay process polls unpublished rows, publishes them to the broker, and marks them published. The DB transaction is the single atomic commit point.

I've implemented this **twice**, in two different stacks.

## E-Commerce-System (design + polling publisher)

[ADR 005](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/005-async-fanout-transactional-outbox.md) — order finalization inserts `orders`, `order_items`, marks the reservation consumed, **and** inserts the `OrderPlaced` outbox event, all in one transaction ([place_order.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/place_order.go) → `FinalizeOrder`).

Table ([migration 0006](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/migrations/0006_create_outbox.up.sql)): `id, aggregate_type, aggregate_id, event_type, payload jsonb, created_at, published_at`.

Relay queries ([queries/order/outbox.sql](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/queries/order/outbox.sql)):

```sql
-- name: ListUnpublishedOutboxEvents :many
SELECT ... FROM outbox WHERE published_at IS NULL ORDER BY created_at ASC LIMIT $1;

-- name: MarkOutboxEventPublished :exec
UPDATE outbox SET published_at = now() WHERE id = $1;
```

## Restaurant System (full relay → Kafka, running in prod-shape)

[outbox.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/outbox/outbox.go) — a goroutine ticks every second, drains up to 100 unpublished events, publishes each to Kafka topic `orders.events`, then marks it published:

```go
func (r *OutboxRelay) publishPending(ctx context.Context) {
    events, _ := r.queries.GetUnpublishedOutboxEvents(ctx, 100)
    for _, event := range events {
        msg := kafka.Message{
            Key:   []byte(strconv.Itoa(int(event.AggregateID))), // partition by order_id
            Value: event.Payload,
        }
        if err := r.writer.WriteMessages(ctx, msg); err != nil { return } // retry next tick
        r.queries.MarkOutboxEventPublished(ctx, event.ID)
    }
}
```

Details that matter:
- **Keyed by aggregate ID** (`order_id`) → all events for one order land on the same Kafka partition → per-order ordering preserved.
- **Publish, then mark.** If the process crashes between the two, the event is re-published next tick → **at-least-once** delivery. That's the deliberate tradeoff: consumers must be idempotent (see [[Idempotency]] — the kitchen's `ON CONFLICT DO NOTHING`). Mark-then-publish would risk *losing* events, which is worse.
- Ordered draining (`ORDER BY created_at ASC`) and bounded batches.

## Rejected alternatives (from ADR 005)
- **Publish to the broker after commit, from the request handler** — the classic dual-write: broker down = event silently lost.
- **Call every downstream service synchronously inside the order transaction** — checkout latency becomes the max of all consumers; one slow service stalls purchases.

## What you get
- Exactly-once **commit**, at-least-once **delivery**, replayability (events are rows — new consumers can re-read history), and a free audit trail.
- Cost: one more table, a relay process, and ~1s publish latency (the poll interval).
- Fancier variant to name-drop: **CDC/log tailing** (Debezium reads the WAL instead of polling) — same guarantee, lower latency, more moving parts.

## Related
- [[Kafka and Event Streaming]]
- [[Idempotency]]
- [[Database Concurrency and Atomicity]]
- [[Data Index]]
