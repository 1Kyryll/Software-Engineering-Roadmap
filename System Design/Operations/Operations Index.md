---
tags: [index, system-design]
---

# Operations Index

Running, observing, and proving the system: the "does it actually work under load" cluster.

Up: [[System Design MOC]]

## Notes
- [[Observability]] — OpenTelemetry → Grafana LGTM; spans per phase, outcome-labeled metrics
- [[Load Testing with k6]] — scenario design, think time, why 409 is a pass
- [[Docker Compose Orchestration]] — healthcheck-gated startup, one-shot migrations, env anchors, CI/CD

## Connects to
- [[Data Index]] — invariant reconciliation exported as a metric; pool sizes vs max_connections
- [[Architecture Index]] — compose is how the services and proxies get wired
- [[Data Index]] / [[Caching Index]] — load tests are what prove the concurrency and caching designs
