---
tags: [system-design, messaging, async]
---

# Message Queues and Background Workers

Move slow/heavy/retryable work out of the request path: the API enqueues a **job**, returns immediately, and a **worker** processes it asynchronously. The queue is the buffer that absorbs load spikes and survives worker crashes.

## Where I built it

### OneTube — RabbitMQ + ffmpeg transcode worker
The API publishes a transcode job when an upload completes; a separate Go binary consumes it, runs ffmpeg (minutes of CPU), and uploads HLS renditions to S3. ([queue/rabbit.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/common/queue/rabbit.go))

Every production-relevant knob is visible in that one file:

```go
// durable queue: survives broker restart
ch.QueueDeclare(queue, true, false, false, false, nil)

// persistent messages: survive broker restart
ampq.Publishing{ DeliveryMode: ampq.Persistent, Body: body }

// prefetch=1: don't hand a worker a second job until it acks the first —
// fair dispatch for long-running CPU jobs
ch.Qos(1, 0, false)

// consumer loop: explicit ack on success, nack on failure
if err := h(ctx, job); err != nil {
    d.Nack(false, false)   // failed → don't requeue (would need a DLQ story)
    continue
}
d.Ack(false)
```

Why the split exists at all: transcoding is CPU-bound (ffmpeg pegs cores for minutes), the API is I/O-bound. Coupling them means one upload starves every HTTP request. See [[Monolith vs Microservices]]. And because RabbitMQ redelivers unacked jobs after a crash, **the worker must be idempotent** — it overwrites the same S3 keys and upserts asset rows ([[Idempotency]]).

### Lychee-Chat — BullMQ image compression
Images are shown to users **immediately** (raw base64 saved + broadcast), then a BullMQ job compresses them in the background and swaps the stored copy — latency-critical path stays fast, heavy work happens async. ([queues/media.queue.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/queues/media.queue.ts), [workers/media.worker.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/workers/media.worker.ts))

```ts
await mediaQueue.add("processMedia", { messageId, index, buffer, mimeType }, {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },  // 5s, 10s, 20s
    removeOnComplete: true,
    removeOnFail: true,
});
```

- **Retries with exponential backoff** declared *on the job* — the queue framework owns the retry policy, not hand-rolled loops.
- The worker compresses with sharp and updates exactly one array element in MongoDB via positional `$set` on `media[index]` — a surgical, idempotent write.
- BullMQ runs on Redis and needs a dedicated connection with `maxRetriesPerRequest: null` (its blocking commands can't share the cache connection).

### E-Commerce — cleanup worker (scheduled, not queue-driven)
Not every worker needs a queue: the [cleanup-worker](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/cleanup.go) polls Postgres for expired reservations on a schedule and releases them in bounded batches, tolerating per-row failures. When the "queue" is naturally a DB table with a state column (see [[Transactional Outbox]]), polling beats adding a broker.

## The decision points (interview checklist)
- **At-least-once is the default reality** → consumers must be idempotent, always.
- **Ack semantics**: ack after success, nack/requeue on failure — and know what happens on poison messages (dead-letter queues; my OneTube `Nack(requeue=false)` drops them, a stated gap).
- **Prefetch/concurrency**: `Qos(1)` for fair long-job dispatch; higher prefetch for cheap fast jobs.
- **Durability levels**: durable queue + persistent messages + (optionally) publisher confirms.
- **Queue vs pub/sub vs log**: work distribution vs ephemeral broadcast vs replayable stream — see the table in [[Redis Pub-Sub]].

## Related
- [[Idempotency]]
- [[Transactional Outbox]]
- [[Kafka and Event Streaming]]
- [[Object Storage and Presigned URLs]] — where the transcode output goes
- [[Messaging Index]]
