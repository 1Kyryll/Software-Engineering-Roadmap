---
tags: [index, system-design]
---

# Architecture Index

How systems are shaped and fronted: deployment units, edges, and traffic control.

Up: [[System Design MOC]]

## Notes
- [[Monolith vs Microservices]] — the spectrum across my five projects, when to split and what it costs
- [[API Gateway Pattern]] — REST→gRPC (E-Commerce) and GraphQL (Restaurant) gateways; auth perimeter at the edge
- [[Reverse Proxy with Nginx]] — path routing, WebSocket upgrade headers, SPA+API split
- [[Rate Limiting]] — fixed/sliding window, token bucket, distributed limits in Redis

## Connects to
- [[Communication Index]] — the gateway is where protocol translation happens
- [[Authentication and Authorization]] — auth and rate limiting both live at the edge
- [[Operations Index]] — compose orchestration wires these services together
