---
tags: [system-design, messaging, kafka]
---

# Kafka and Event Streaming

Kafka is a **distributed, partitioned, replicated commit log**, not a queue: events are appended to topic partitions and retained for a window; consumers track their own **offsets** and can replay. Multiple consumer groups read the same events independently — that's the property queues don't give you.

## Where I built it — Restaurant System

Order events flow `orders service → outbox → Kafka → kitchen service` ([architecture](https://github.com/1Kyryll/Restaurant-System-Microservices#architecture)):

- **Broker**: Kafka in **KRaft mode** (no ZooKeeper — Kafka's own Raft-based metadata quorum; the modern deployment default) via [docker-compose.yaml](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/docker-compose.yaml), plus Kafka UI on `:8090` for inspecting topics/consumer groups.
- **Producer**: the outbox relay ([outbox.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/outbox/outbox.go)) publishes to topic `orders.events` with `segmentio/kafka-go`:

```go
writer := &kafka.Writer{
    Addr:     kafka.TCP(brokers...),
    Topic:    "orders.events",
    Balancer: &kafka.Hash{},   // hash by key → same key, same partition
}
msg := kafka.Message{
    Key:   []byte(strconv.Itoa(int(event.AggregateID))),  // order_id
    Value: event.Payload,                                  // Protobuf
}
```

- **Consumer**: the kitchen service reads with consumer group `kitchen-service` and creates tickets idempotently (`ON CONFLICT DO NOTHING`), then pushes to the dashboard via SSE.

## The concepts, mapped to my code

**Partitioning & ordering.** Kafka guarantees order *within a partition only*. Keying messages by `order_id` (the `Hash` balancer) puts all of one order's events on one partition → per-order ordering, while different orders parallelize across partitions. Choosing the partition key = choosing your ordering + parallelism unit. This is *the* Kafka interview question.

**Consumer groups.** Partitions are divided among a group's members: scale consumers up to (at most) the partition count. A second group (say, `analytics-service`) would receive every event again, independently — this is why Kafka suits event fan-out where a queue would need one queue per consumer.

**Delivery semantics.** My pipeline is **at-least-once** twice over: the outbox relay may re-publish (publish-then-mark), and Kafka redelivers uncommitted offsets after a consumer crash. Idempotent consumption absorbs both — see [[Idempotency]] and [[Transactional Outbox]].

**Schema on the wire.** Events carry **Protobuf payloads**, not ad-hoc JSON — producers and consumers share the contract from [[gRPC and Protocol Buffers]]. (In bigger setups: schema registry.)

## Kafka vs RabbitMQ (both are in my projects — the comparison is mine to make)
- **RabbitMQ (OneTube)**: smart broker, dumb consumer — broker tracks delivery, routes, retries. Message is *gone* once acked. Ideal for job/work distribution.
- **Kafka (Restaurant)**: dumb broker, smart consumer — consumers track offsets, events are retained and replayable, multiple groups read independently. Ideal for event streams that several services care about, audit, and reprocessing.
- Heuristic: "a task to be done exactly-ish once by someone" → queue; "a fact that happened, of interest to many" → log.

## Related
- [[Transactional Outbox]] — how events get into Kafka without dual-writes
- [[Message Queues and Background Workers]]
- [[Real-Time Communication]] — SSE on the consuming end
- [[Messaging Index]]
