---
tags: [system-design, communication, rest]
---

# REST API Design

Resource-oriented HTTP APIs. Every project I've built exposes one; the interesting parts are the conventions that make them production-grade.

## Where I built it

| Project | API |
|---|---|
| [E-Commerce gateway handlers](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/gateway/handlers) | `/products`, `/cart/items`, `/checkout`, `/orders/{id}`, `/me`, `/healthz` |
| [OneTube API table](https://github.com/1Kyryll/OneTube#api) | `/api/videos`, `/api/videos/{id}/complete`, `/api/videos/feed`, `/api/auth/*` |
| [Personal-Finance-Tracker](https://github.com/1Kyryll/Personal-Finance-Tracker) | Flask REST API under `/api`, consumed by the React SPA |

## Conventions I actually apply (with evidence)

**Nouns for resources, verbs from HTTP methods.** `POST /videos` creates an upload intent, `DELETE /videos/{id}` soft-deletes, `GET /videos/feed` lists. State transitions that don't map cleanly onto CRUD get a sub-resource action: `POST /videos/{id}/complete`, `POST /order/{orderId}/done`.

**Status codes carry semantics.**
- E-Commerce checkout returns **`201 Created`** with the order, **`409 Conflict`** when inventory is insufficient (the atomic decrement matched no rows — see [[Database Concurrency and Atomicity]]). The k6 load tests explicitly assert `'checkout ok or 409'` ([load-tests/checkout.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/checkout.js)) because under contention 409 is a *correct* answer, not an error.
- `401` unauthenticated vs `403` authenticated-but-forbidden (Restaurant RBAC — see [[Authentication and Authorization]]).

**Idempotency for unsafe retries.** Checkout requires an `Idempotency-Key` header; replays return the original order instead of double-charging. Full pattern in [[Idempotency]] and [ADR 003](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/003-idempotency-on-order-creation.md).

**Cursor pagination, never OFFSET.** OneTube's design rules say it outright: "Cursor pagination on `(published_at, id)`. Never `OFFSET`." Details in [[Pagination - Cursor vs Offset]].

**Health endpoints.** `/healthz` on the E-Commerce gateway ([handlers/healthz.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/gateway/handlers/healthz.go)) is what docker-compose polls to gate dependent services (`condition: service_healthy`).

**Ownership enforced in the query, not the handler.** OneTube mutations: `WHERE id = ? AND uploader_id = ?` — a non-owner gets zero affected rows, which is both authorization and a 404 in one atomic step. No TOCTOU gap between "check ownership" and "mutate."

**Auth via HttpOnly cookies, not localStorage.** All my apps set the JWT as `HttpOnly; Secure; SameSite=Lax` — JavaScript can't read it, which kills XSS token theft. See [[Authentication and Authorization]].

## Interview checklist
- Versioning (`/v1/`) — I haven't needed it yet; know the options (URL, header, media type).
- Error body shape — consistent JSON envelope (`{"error": {"code", "message"}}`).
- `PUT` vs `PATCH` vs `POST` — idempotent replace vs partial update vs create/action.
- REST's limits → when to reach for [[gRPC and Protocol Buffers]] (internal, typed, streaming) or [[GraphQL]] (client-shaped reads).

## Related
- [[API Gateway Pattern]]
- [[Idempotency]]
- [[Pagination - Cursor vs Offset]]
- [[Communication Index]]
