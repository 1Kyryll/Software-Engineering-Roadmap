---
tags: [index, system-design]
---

# Communication Index

How services and clients talk: request/response protocols and push channels.

Up: [[System Design MOC]]

## Notes
- [[REST API Design]] — resources, status-code semantics, ownership-in-the-query, health endpoints
- [[gRPC and Protocol Buffers]] — contract-first internal RPC, streaming, deadline propagation
- [[GraphQL]] — client-shaped reads, N+1, per-request DataLoaders
- [[Real-Time Communication]] — WebSockets vs SSE vs gRPC streaming vs polling, chosen per case

## Connects to
- [[Architecture Index]] — gateways translate between these protocols
- [[Messaging Index]] — async messaging is the other half of inter-service communication
- [[Authentication and Authorization]] — every channel needs an auth story (including SSE's header limitation)
