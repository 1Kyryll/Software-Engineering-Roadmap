---
tags: [system-design, messaging, kafka, rabbitmq]
---

# RabbitMQ vs Kafka

I run both in production-shaped projects — RabbitMQ in [OneTube](https://github.com/1Kyryll/OneTube) (transcode jobs), Kafka in [Restaurant-System-Microservices](https://github.com/1Kyryll/Restaurant-System-Microservices) (order events) — and the two choices were made for opposite reasons. That contrast *is* the answer.

## The mental models

**RabbitMQ = smart broker, dumb consumer.** A message is a **task**: the broker routes it (exchanges/bindings), tracks per-message delivery state, redelivers unacked messages, and deletes it once acked. Consumed = gone.

**Kafka = dumb broker, smart consumer.** A message is a **fact in a log**: appended to a partition, retained for a time window regardless of consumption. Consumers just track their own **offset**. Reading doesn't delete; anyone can re-read.

## Side by side

| | RabbitMQ | Kafka |
|---|---|---|
| Unit of work | task to be done once | event that happened |
| After consumption | message deleted | message retained (retention window) |
| Multiple independent readers | needs a queue per consumer (fanout exchange) | free — each consumer group has its own offsets |
| Replay / reprocess history | no | yes — rewind the offset |
| Ack granularity | per message (ack/nack/requeue) | per offset (position in partition) |
| Ordering | per queue (single consumer) | per partition, by key ([[Message Ordering]]) |
| Routing | rich: direct/topic/headers/fanout exchanges | none — producers pick topic+key, that's it |
| Backpressure to long jobs | `Qos(prefetch=1)` fair dispatch | partition assignment (max parallelism = partition count) |
| Poison message handling | built-in DLX (dead-letter exchange) | manual: DLQ topics + commit-and-skip ([[Poison Messages and Dead Letter Queues]]) |
| Throughput ceiling | high | very high (sequential log I/O, zero-copy) — the "firehose" choice |
| Priority queues / per-message TTL | yes | no |

## Why each of my systems chose what it chose

**OneTube → RabbitMQ.** Transcode jobs are classic work distribution: each job should be done ~once by *some* worker, jobs are independent (no ordering — see [[Message Ordering]]), jobs are heavy and long → `Qos(1)` fair dispatch matters ([rabbit.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/common/queue/rabbit.go)). Nobody will ever want to "replay all transcodes from Tuesday." A log's retention/replay machinery buys nothing here; per-message ack and prefetch control buy a lot.

**Restaurant → Kafka.** An order event is a fact that *multiple* services care about — kitchen today; analytics, notifications, audit tomorrow — each as an independent consumer group reading the same topic with **zero producer changes**. Per-order ordering falls out of partition keying. Replay means a new consumer can bootstrap from history. With RabbitMQ this fan-out would need explicit exchange/queue topology per new consumer, and history would be gone.

## The heuristic

> **"A task for someone" → queue (RabbitMQ). "A fact for anyone" → log (Kafka).**

Follow-ups to be ready for:
- **Can Kafka do job queues?** Awkwardly — no per-message ack means one slow job blocks its partition; no priority; parallelism capped by partitions. Possible, not natural.
- **Can RabbitMQ do event streams?** Somewhat (fanout exchanges, and RabbitMQ Streams exists now), but replay/multi-reader is bolted on rather than the core model.
- **Where does Redis pub/sub fit?** Below both: no persistence at all — ephemeral broadcast only (see the ladder in [[Redis Pub-Sub]]).
- **Both need idempotent consumers** — at-least-once is the practical reality in either ([[Delivery Semantics and Offset Commits]]).

## Related
- [[Message Queues and Background Workers]]
- [[Kafka and Event Streaming]]
- [[Message Ordering]]
- [[Poison Messages and Dead Letter Queues]]
- [[Messaging Index]]
