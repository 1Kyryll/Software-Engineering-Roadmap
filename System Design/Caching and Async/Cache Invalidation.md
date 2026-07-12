---
tags: [system-design, caching, consistency]
---

# Cache Invalidation

"There are only two hard things in Computer Science: cache invalidation and naming things." It's hard because it's a **distributed consistency problem in disguise**: the cache and the database are two systems, updated non-atomically, read concurrently. Every strategy is a different answer to "how stale can this read be, and what does that staleness cost?"

## Strategy 1 — Delete-on-write (what Lychee-Chat does)

Write to the DB, then `DEL` the affected cache keys; the next read repopulates from fresh data ([redisCache.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/redisCache.ts)): new message → delete `chat:{chatId}:messages`; contact change → delete `contacts:{ownerId}` and `contact:{ownerId}:{userId}`.

Two details in that implementation worth defending:
- **Delete, don't update, the cache on write.** Writing the new value into the cache directly is tempting (saves a miss) but racy: two concurrent writers can interleave so the cache ends up with the older value *forever*. Deletion is self-healing — worst case is one extra DB read.
- **Centralized key builders** (`cacheKeys.contacts(ownerId)`) — writer and invalidator can't drift apart on the key string. Cache keys are a schema; treat them like one.

### The race that survives even correct delete-on-write
1. Reader misses cache, reads DB (old value) … pauses (GC, network)
2. Writer updates DB, deletes cache key
3. Reader resumes, **fills the cache with the stale value it read in step 1** → stale until the next invalidation/TTL

Low probability, nonzero, and famous (it's why "cache invalidation is hard"). Mitigations: **TTL as a backstop** (bounds the damage window — always set one), delayed double-delete (delete again after ~500ms), or compare-and-set with versions. For chat lists, TTL-backstop is proportionate; for money, don't cache it at all (see below).

## Strategy 2 — TTL-only (bounded staleness, zero invalidation code)
No write hooks; entries just expire. Right when staleness has a known acceptable bound and writes are frequent or writers are many/unknown. [ADR 006](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/006-catalog-reads-cache-aside-cursor-pagination.md) accepts catalog data "stale for a couple of seconds" explicitly — a design *decision*, with the consequence reasoned through (a stale in-stock flag just earns a 409 at checkout, where the DB is the truth).

## Strategy 3 — Event-driven invalidation
Publish invalidation events (Redis pub/sub, or CDC from the DB log) and let every cache holder subscribe. Needed when caches live in many processes (in-memory per-instance caches) that a writer can't reach with a `DEL`. My infrastructure for this already exists — the [[Redis Pub-Sub]] backbone and the [[Transactional Outbox]] are exactly the delivery mechanisms; an `order_placed` outbox event invalidating a product cache entry would make invalidation *transactional with the write*.

## Strategy 4 — Versioned/namespaced keys (invalidation by never invalidating)
Include a version in the key: `products:v42:page1`. To "invalidate," bump the version — new reads assemble new keys, old entries die by eviction/TTL ([[Cache Eviction Policies]]). O(1) invalidation of arbitrarily many derived keys (every page of a listing at once — note Lychee's per-chat message key invalidates a whole conversation's cache with one DEL only because it's a single key; versioning generalizes that).

## The decision framework
1. **Who writes?** One service (delete-on-write is tractable) or many (TTL/events)?
2. **What does a stale read cost?** Chat contact list: nothing. Product name: nothing. Inventory count: money — so it's *excluded from the cache entirely* ([[Caching Strategies]]). The best invalidation strategy is not caching the thing that can't be stale.
3. **How many keys does one write dirty?** One → targeted delete. A family → versioned namespace.
4. **Always set a TTL anyway.** Every invalidation path has a bug someday; TTL turns "stale forever" into "stale for minutes."

## Related
- [[Caching Strategies]]
- [[Cache Stampede]] — invalidating a hot key creates a synchronized miss
- [[Cache Eviction Policies]]
- [[Denormalization and Invariants]] — the in-DB analogue with transactional "invalidation"
- [[Caching Index]]
