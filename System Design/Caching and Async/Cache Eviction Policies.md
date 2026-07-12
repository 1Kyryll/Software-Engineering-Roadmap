---
tags: [system-design, caching, redis]
---

# Cache Eviction Policies

A cache is by definition smaller than the data behind it — eventually it fills, and something must go. **Eviction** (capacity pressure decides) is distinct from **expiration** (TTL decides) and from **invalidation** (correctness decides, see [[Cache Invalidation]]). All three remove entries; conflating them causes real design errors.

## The classic policies

| Policy | Evicts | Good when | Weakness |
|---|---|---|---|
| **LRU** (least recently used) | oldest *access* | recency predicts re-use (most web workloads) — the default | one big scan flushes the whole cache ("scan pollution") |
| **LFU** (least frequently used) | lowest *access count* | stable hot set (popular products, celebrity profiles) | slow to adapt: yesterday's hot key hogs space; needs decay |
| **FIFO** | oldest *insert* | simplicity; roughly uniform access | ignores popularity entirely |
| **Random** | anything | cheap, surprisingly OK approximation | no intelligence |
| **TTL-based** (volatile-ttl) | closest to expiry | entries have meaningful, varied TTLs | TTL ≠ value |

Redis implements approximated LRU/LFU (samples N keys rather than tracking a perfect global order — a deliberate memory/accuracy tradeoff worth name-dropping).

## Redis `maxmemory-policy` — the decision that actually gets made

| Setting | Meaning |
|---|---|
| `noeviction` | writes fail when full — **default**, and the right choice when Redis holds *data* (queues!) rather than cache |
| `allkeys-lru` | evict any key by LRU — pure cache, the usual pick |
| `volatile-lru` | evict only keys *that have a TTL* — mixed cache + durable keys in one instance |
| `allkeys-lfu` / `volatile-lfu` | frequency instead of recency |
| `volatile-ttl` | evict soonest-to-expire first |

## Applying it to my own systems (the interview answer)

**Lychee-Chat's Redis wears two hats, and they want different policies.** The same Redis holds cache keys (`chat:{id}:messages`, `contacts:{owner}` — [redisCache.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/redisCache.ts)) *and* **BullMQ's job queue** ([[Message Queues and Background Workers]]). Under `allkeys-lru`, memory pressure could evict *pending jobs* — silently losing queued image-compression work. Correct configs: `volatile-lru` with TTLs on cache keys only (queue keys have no TTL → never evicted), or better, **separate Redis instances** — `allkeys-lru` for cache, `noeviction` for the queue. This "cache and durable state must not share an eviction domain" point is exactly the kind of thing eviction questions are probing for.

**E-Commerce ADR 004 is an eviction argument in disguise.** Why is the cart in Postgres instead of Redis sessions? _"Server-side session in Redis: lost on Redis eviction or restart"_ ([ADR 004](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/004-cart-persistent-not-session-bound.md)). A cart is *state the business can't lose* → it doesn't belong in a store whose contract includes eviction. Eviction policy choice starts with "is losing this entry acceptable?" — if no, it's not cache.

**Catalog cache (ADR 006)**: pure cache-aside over Postgres — everything is re-fetchable, so `allkeys-lru` + TTL safety net is the natural fit.

## Common cases → common answers
- **Pure cache over a DB** → `allkeys-lru` (or LFU for a stable popularity distribution), plus TTLs as a staleness backstop.
- **Sessions you'd rather not lose** → don't rely on eviction being rare: persistent store, or Redis with `noeviction` + real capacity planning (or accept loss explicitly, like ADR 004 refused to).
- **Queues/locks/counters in Redis** → `noeviction`, always. Eviction here = data loss, not a cache miss.
- **Mixed instance** → `volatile-*` and discipline about which keys get TTLs — or just split instances.

## Related
- [[Caching Strategies]]
- [[Cache Invalidation]]
- [[Cache Stampede]] — what mass expiry does to the database
- [[Caching Index]]
