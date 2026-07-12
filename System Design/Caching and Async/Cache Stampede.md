---
tags: [system-design, caching, reliability]
---

# Cache Stampede

(aka thundering herd, dog-piling.) A hot key expires — say the E-Commerce catalog's first page, read on every storefront render — and in the next instant **every concurrent request misses simultaneously and goes to Postgres**. A cache that was absorbing 5,000 req/s hands the database 5,000 req/s of the *same query* at once. The DB slows, misses pile up further, and a healthy system falls over from one key's expiry. The worst case is a full cache restart (everything cold at once).

The cruel part: cache-aside ([[Caching Strategies]]) has this failure mode *built in* — "miss → read DB → fill" is correct for one request and catastrophic for ten thousand identical ones.

## Mitigations (know several; they compose)

**1. Request coalescing / per-key lock (singleflight).** Only the *first* miss recomputes; everyone else waits for that result (or briefly serves stale). In Go this is literally `golang.org/x/sync/singleflight` — one in-flight DB query per key per process:

```go
val, err, _ := group.Do("products:page1", func() (any, error) {
    return repo.ListByCursor(ctx, cursor, limit)   // runs once; concurrent callers share the result
})
```

Cross-instance version: a short-TTL Redis lock (`SET stampede:products:page1 1 NX PX 3000`) — lock holder recomputes, others serve stale or retry. This is the strongest general fix and the one I'd cite first for the catalog cache (ADR 006's design).

**2. TTL jitter.** Never give a large key population the same lifetime — `TTL + rand(0..10%)` — or keys warmed together (deploy, cache flush, bulk seed) expire together and stampede *as a cohort*.

**3. Probabilistic early refresh (XFetch).** Each read *may* voluntarily refresh before expiry, with probability rising as expiry approaches. Statistically one request refreshes early; the expiry cliff never arrives. Elegant, lockless.

**4. Stale-while-revalidate.** Serve the expired value immediately; refresh asynchronously in the background. Users get slightly stale data, the DB gets one refresh, latency stays flat. For a product listing (already tolerating staleness by design — [ADR 006](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/006-catalog-reads-cache-aside-cursor-pagination.md) accepts "stale for a couple of seconds") this is free correctness-wise.

**5. External recomputation.** Don't tie refresh to request traffic at all: a background job rewrites the hot keys on a schedule; readers never miss. Only worth it for a small, known-hot set (front page, config).

## How this maps to my systems
- **Catalog first page** is the textbook stampede candidate: one extremely hot key, expensive-ish query, every anonymous visitor reads it. Design: cache-aside + singleflight + jittered TTL + stale-while-revalidate for the front page.
- **Lychee-Chat** is naturally stampede-resistant: keys are per-chat/per-user (`chat:{id}:messages`), so no single key concentrates global traffic — a reminder that **key cardinality shapes stampede risk**. The risk there is instead invalidation-driven: a busy group chat invalidates its key on every message, so every member's next read misses together. Same medicine (coalescing), different trigger.
- Related-but-different: locks/reservations under contention ([[Database Concurrency and Atomicity]]) — there the herd is *writers* and the answer is atomic decrement, not caching.

## Interview sound bite
"TTL expiry is a synchronization point: everyone discovers the miss together. Every fix works by breaking that synchronization — serialize the recomputation (locks/singleflight), desynchronize the expiry (jitter, early refresh), or decouple serving from recomputation (stale-while-revalidate, background refresh)."

## Related
- [[Caching Strategies]]
- [[Cache Invalidation]]
- [[Cache Eviction Policies]]
- [[Caching Index]]
