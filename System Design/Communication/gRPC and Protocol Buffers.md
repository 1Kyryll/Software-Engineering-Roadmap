---
tags: [system-design, communication, grpc]
---

# gRPC and Protocol Buffers

RPC framework over HTTP/2 with Protobuf as the typed, binary wire format. My default for **service-to-service** communication; REST/GraphQL stay at the edge for browsers.

## Why gRPC internally
- **Contract-first**: the `.proto` file *is* the API. Both sides generate typed stubs — an entire class of "field renamed, JSON silently null" bugs disappears at compile time.
- **Performance**: binary encoding + HTTP/2 multiplexing beats JSON-over-HTTP/1.1 for chatty internal traffic.
- **Streaming built in**: server/client/bidirectional streams (I use server streaming for real-time fan-out — see below).
- Why *not* for browsers: no native browser support (needs grpc-web proxy), binary payloads are hostile to debugging with curl. Hence the [[API Gateway Pattern]].

## Where I built it

### E-Commerce — contracts under `proto/`, managed with buf
- Contracts: [`proto/catalog`](https://github.com/1Kyryll/E-Commerce-System/tree/main/proto/catalog), [`proto/cart`](https://github.com/1Kyryll/E-Commerce-System/tree/main/proto/cart), [`proto/order`](https://github.com/1Kyryll/E-Commerce-System/tree/main/proto/order), with [`buf.yaml`](https://github.com/1Kyryll/E-Commerce-System/blob/main/proto/buf.yaml) + [`buf.gen.yaml`](https://github.com/1Kyryll/E-Commerce-System/blob/main/proto/buf.gen.yaml) for linting and codegen (`make generate-backend`; generated code is **gitignored** and rebuilt in CI — the proto is the source of truth, not the generated Go).
- Each service implements its proto in `services/<name>/handler/` (e.g. [order handler](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/handler/handler.go)); the gateway consumes them via typed clients in [`gateway/clients/`](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway/clients).
- gRPC calls are traced with `otelgrpc` interceptors so a checkout trace spans gateway → order → Postgres (see [[Observability]]).

### Restaurant System — gRPC + server streaming
- Protobuf definitions: [`server/protobuf/`](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/protobuf); orders and user services each expose gRPC ([orders grpc.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/orders/handlers/grpc.go), [user grpc.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/user/handlers/grpc.go)).
- **gRPC server streaming** feeds the GraphQL `orderCreated` subscription: the orders service streams new orders to the gateway, which relays them to browsers over WebSocket. Kafka events additionally use **Protobuf payloads** on the wire (topic `orders.events`), so even async messages are schema'd.

## Concepts to be fluent in
- **Field numbers are the contract.** Renaming a field is free; changing its number/type is a breaking change. Never reuse numbers of deleted fields (`reserved`).
- **Backward/forward compatibility**: unknown fields are ignored on decode → old clients survive new servers and vice versa, *if* you only add optional fields.
- The four call types: unary, server-streaming, client-streaming, bidi.
- **Deadlines propagate**: `context.Context` deadlines flow across the wire; a gateway timeout cancels downstream work instead of orphaning it. This is why "context is the first arg of any I/O function" is a hard rule in my repos.
- Status codes are gRPC's own (`NotFound`, `AlreadyExists`, `ResourceExhausted`...) — the gateway maps them back to HTTP (`NotFound` → 404, `ResourceExhausted`/inventory → 409).

## Related
- [[API Gateway Pattern]]
- [[Real-Time Communication]] — server streaming as a push channel
- [[Kafka and Event Streaming]] — protobuf on the topic
- [[Communication Index]]
