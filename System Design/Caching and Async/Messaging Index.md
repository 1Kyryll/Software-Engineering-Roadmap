---
tags: [index, system-design]
---

# Messaging Index

Async communication: queues, logs, broadcast — and the reliability problems they all share.

Up: [[System Design MOC]]

## The systems
- [[Message Queues and Background Workers]] — RabbitMQ (OneTube transcode), BullMQ (Lychee media); ack/nack, QoS, backoff (start here)
- [[Kafka and Event Streaming]] — partitioned log, consumer groups, KRaft (Restaurant)
- [[RabbitMQ vs Kafka]] — the head-to-head: "a task for someone" vs "a fact for anyone"
- [[Redis Pub-Sub]] — ephemeral broadcast for WebSocket fan-out (Lychee)

## The reliability problems (transport-agnostic)
- [[Delivery Semantics and Offset Commits]] — at-most/at-least/exactly-once; the read-but-crash-before-commit scenario
- [[Message Ordering]] — partition keys, retries that reorder, sequence numbers
- [[Poison Messages and Dead Letter Queues]] — crash loops, bounded retries, DLQ + replay

## Connects to
- [[Data Index]] — [[Transactional Outbox]] gets events *into* the broker; [[Idempotency]] absorbs redelivery on the way out
- [[Communication Index]] — [[Real-Time Communication]] is the sync/push counterpart (SSE consumes Kafka in Restaurant)
- [[Caching Index]] — BullMQ and cache keys share a Redis; eviction policy must protect the queue
