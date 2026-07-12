---
tags: [system-design, messaging, kafka, reliability]
---

# Delivery Semantics and Offset Commits

The core question of any messaging system: **when the consumer crashes at the worst possible moment, is the message processed zero, one, or two times?** The answer is decided by *when you acknowledge* relative to *when you do the work*.

## The three semantics

| Semantics | How you get it | Failure mode |
|---|---|---|
| **At-most-once** | commit/ack *before* processing | crash after commit, before processing → **message lost** |
| **At-least-once** | commit/ack *after* processing | crash after processing, before commit → **message reprocessed** |
| **Exactly-once** | at-least-once + idempotent/transactional processing | engineering cost |

There is no free exactly-once across independent systems — you *construct* it from at-least-once delivery plus deduplication on the consumer side.

## The Kafka offset-commit pitfall (the interview scenario)

Kafka consumers don't ack individual messages; they **commit offsets** — "I have handled everything up to position N in this partition." The failure windows:

**Process-then-commit (at-least-once — the right default):**
1. Consumer reads message at offset 42
2. Writes a kitchen ticket to Postgres ✓
3. **Crashes before committing offset 42**
4. Rebalance → another consumer starts from the last committed offset (41) → **re-reads 42** → would create a duplicate ticket

This is exactly the case my Restaurant System handles: ticket creation is `INSERT ... ON CONFLICT DO NOTHING` ([kitchen.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/kitchen/services/kitchen.go)), so the redelivered event is a no-op. **Redelivery is not an edge case — rebalances alone guarantee you'll see duplicates in normal operation.**

**Commit-then-process (at-most-once — usually wrong):**
Crash between commit and processing silently loses the order event. A kitchen that occasionally doesn't hear about an order is worse than one that occasionally prints two tickets and throws one away.

**Auto-commit is a trap worth naming**: `enable.auto.commit=true` commits on a timer, decoupled from your processing — it can commit offsets for messages you haven't finished (loss) *and* still redeliver ones you have (duplicates). For anything correctness-sensitive, commit manually after the side effect.

## The same problem in my other systems

- **Outbox relay** ([outbox.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/outbox/outbox.go)): *publish to Kafka, then* `MarkOutboxEventPublished`. Crash between the two → event re-published next tick. Deliberately at-least-once; mark-then-publish would be at-most-once and could lose events. Same shape, DB row instead of offset.
- **RabbitMQ** ([OneTube rabbit.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/common/queue/rabbit.go)): per-message `Ack` after the handler succeeds — at-least-once, absorbed by the idempotent worker ([[Idempotency]]).
- **BullMQ** (Lychee): job completion is recorded after the worker returns; retries redo the job — absorbed by the positional `$set` overwrite being idempotent.

## Toward "effectively exactly-once" — the ladder
1. **Idempotent consumer** (my approach): unique key + `ON CONFLICT DO NOTHING` / upsert / overwrite. Works across any transport.
2. **Store offset with the result**: write the processed result *and* the consumed offset in the same DB transaction; on restart, resume from the DB's offset. Turns "two systems" back into one atomic commit — the consumer-side mirror of the [[Transactional Outbox]].
3. **Kafka transactions / idempotent producer** (`read_process_write` within Kafka): exactly-once *within* Kafka pipelines (Streams); doesn't help once you touch an external DB or API.

## Rules of thumb
- Default to **at-least-once + idempotency**. It's the simplest thing that's actually correct.
- Decide *per side effect*: charging a card twice ≠ printing a ticket twice ≠ sending an email twice. My checkout pushes dedup down to the payment provider via the same idempotency key ([[Idempotency]]).
- Never let the ack/commit happen implicitly (auto-commit, auto-ack) on a path where the side effect matters.

## Related
- [[Idempotency]]
- [[Transactional Outbox]]
- [[Kafka and Event Streaming]]
- [[Poison Messages and Dead Letter Queues]] — when the message *can't* be processed at all
- [[Messaging Index]]
