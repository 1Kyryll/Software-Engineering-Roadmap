---
tags: [system-design, messaging, reliability]
---

# Poison Messages and Dead Letter Queues

A **poison message** is one the consumer can never process successfully — malformed payload, references a deleted entity, triggers a bug. Combined with at-least-once redelivery it produces the nastiest queue failure mode: **the crash loop**. Worker reads message → crashes → broker redelivers → crashes again → the whole consumer is wedged behind one bad message, and every healthy message behind it starves.

## The failure amplifier
Naive "retry on failure" turns one bad message into an outage:
- RabbitMQ `Nack(requeue=true)` puts it right back at the head → hot loop, 100% CPU, zero throughput.
- Kafka: crash-before-commit means the consumer restarts *at the same offset* → the partition is blocked entirely (Kafka can't skip within a partition without committing past the message).

## The standard solution ladder

1. **Bounded retries with backoff** — retry N times, spaced out (transient failures resolve; real poison doesn't).
2. **Dead Letter Queue (DLQ)** — after N failures, move the message to a side queue instead of retrying forever. The main queue keeps flowing; humans/tools inspect the DLQ, fix the cause, and **replay** from it.
3. **Alert on DLQ depth** — a growing DLQ is a symptom feed, not a garbage can.
4. **Validate at the boundary** — reject unparseable messages *immediately* to the DLQ (no retries; parse failures are never transient).

## How my projects handle it (including honest gaps)

**Lychee-Chat (BullMQ) — the complete pattern, declaratively** ([media.queue.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/queues/media.queue.ts)):

```ts
{ attempts: 3, backoff: { type: "exponential", delay: 5000 } }
```

Three tries at 5s/10s/20s, then BullMQ parks the job in its built-in **failed set** (a DLQ in all but name) — inspectable and replayable. `removeOnFail: true` in my config trades that inspectability away to keep Redis small; the right production call is keeping failed jobs with a retention cap.

**OneTube (RabbitMQ) — deliberate drop, stated gap** ([rabbit.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/common/queue/rabbit.go)):

```go
if err := h(ctx, job); err != nil {
    d.Nack(false, false)   // requeue=false: drop, don't crash-loop
    continue
}
```

`requeue=false` correctly avoids the hot loop, but with **no DLQ configured the job is simply lost** — a failed transcode leaves the video stuck in `processing` forever. The RabbitMQ fix is small and worth knowing cold: declare the queue with `x-dead-letter-exchange`, and nacked messages route there automatically. (A reconciliation sweep for stuck-in-`processing` videos would be the belt-and-suspenders.)

**Kafka (Restaurant) — skip-and-log or retry topics.** Since a partition can't be selectively skipped, the pattern is: catch the processing error, publish the event to a `orders.events.dlq` topic, **commit the offset anyway**, continue. Larger setups use tiered retry topics (`retry.5s`, `retry.1m`, `retry.10m`) before the DLQ — retries without blocking the main partition.

## Design rules
- **Distinguish transient from permanent** errors in the handler: DB timeout → retry; unmarshal failure → straight to DLQ (my OneTube consumer nacks unparseable bodies immediately — right instinct, wrong destination).
- Retries must be **bounded and backed off**; unbounded retry is self-DoS.
- A DLQ without monitoring and a replay path is just a slower way to lose messages.
- Idempotency ([[Idempotency]]) makes replay-from-DLQ safe — the two patterns are a set.

## Related
- [[Delivery Semantics and Offset Commits]]
- [[Message Queues and Background Workers]]
- [[RabbitMQ vs Kafka]]
- [[Messaging Index]]
