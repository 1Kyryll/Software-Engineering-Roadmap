---
tags:
  - interview-qa
---

# Q: What eviction policy makes sense for a session store versus a content feed?

## My original answer
> Eviction policy decides how the cached data should be invalidated. Without it storage memory would overflow, consequently causing a crash. For a user's session data it makes sense to use LRU because users who haven't been active for a long period should be prioritized for eviction over those who are frequently using the application. Content Feed is a different story — use LFU because some posts may go viral, more people see them and they should stay in cache; we can afford not to cache unpopular posts.

## Review — mostly right, three corrections

1. **Terminology: eviction ≠ invalidation.** Eviction is *capacity-driven* removal (memory is full, something must go). Invalidation is *correctness-driven* removal (the data changed). Saying "eviction decides how data is invalidated" mixes them — an interviewer may probe exactly this distinction. Also, without a policy Redis doesn't crash — with the default `noeviction` it **rejects writes** when full; a crash isn't the failure mode.
2. **LRU-for-sessions is right, but state the deeper question first**: *can I afford to lose a session?* Evicting a session logs a user out. If that's unacceptable, no eviction policy is "right" — sessions belong in a persistent store or a `noeviction` Redis with capacity planning (same argument as [ADR 004](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/004-cart-persistent-not-session-bound.md) keeping carts out of Redis). If bounded loss is acceptable, LRU + per-session TTL is the standard combo — sessions have a natural lifetime, so TTL does most of the work and LRU handles pressure.
3. **Plain LFU has a virality problem, not a virality advantage.** Frequency counters accumulate: *yesterday's* viral post keeps a huge count and hogs cache long after it cooled, while a *newly* viral post starts at zero — LFU is slow to adapt exactly when virality moves. The fix is **LFU with decay/aging** (Redis's `allkeys-lfu` uses a decaying probabilistic counter for this reason). Worth noting LRU also performs fine for feeds: currently-viral content is re-accessed constantly, so recency tracks popularity closely.

## Say this in the interview

"Let me first separate three things that often get mixed up: **eviction** is capacity-driven — memory is full and something must go; **expiration** is TTL-driven; **invalidation** is correctness-driven — the data changed. This question is about eviction, and the answer differs because the two workloads have different answers to one underlying question: *what does losing an entry cost?*

For a **session store**, losing an entry logs a user out. So before picking a policy I'd ask whether that's acceptable at all. If it isn't, no eviction policy is 'right' — sessions shouldn't live in a store that evicts: use a persistent store, or Redis with `noeviction` and real capacity planning — and note that a full Redis under `noeviction` rejects writes, it doesn't crash. If bounded loss *is* acceptable, sessions have a natural lifetime, so I'd lean on a **per-session TTL** as the primary mechanism — it does most of the cleanup for free — and **LRU under pressure**, specifically `volatile-lru` in Redis so only TTL'd keys are eligible. The semantics match: the users evicted first are the ones who haven't been active longest, which is exactly who you'd choose to log out.

A **content feed** is the opposite profile: every entry is re-fetchable from the database, so eviction costs a cache miss, not data loss — I can evict aggressively. The access pattern is popularity-driven — a small set of hot posts takes most of the reads — which points at **LFU rather than LRU**. But plain LFU has a known failure mode: frequency counters accumulate, so yesterday's viral post keeps a huge count and stays pinned while a newly-viral post starts from zero — LFU adapts slowly exactly when virality moves. The fix is **LFU with decay**, which is what Redis's `allkeys-lfu` actually implements: a probabilistic counter that ages out. That's what I'd configure.

One operational rule on top: cache and durable state must never share an eviction domain. In my chat app, the same Redis holds cache keys *and* the BullMQ job queue — under `allkeys-lru`, memory pressure could evict pending jobs. So: separate instances, or `volatile-*` with TTLs strictly on the cache keys."

## Related
- [[Cache Eviction Policies]]
- [[Cache Invalidation]]
- [[Interview QA Index]]
