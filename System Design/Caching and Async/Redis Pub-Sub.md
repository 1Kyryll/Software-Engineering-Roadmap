---
tags: [system-design, redis, messaging, realtime]
---

# Redis Pub/Sub

Fire-and-forget message broadcast: publishers send to a **channel**, every currently-subscribed client receives a copy. No persistence, no acknowledgment, no replay — if nobody is listening (or a subscriber is down), the message is gone.

## Where I built it — Lychee-Chat message fan-out

[packages/redis/pubsub.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/pubsub.ts):

```ts
export async function subscribe(channel: string, callback: (payload: any) => void) {
    const subscriber = redisPubSubClient.duplicate();  // dedicated connection!
    await subscriber.subscribe(channel);
    subscriber.on("message", (chan, payload) => {
        if (chan === channel) callback(JSON.parse(payload));
    });
}
```

Flow: `message:send` handler saves the message to MongoDB, publishes to the `chat:message` channel; the subscriber in the WS server broadcasts to the Socket.IO room ([events/message.send.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/events/message.send.ts)).

**Why it exists**: with multiple WebSocket server replicas, users in one chat can be connected to different instances. Redis pub/sub is the shared backbone that lets *any* instance's write reach *every* instance's sockets. Full context in [[Real-Time Communication]].

**Implementation detail that interviews love**: a Redis connection in subscriber mode can *only* issue subscribe-family commands — hence `duplicate()` to get a dedicated connection, separate from the one doing GET/SET cache work ([redisClient.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/redisClient.ts) keeps them apart). BullMQ needs its own duplicated connection for the same reason (blocking commands).

## Pub/Sub vs a real queue vs a log — know the ladder

| | Redis Pub/Sub | Queue (RabbitMQ/BullMQ) | Log (Kafka) |
|---|---|---|---|
| Delivery | at-most-once, only to *current* subscribers | at-least-once, persisted until acked | at-least-once, persisted for retention window |
| Consumer down? | message lost | message waits | message waits, offset resumes |
| Replay | no | no | yes (re-read from offset) |
| Fan-out | yes, free | via exchanges/bindings | consumer groups |
| Right for | ephemeral broadcast (live chat, presence) | work distribution (jobs) | event streams, audit, multiple independent consumers |

Losing a chat broadcast is fine — the message is already durably in MongoDB, and a reconnecting client refetches history. That's *why* pub/sub is acceptable here: **durability lives in the DB, pub/sub only handles liveness**. If delivery mattered, I'd reach for [[Message Queues and Background Workers]] or [[Kafka and Event Streaming]].

(Redis Streams — persistent, consumer groups, replay — is Redis's own answer when you want queue-like semantics without a second system. Good name-drop.)

## Related
- [[Real-Time Communication]]
- [[Caching Strategies]]
- [[Kafka and Event Streaming]]
- [[Messaging Index]]
