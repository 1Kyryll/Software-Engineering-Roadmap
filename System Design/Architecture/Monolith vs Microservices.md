---
tags: [system-design, architecture]
---

# Monolith vs Microservices

I've built both, so I can argue this from experience rather than blog posts.

## The spectrum in my projects

| Project | Style | Why it fits |
|---|---|---|
| [Personal-Finance-Tracker](https://github.com/1Kyryll/Personal-Finance-Tracker) | **Monolith** — one Flask API + React SPA + Postgres behind Nginx | CRUD app, one team (me), one deploy unit. Anything more would be ceremony. |
| [Lychee-Chat](https://github.com/1Kyryll/Lychee-Chat) | **Modular monorepo** — `apps/web`, `apps/ws`, shared `packages/db` + `packages/redis` (npm workspaces) | Two runtime processes (Next.js + Socket.IO server) sharing code via workspace packages; not microservices, but no longer one process. |
| [OneTube](https://github.com/1Kyryll/OneTube) | **API + worker split** — one Go module, two binaries: [`cmd/api`](https://github.com/1Kyryll/OneTube/tree/main/server/cmd/api) and [`cmd/worker`](https://github.com/1Kyryll/OneTube/tree/main/server/cmd/worker) | CPU-heavy transcoding must not block request serving; queue decouples them. First real step away from a monolith, driven by a *workload* difference, not fashion. |
| [E-Commerce-System](https://github.com/1Kyryll/E-Commerce-System) | **Microservices** — gateway, catalog, cart, order, cleanup-worker ([docker-compose.yml](https://github.com/1Kyryll/E-Commerce-System/blob/main/docker-compose.yml)) | Services scale/deploy independently; order service owns the hard concurrency logic in isolation. |
| [Restaurant-System-Microservices](https://github.com/1Kyryll/Restaurant-System-Microservices) | **Event-driven microservices** — gateway, orders, kitchen, user + Kafka | Kitchen reacts to order events asynchronously; services communicate via gRPC + Kafka. |

## Key interview points

**Start with a monolith.** Finance-Tracker's whole architecture is `Browser → Nginx → Flask → Postgres` — deployable in minutes, debuggable with one log stream. Microservices buy you independent scaling, deploys, and failure isolation; they cost you network calls where function calls used to be, distributed transactions ([[Transactional Outbox]]), service discovery, and observability overhead ([[Observability]]).

**Split on real forces, not aesthetics.** The legitimate reasons I've hit personally:
- *Different workload shapes* — OneTube's transcoding worker is CPU/ffmpeg-bound while the API is I/O-bound. Scaling them together wastes money.
- *Different failure/latency budgets* — E-Commerce checkout must be transactional and fast; email/analytics fan-out can lag by seconds (hence the outbox + async workers).
- *Independent contention domains* — the order service owns inventory contention; the catalog service is read-heavy and cacheable. Different tuning, different code.

**Shared DB vs DB-per-service.** Both my microservice projects use one Postgres instance with logically separated tables — a pragmatic middle ground. True DB-per-service adds data duplication and eventual consistency; a shared schema couples deploys. Know that the "pure" answer is DB-per-service, and be able to say why you'd defer it (single team, single region, transactional integrity across order+inventory is much easier in one DB).

**A modular monolith is a valid endpoint, not just a waypoint.** Lychee-Chat's workspace packages (`@packages/db`, `@packages/redis`) give code-level modularity without operational overhead.

## The distributed-systems tax (what you sign up for)
- Network partial failure between services → retries, timeouts, [[Idempotency]]
- No cross-service transactions → [[Transactional Outbox]], sagas
- Debugging spans processes → distributed tracing ([[Observability]])
- Local dev requires orchestration → [[Docker Compose Orchestration]]

## Related
- [[API Gateway Pattern]]
- [[Architecture Index]]
