---
tags: [system-design, messaging, kafka]
---

# Message Ordering

"How do you ensure messages are consumed in the exact order they were sent?" The honest first answer: **you scope the guarantee**. Global total ordering and parallelism are fundamentally at odds — a system that processes everything in one strict sequence has exactly one lane. The design skill is finding the *smallest unit that actually needs ordering* and guaranteeing it there only.

## The ordering toolbox

**1. One partition/queue + one consumer = trivially ordered, trivially unscalable.** The baseline. Sometimes right (low volume, strict global sequence like a ledger).

**2. Partition by ordering key — the standard answer.** Order matters *per entity*, not globally: order #17's events must be sequenced relative to each other, but may interleave freely with order #18's. Kafka gives ordering **within a partition**, so key messages by the entity ID:

```go
// Restaurant System outbox relay (outbox.go)
writer := &kafka.Writer{ Balancer: &kafka.Hash{} }   // same key → same partition
msg := kafka.Message{
    Key:   []byte(strconv.Itoa(int(event.AggregateID))),  // order_id
    Value: event.Payload,
}
```

All events for one order → one partition → consumed in order; different orders parallelize across partitions. This is my [Restaurant System](https://github.com/1Kyryll/Restaurant-System-Microservices)'s actual design ([outbox.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/outbox/outbox.go)) — `created → preparing → ready` can never arrive scrambled *for the same order*.

**3. Ordered at the source.** Partitioning only preserves the order messages are *published* in — so the publisher must emit them in order. My outbox relay drains `ORDER BY created_at ASC` in a single goroutine ([[Transactional Outbox]]): DB commit order ≈ publish order. A pool of concurrent relay workers would silently destroy the guarantee the partition key is supposed to provide.

## The subtle ways ordering breaks (interview differentiators)

- **Retries reorder.** Producer sends msg1, it times out (but actually succeeded), producer sends msg2, then retries msg1 → broker has 1,2,1 or 2,1. Kafka's fix: **idempotent producer** (`enable.idempotence=true` — sequence numbers per partition let the broker dedupe/reject out-of-order retries) — without it, `max.in.flight > 1` + retries = reordering risk.
- **Consumer concurrency reorders.** Reading in order then dispatching to a worker pool loses the order at the pool. If you parallelize, shard the pool by the same key (per-key serial execution).
- **Rebalances + reprocessing**: after a crash you re-see messages ([[Delivery Semantics and Offset Commits]]) — consumers must handle "old" events (idempotency, or version checks: ignore status regressions).
- **Repartitioning changes key→partition mapping** — in-flight history for a key straddles two partitions. Plan partition counts up front.

**4. Sequence numbers when transport can't be trusted.** Stamp each message (`seq: 1, 2, 3...` per entity); the consumer buffers/reorders or rejects gaps. This is TCP's trick, and the fallback over inherently unordered transports (SQS standard, multi-queue fan-in). Cheaper cousin: **version on the entity** — consumer ignores events whose version ≤ the last applied (turns ordering into idempotency).

## Where ordering *doesn't* matter (say this too)
- OneTube transcode jobs: each video is independent; renditions could even run in parallel per video — no ordering requirement at all, which is why a plain RabbitMQ work queue fits.
- Lychee chat: messages are timestamped and persisted in MongoDB; clients render by timestamp, so momentary broadcast reordering self-heals on the next history fetch.

Declaring "this flow needs no ordering" is as much a design decision as guaranteeing it.

## Related
- [[Kafka and Event Streaming]]
- [[Delivery Semantics and Offset Commits]]
- [[Transactional Outbox]]
- [[RabbitMQ vs Kafka]]
- [[Messaging Index]]
