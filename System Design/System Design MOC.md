---
tags: [moc, system-design]
---

# System Design — Map of Content

Every topic here is grounded in my own projects — each note explains the concept, then points to the **actual implementation in my GitHub repos** with file-level links. This is the "I didn't just read about it, I built it" layer for interviews.

**Structure**: this MOC links to cluster **indexes**; each index links to its notes and names its connections to other clusters. Follow `Up:` links to climb back.

## Clusters

- [[Architecture Index]] — monolith vs microservices, gateways, reverse proxy, rate limiting
- [[Communication Index]] — REST, gRPC, GraphQL, real-time channels
- [[Data Index]] — concurrency, idempotency, outbox, modeling decisions, Postgres ops *(largest cluster)*
- [[Caching Index]] — cache-aside, invalidation, eviction, stampede
- [[Messaging Index]] — queues vs logs, delivery semantics, ordering, poison messages
- [[Operations Index]] — observability, load testing, orchestration & CI

## Standalone notes
- [[Object Storage and Presigned URLs]] — fundamentals + direct-to-S3 + private-bucket HLS
- [[Authentication and Authorization]] — JWT, cookies, RBAC, password hashing

## My projects (the evidence base)

| Repo | What it demonstrates |
|---|---|
| [E-Commerce-System](https://github.com/1Kyryll/E-Commerce-System) | Go microservices, gRPC + REST gateway, atomic inventory, reservations, outbox, ADRs, OTel/LGTM observability, k6 load testing |
| [Restaurant-System-Microservices](https://github.com/1Kyryll/Restaurant-System-Microservices) | GraphQL gateway, gRPC, Kafka (KRaft) + outbox relay, SSE, WebSocket subscriptions, RBAC |
| [OneTube](https://github.com/1Kyryll/OneTube) | Object storage (MinIO S3), presigned URLs, RabbitMQ, idempotent ffmpeg worker, HLS streaming |
| [Lychee-Chat](https://github.com/1Kyryll/Lychee-Chat) | Socket.IO real-time chat, Redis caching + pub/sub, BullMQ background jobs, MongoDB, Nginx |
| [Personal-Finance-Tracker](https://github.com/1Kyryll/Personal-Finance-Tracker) | Classic monolith: React SPA + Flask REST API + Postgres behind Nginx |

## Related
- [[Home]]
- [[DSA MOC]]
