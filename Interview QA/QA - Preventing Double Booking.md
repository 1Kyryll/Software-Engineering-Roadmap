---
tags:
  - interview-qa
---

# Q: How do you prevent double booking in a distributed system?

## My original answer
> To prevent race conditions and double bookings in a distributed system, we should utilize distributed locking. Common implementations include Redis Redlock or ZooKeeper. Which one depends on business logic constraints: for highly performant logic but not necessarily strict and correct, lean on Redis; ZooKeeper has consensus protocols like Raft and Paxos that guarantee lock information is strongly consistent and highly reliable under network partitions.

## Review — leads with the wrong tool ⚠️

The content about Redis-vs-ZooKeeper tradeoffs is fine, but **distributed locking as the *first* answer is the trap in this question** — and it contradicts my own [ADR 001](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/001-inventory-atomic-decrement.md), which explicitly *rejected* Redis locks for exactly this problem.

1. **The strongest answer is: don't distribute the decision at all.** A booking conflict is about one resource (a seat, a unit of stock), and that resource has one authoritative home — a database row. A single Postgres already serializes writes to a row; pushing mutual exclusion into it is simpler, crash-safe (transaction rollback releases everything), and has no TTL races. The system being "distributed" (many app servers) doesn't matter — N stateless services contending on one DB row is *the* solved problem. Concretely, two shapes:
   - **Unique constraint**: `INSERT INTO bookings (event_id, seat_id, ...)` with `UNIQUE (event_id, seat_id)` — two concurrent inserts, one wins, the loser gets a constraint violation → "seat taken." The database is the arbiter; no read-check-write window exists.
   - **Atomic conditional decrement** (capacity-N): `UPDATE ... SET available = available - 1 WHERE id = $1 AND available > 0 RETURNING id` — check and write in one atomic statement. This is literally my e-commerce checkout: 10K concurrent buyers, 10 units, exactly N successes ([[Database Concurrency and Atomicity]]).
2. **The full booking flow needs two more pieces the answer skipped**: a **TTL reservation** for the hold-during-payment window (hold the seat without blocking others; release on failure/expiry — [ADR 002](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/002-reservation-table-payment-window.md)), and an **idempotency key** so a retried "book" request doesn't double-book ([[Idempotency]]).
3. **Distributed locks are the fallback, and the pitfalls need precision.** Redlock's safety is famously contested (Kleppmann's "How to do distributed locking"): a GC pause or network delay longer than the lock TTL yields **two lock holders**, and the fix — **fencing tokens** checked by the storage — only works if the storage can check them, at which point the storage could have enforced the constraint itself. Factual fix: **ZooKeeper runs ZAB** (its own atomic broadcast protocol), not Raft/Paxos — **etcd** is the one using Raft. The Redis-fast-but-weaker / ZK-consistent-but-heavier instinct was right.
4. When *is* a distributed lock legitimate here? When there's genuinely **no single authoritative store** — e.g., coordinating a booking across two independent systems (airline + hotel), or protecting a non-transactional external resource. Even then it coordinates *work*; the final conflict decision should still land on some store's atomic operation.

## Say this in the interview

"My first move is to *not* solve this with distributed locks — a booking conflict is about one resource, and that resource has one authoritative home: a database row. The database already serializes writes to a row, so I push the decision there. Two shapes depending on the resource: if it's a specific seat, a **unique constraint** on `(event_id, seat_id)` — two concurrent inserts race, exactly one wins, the other gets a constraint violation that I map to 'seat taken.' If it's N units of capacity, an **atomic conditional decrement** — `UPDATE ... SET available = available - 1 WHERE available > 0 RETURNING id` — the check and the write are one atomic statement, so there's no window between 'looks free' and 'is mine.' This works at default Read Committed isolation, holds the row lock for microseconds, and a crash rolls back cleanly. I built exactly this in my e-commerce project — ten thousand concurrent buyers on ten units resolves to exactly ten successes.

A real booking flow needs two more pieces: a **reservation with a TTL** so the seat is held during the payment window without blocking anyone forever — payment succeeds, the reservation is consumed; it fails or expires, a cleanup process releases it and restores availability in the same transaction. And an **idempotency key** on the booking request, enforced as a unique constraint, so a client retry after a network blip returns the original booking instead of creating a second one.

Distributed locks — Redlock, ZooKeeper, etcd — are the fallback for when there's genuinely no single authoritative store, like coordinating across two independent systems. And they have sharp edges worth naming: a lock with a TTL can expire while its holder is paused in a GC, giving you two holders — Kleppmann's critique of Redlock is exactly this — and the fix, fencing tokens, requires the storage to validate tokens, at which point the storage could have enforced the constraint directly. If I need one anyway: ZooKeeper or etcd for correctness — they're built on consensus, ZAB and Raft respectively — Redis single-instance locks only where a rare double-execution is tolerable. But the headline is: constraint or atomic write in the data's home first; locks are coordination, not correctness."

## Related
- [[Database Concurrency and Atomicity]]
- [[Locking]]
- [[Idempotency]]
- [[Transaction Isolation Levels]]
- [[Interview QA Index]]
