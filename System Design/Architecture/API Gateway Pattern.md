---
tags: [system-design, architecture]
---

# API Gateway Pattern

A single service that fronts all client traffic and fans out to internal services. The client sees one API; the internals stay free to use whatever protocol suits them.

## Why
- **Protocol translation** — browsers can't speak gRPC (natively); the gateway translates REST/GraphQL → gRPC.
- **Cross-cutting concerns in one place** — auth, rate limiting, request IDs, CORS live at the edge instead of being duplicated in every service.
- **Backend-for-frontend** — the gateway shapes responses for the UI (aggregation, joining data from multiple services).

## Where I built it

### E-Commerce: REST gateway → gRPC services
The browser talks REST to [`gateway`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway), which fans out over gRPC to catalog/cart/order:

- Routing + handlers: [`gateway/router.go`](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/gateway/router.go), [`gateway/handlers/`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway/handlers)
- gRPC clients per downstream service: [`gateway/clients/`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway/clients)
- **The gateway owns auth entirely** — JWT issue/verify in [`gateway/auth/`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway/auth); internal services trust the user ID passed to them and never see credentials. This keeps the auth perimeter at one door.
- Middleware at the edge: [`internal/middleware/`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/internal/middleware) — auth, panic recovery, request ID propagation.

### Restaurant System: GraphQL gateway → gRPC services
[`gateway`](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/internal/services/gateway) exposes GraphQL (gqlgen) and calls the orders/user services over gRPC:

- Resolvers: [`graph/mutation.resolvers.go`](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/mutation.resolvers.go), [`graph/subscription.resolvers.go`](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/subscription.resolvers.go)
- JWT claims extracted by middleware and injected into request context: [`internal/middleware/authmiddleware.go`](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/middleware/authmiddleware.go)
- Aggregation via per-request DataLoaders (see [[GraphQL]]): [`gateway/dataloaders/`](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/internal/services/gateway/dataloaders)
- Real-time: GraphQL subscriptions over WebSocket, backed by **gRPC server streaming** from the orders service (see [[Real-Time Communication]]).

## Tradeoffs to name in an interview
- The gateway is a **single point of failure** and a potential bottleneck → in production you run N replicas behind a load balancer (my E-Commerce TODO lists exactly this).
- It can silently grow into a "distributed monolith's brain" — business logic creeping into the gateway. Rule I follow: gateway does *translation, auth, aggregation*; domain logic stays in services.
- One more network hop of latency. Cheap (~ms intra-datacenter) but non-zero.

## Gateway vs Reverse Proxy
A gateway understands your API (routes, auth, aggregation). A reverse proxy ([[Reverse Proxy with Nginx]]) works at the HTTP level (routing by path/host, TLS termination, buffering). Lychee-Chat and Finance-Tracker use plain Nginx because there's nothing to translate — one backend each.

## Related
- [[Monolith vs Microservices]]
- [[gRPC and Protocol Buffers]]
- [[GraphQL]]
- [[Architecture Index]]
