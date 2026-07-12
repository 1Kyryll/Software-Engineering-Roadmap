---
tags: [system-design, communication, realtime, websockets]
---

# Real-Time Communication

Pushing data to clients without them asking. I've shipped **four different mechanisms** across projects — knowing when each fits is the interview differentiator.

## The options

| Mechanism | Direction | Transport | Where I used it |
|---|---|---|---|
| **WebSockets** (Socket.IO) | bidirectional | single TCP conn, upgraded from HTTP | Lychee-Chat messaging |
| **GraphQL subscriptions** | server→client (over WS) | `graphql-transport-ws` | Restaurant order status |
| **Server-Sent Events (SSE)** | server→client only | plain HTTP, `text/event-stream` | Restaurant kitchen dashboard |
| **gRPC server streaming** | server→client | HTTP/2 | Restaurant orders → gateway |
| (baseline) **polling** | client pulls | HTTP | OneTube client polls video status until `ready` |

## Lychee-Chat — WebSockets + Redis pub/sub for multi-instance fan-out

Flow (from [apps/ws](https://github.com/1Kyryll/Lychee-Chat/tree/main/apps/ws)):
1. Client emits `chat:join` → server joins the socket to a **room** ([events/chat.join.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/events/chat.join.ts))
2. Client emits `message:send` → handler saves to MongoDB, then **publishes to Redis** ([events/message.send.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/apps/ws/events/message.send.ts))
3. A Redis subscriber broadcasts to all sockets in the room ([packages/redis/pubsub.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/pubsub.ts))

**Why the Redis hop when one server could broadcast directly?** Horizontal scaling. With N WebSocket server instances behind a load balancer, users in the same chat may be connected to *different* instances. In-process broadcast only reaches sockets on the same instance; publishing through Redis reaches all instances. This is the standard "WebSocket fan-out across replicas" answer — see [[Redis Pub-Sub]].

Also note: WebSocket traffic needs explicit `Upgrade`/`Connection` header handling at the proxy — see [[Reverse Proxy with Nginx]].

## Restaurant System — three push mechanisms in one pipeline

```
orders service ──gRPC server stream──► gateway ──GraphQL subscription (WS)──► customer browser
orders service ──outbox──► Kafka ──consumer──► kitchen service ──SSE──► kitchen dashboard
```

- **gRPC streaming** ([orders handlers/grpc.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/orders/handlers/grpc.go)): the gateway subscribes once and receives every new order — service-to-service push.
- **GraphQL subscriptions** ([subscription.resolvers.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/subscription.resolvers.go)): `orderStatusChanged(orderId)` lets a customer watch their own order live; auth-scoped so they can't watch others'.
- **SSE** ([kitchen handlers/http.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/kitchen/handlers/http.go)): the kitchen dashboard consumes `GET /stream`. SSE quirk I hit in practice: **`EventSource` cannot send custom headers**, so the JWT goes in a query param (`/stream?access_token=<jwt>`) — a real-world auth wrinkle worth mentioning.

## Choosing between them (interview framing)
- Need **bidirectional** (chat, games, collaborative editing) → WebSockets.
- Need **server→client only** (dashboards, notifications, progress) → SSE is simpler: plain HTTP, auto-reconnect built in, works through proxies without upgrade config. Its limits: unidirectional, header restrictions, browser connection cap on HTTP/1.1.
- **Internal service push** → gRPC streaming (typed, multiplexed, deadline-aware).
- **Low frequency / simplicity wins** → polling is fine. OneTube polls transcode status because uploads take minutes and a WS connection buys nothing.
- At scale, all stateful connections raise the same issues: sticky sessions or shared pub/sub backbone, heartbeats/reconnect logic, and fan-out cost.

## Related
- [[Redis Pub-Sub]]
- [[GraphQL]]
- [[gRPC and Protocol Buffers]]
- [[Kafka and Event Streaming]]
- [[Communication Index]]
