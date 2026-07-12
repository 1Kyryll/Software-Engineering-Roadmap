---
tags: [system-design, caching, redis]
---

# Caching Strategies

Trade freshness for speed: keep hot data in a fast store (Redis) in front of the database. The two design questions are always **how does data get into the cache** and **how does it get out (invalidation)**.

## Cache-aside (lazy loading) — the pattern I use

Read path: check cache → hit? return → miss? read DB, write cache, return.
Write path: write DB → **invalidate** (delete) the affected keys. Next read repopulates.

### Implemented: Lychee-Chat

[packages/redis/redisCache.ts](https://github.com/1Kyryll/Lychee-Chat/blob/main/packages/redis/redisCache.ts) — get/set with optional TTL, delete for invalidation, and **centralized key builders**:

```ts
export const cacheKeys = {
    chatMessages: (chatId: string) => `chat:${chatId}:messages`,
    contact: (ownerId: string, userId: string) => `contact:${ownerId}:${userId}`,
    contacts: (ownerId: string) => `contacts:${ownerId}`,
};
```

Caches are **invalidated on write**: a new message deletes `chat:{chatId}:messages`; adding/removing a contact deletes the owner's contact keys. Key-naming conventions (`entity:id:collection`) centralized in one module prevent the classic "writer and invalidator disagree about the key string" bug.

### Designed: E-Commerce catalog ([ADR 006](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/006-catalog-reads-cache-aside-cursor-pagination.md))

Catalog reads go cache-first for the stable fields (name, description, image) while **inventory count is deliberately excluded from the cache**: it changes constantly and must be consistent — a stale "in stock" would let a user click through only to get 409'd anyway, but a stale count on the checkout path would be a correctness bug. The ADR's rejected alternatives: *no cache* (every page render hits the DB — doesn't scale) and *cache the inventory too* (consistency-critical data doesn't belong in a cache). Redis is provisioned in [docker-compose.yml](https://github.com/1Kyryll/E-Commerce-System/blob/main/docker-compose.yml); wiring the cache layer is a live TODO.

## The decisions every cache design must make

1. **What NOT to cache** — consistency-critical, high-churn values (inventory). Knowing this is worth more than knowing the patterns.
2. **Invalidation strategy** — delete-on-write vs TTL-only vs both. Full treatment (including the stale-refill race): [[Cache Invalidation]].
3. **Cache stampede** — a hot key expires and 1000 requests hit the DB simultaneously. Full treatment: [[Cache Stampede]].
4. **Eviction under memory pressure** — LRU/LFU, Redis `maxmemory-policy`, and why cache and queue keys must not share an eviction domain: [[Cache Eviction Policies]].
5. **Write-through / write-behind** — alternatives where writes go through the cache; write-behind risks loss on cache crash. I default to cache-aside because the DB stays the unambiguous source of truth.

## Related concepts in the vault
- [[Denormalization and Invariants]] — a materialized column is "a cache inside the DB" with transactional invalidation
- [[Redis Pub-Sub]] — same Redis, different job
- [[Pagination - Cursor vs Offset]] — ADR 006 couples the two for catalog reads
- [[Caching Index]]
