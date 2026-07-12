---
tags: [system-design, database, concurrency]
---

# Locking

When concurrent actors touch shared state, someone must serialize the conflicting part. The options form a ladder — from "let the database's row locks handle it implicitly" up to "distributed lock across machines" — and my [ADR 001](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/001-inventory-atomic-decrement.md) is literally a walk down this ladder, rejecting each rung it didn't need.

## Pessimistic locking — lock first, then work

**`SELECT ... FOR UPDATE`**: lock the rows now; anyone else wanting them blocks until you commit.

```sql
BEGIN;
SELECT inventory_available FROM products WHERE id = $1 FOR UPDATE;
-- check in app code, then:
UPDATE products SET inventory_available = inventory_available - 1 WHERE id = $1;
COMMIT;
```

Correct, easy to reason about, and the right call when the read-decide-write logic is genuinely too complex for one statement (multi-row financial transfers, seat maps). Cost under contention: everyone queues single-file on the hot row *for the whole transaction duration* — ADR 001 rejects it for checkout: "high contention, waiting times for customers... nice for a banking system, not e-commerce."

Variants that matter in practice:
- **`FOR UPDATE NOWAIT`** — error instead of waiting (fail fast, let the app decide)
- **`FOR UPDATE SKIP LOCKED`** — skip locked rows entirely; *the* trick for using a table as a job queue. My [cleanup worker](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/cleanup.go) processes expired reservations in batches; running **multiple** cleanup workers safely would be exactly `SELECT ... FROM reservations WHERE status='active' AND expires_at < now() LIMIT 100 FOR UPDATE SKIP LOCKED` — each worker grabs a disjoint batch, zero coordination.
- **Lock ordering** is the deadlock defense: two transactions locking rows A,B in opposite orders deadlock (Postgres detects, kills one). Rule: always lock in a canonical order (e.g. sort IDs) — my multi-item checkout reserves items in cart order, so consistent iteration order matters.

## Optimistic locking — work first, detect conflict at write

No locks held during the think-time; writes carry proof the data didn't change underneath:

```sql
UPDATE products SET ..., version = version + 1
 WHERE id = $1 AND version = $2;   -- 0 rows updated = someone beat you → retry
```

Great when conflicts are *rare* (admin edits, user profiles, CRDT-ish flows) — no waiting, no deadlocks. Bad when conflicts are the norm: 10K buyers on one product = 9,999 retries per unit sold. ADR 001: "forces retry loops for a logically single decision."

## The atomic write — the rung below locking (what I actually use)

`UPDATE ... WHERE id = $1 AND inventory_available >= $1 RETURNING id` — the check rides inside the write, the row lock is held for one statement's duration (microseconds), and there's nothing to retry: no row back means a clean business "no" (sold out → 409), not a transient conflict. Full story in [[Database Concurrency and Atomicity]]. **Always ask first: can the check-then-act collapse into one conditional statement?** If yes, both locking disciplines above are overkill.

## Advisory locks — app-defined locks inside Postgres
`pg_advisory_lock(key)` / `pg_try_advisory_lock(key)`: locks on arbitrary numbers, no rows involved. Good for "only one instance runs this job" (e.g. ensuring a single outbox relay / cleanup worker across replicas) — you get mutual exclusion with the DB's crash-safety (connection dies → lock released) without inventing infrastructure.

## Distributed locks — the last resort
Redis `SET key val NX PX 30000` (+ Redlock across nodes). ADR 001 rejected it for inventory: **new point of failure** (holder crashes → wait out the TTL), **TTL races** (holder pauses > TTL, lock expires, two holders — fencing tokens are the fix), and the DB already serializes the row natively. Legitimate uses are coordination *outside* any single database: cron singleton across k8s pods, protecting a non-transactional external resource. If the contended state lives in Postgres, lock it in Postgres.

## Decision ladder (interview summary)
1. Single conditional statement? → **atomic write**, no explicit locking (my checkout)
2. Complex read-decide-write, moderate contention? → **`SELECT FOR UPDATE`** (+ NOWAIT/SKIP LOCKED as fits)
3. Conflicts rare, long think-time (human editing)? → **optimistic versioning**
4. Cross-process singleton, state in the DB anyway? → **advisory lock**
5. Coordination across systems with no shared DB? → **distributed lock**, with fencing tokens and humility

## Related
- [[Database Concurrency and Atomicity]]
- [[Transaction Isolation Levels]]
- [[Idempotency]] — retries need it regardless of the locking discipline
- [[Data Index]]
