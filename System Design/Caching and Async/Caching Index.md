---
tags: [index, system-design]
---

# Caching Index

Keeping hot data fast without lying about it: population, invalidation, eviction, and the herd.

Up: [[System Design MOC]]

## Notes
- [[Caching Strategies]] — cache-aside as the default; what NOT to cache (start here)
- [[Cache Invalidation]] — delete-on-write, the stale-refill race, versioned keys, TTL backstops
- [[Cache Eviction Policies]] — LRU/LFU, Redis maxmemory-policy, cache vs durable-state eviction domains
- [[Cache Stampede]] — synchronized misses; singleflight, jitter, stale-while-revalidate

## Connects to
- [[Data Index]] — [[Denormalization and Invariants]] is "caching inside the DB" with transactional invalidation
- [[Messaging Index]] — Redis pub/sub and outbox events as invalidation transport; BullMQ shares the Redis
- [[Architecture Index]] — [[Rate Limiting]] counters live in the same Redis
