---
tags: [system-design, architecture, reliability, security]
---

# Rate Limiting

Bound how many requests a client may make per time window. It's simultaneously a security control (brute-force login, scraping), a fairness control (one client can't starve the rest), and a stability control (backpressure before overload). My repos are honest about its absence — OneTube's known gaps: *"No rate limiting, captcha, or email verification"* — and the natural home in my architectures is the edge: the [[API Gateway Pattern]] gateway or [[Reverse Proxy with Nginx]].

## The algorithms

### Fixed window
Counter per `(client, window)`: `rate:{user}:{floor(now/60s)}`, increment on each request, reject over the limit, key expires with the window.

- ✅ Trivial: one Redis `INCR` + `EXPIRE`, O(1) memory.
- ❌ **Boundary burst**: 100/min limit → 100 requests at 0:59 and 100 more at 1:01 = 200 in two seconds, all "legal." Each window forgets the past completely.

### Sliding window log
Store the *timestamp of every request* (Redis sorted set); on each request, drop entries older than the window, count the rest.

```
ZREMRANGEBYSCORE rate:{user} 0 (now - 60s)
ZCARD rate:{user}                      → reject if ≥ limit
ZADD rate:{user} now now  ·  EXPIRE rate:{user} 60
```

- ✅ Exact — no boundary artifacts at all.
- ❌ Memory scales with request count per client per window; a few thousand heavy clients at high limits gets expensive.

### Sliding window counter (the pragmatic middle)
Keep the two adjacent fixed-window counters and weight them by overlap:

```
estimate = curr_count + prev_count × (overlap of prev window with the sliding window)
```

30s into a 60s window: `curr + prev × 0.5`. Assumes the previous window's traffic was uniform — a smoothing approximation that eliminates the boundary burst at fixed-window cost (two counters). This is the industry default (Cloudflare's published approach).

### Token bucket (the other family — allows controlled bursts)
Bucket of capacity B refills at R tokens/sec; each request takes a token; empty = rejected. **Enforces an average rate while permitting bursts up to B** — matches real client behavior (page load = burst of 10 calls, then quiet). Leaky bucket is the queue-shaped cousin (constant outflow — smooths rather than admits bursts). Nginx `limit_req` is leaky-bucket with a `burst` knob — free at my existing Nginx layer:

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
location /api/ { limit_req zone=api burst=20 nodelay; }
```

## Distributed rate limiting (the follow-up question)
Per-instance in-memory counters multiply the real limit by the replica count and break on uneven load balancing. Centralize the state in Redis — atomic via a Lua script or `INCR`-based fixed/sliding-counter keys — and every gateway replica shares one budget. Tradeoffs to name: +1 network hop per request, Redis as a dependency (fail-open or fail-closed? — availability vs strictness), and hot-key pressure for very hot clients.

## API behavior
- **`429 Too Many Requests`** + `Retry-After` header; `X-RateLimit-Limit / -Remaining / -Reset` as courtesy.
- **Key choice matters**: per-user for authenticated routes ([[Authentication and Authorization]] gives the ID), per-IP for anonymous (imperfect: NAT shares IPs, botnets rotate them), per-API-key for B2B. Layered limits (per-IP *and* per-account) close the obvious workarounds.
- Different budgets per route class: login endpoints get tiny strict limits (brute force), catalog reads get generous ones. A 409-heavy checkout under flash-sale load ([[Load Testing with k6]]) is *not* a rate-limiting problem — don't paper over contention with 429s.

## Where it belongs in my systems
- **E-Commerce**: gateway middleware (sliding window counter in the existing Redis), per-user on `/checkout`, per-IP on `/auth/login`. Slots next to auth/request-ID middleware in [internal/middleware/](https://github.com/1Kyryll/E-Commerce-System/tree/main/backend/internal/middleware).
- **OneTube**: per-user token bucket on upload-intent creation (`POST /videos`) — presigned-URL issuance is the abuse surface for a storage-cost attack ([[Object Storage and Presigned URLs]]).
- **Lychee-Chat**: message-send rate per socket (spam), enforceable in the Socket.IO handler with the same Redis.

## Related
- [[API Gateway Pattern]]
- [[Reverse Proxy with Nginx]]
- [[Authentication and Authorization]]
- [[Architecture Index]]
