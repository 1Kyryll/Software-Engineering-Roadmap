---
tags: [system-design, database, concurrency, postgres]
---

# Transaction Isolation Levels

Isolation levels define **which concurrency anomalies a transaction may observe**. Stronger isolation = fewer anomalies = more blocking/retries. The skill is knowing the anomalies, what each level rules out (in Postgres specifically — it's stricter than the SQL standard), and *when you actually need to escalate*.

## The anomalies

| Anomaly | What happens |
|---|---|
| **Dirty read** | read another transaction's *uncommitted* data |
| **Non-repeatable read** | re-read the same row within one txn → different value (someone committed in between) |
| **Phantom read** | re-run the same *query* → different row set |
| **Lost update** | two read-modify-write txns both base their write on the same original value; one update vanishes |
| **Write skew** | two txns each read a condition, both act, and their *combined* writes violate the condition neither violated alone (classic: two on-call doctors both sign off simultaneously) |

## The levels in Postgres

| Level | Dirty | Non-repeatable | Phantom | Write skew |
|---|---|---|---|---|
| Read Uncommitted | impossible* | possible | possible | possible |
| **Read Committed** (default) | impossible | possible | possible | possible |
| **Repeatable Read** | impossible | impossible | impossible* | possible |
| **Serializable** | impossible | impossible | impossible | impossible |

\* Postgres quirks worth stating: Read Uncommitted *is* Read Committed (dirty reads never happen in Postgres, thanks MVCC); and Postgres's Repeatable Read is **snapshot isolation** — it also prevents phantoms, going beyond what the SQL standard requires of the level.

- **Read Committed**: each *statement* sees a fresh snapshot of committed data. Between statements, the world may change.
- **Repeatable Read**: the whole transaction runs against one snapshot taken at its first query. Concurrent commits are invisible. If you try to *update* a row someone else changed since your snapshot → `serialization failure`, retry.
- **Serializable**: snapshot isolation + SSI (predicate tracking) — the outcome is guaranteed equivalent to *some* serial order, catching write skew. Cost: false-positive serialization failures → **every serializable transaction needs a retry loop**.

## Why my E-Commerce system runs at Read Committed — and that's the point

The oversell problem *looks* like it demands Serializable: read availability, check, decrement — classic read-check-write, vulnerable to write skew at lower levels. My design sidesteps the whole question ([ADR 001](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/001-inventory-atomic-decrement.md), [[Database Concurrency and Atomicity]]):

```sql
UPDATE products SET inventory_available = inventory_available - $1
 WHERE id = $2 AND inventory_available >= $1
RETURNING id;
```

The read (`WHERE`), the check (`>= $1`), and the write are **one statement** — atomic at any isolation level. Under Read Committed, concurrent UPDATEs on the same row queue on the row lock, and crucially each **re-evaluates its WHERE clause against the newest committed version** after the lock clears (the "EvalPlanQual recheck") — so the condition can't act on stale data. Ten thousand concurrent decrements → exactly N successes.

Interview framing: *"I'd rather restructure a read-check-write into an atomic conditional write than escalate isolation — Serializable makes every hot-row transaction a retry loop under contention, which is exactly the flash-sale scenario where retries hurt most."* Same logic in the CTEs that pair the decrement with reservation insert/release — multi-table invariants held by single-statement atomicity, not isolation ([ADR 009](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/009-inventory-count.md)).

## When you genuinely escalate
- **Repeatable Read**: multi-statement *reads* needing one consistent snapshot — reports, backups (`pg_dump`), the nightly inventory reconciliation query ([[Denormalization and Invariants]]) if split across statements.
- **Serializable**: invariants spanning **rows you read but don't write** (write skew) that can't be captured in one statement or a constraint — booking overlaps, sum-across-accounts limits. Budget for retries.
- Middle grounds first: unique constraints catch many phantoms; `SELECT ... FOR UPDATE` pins the specific read-then-write rows ([[Locking]]) without paying for full serializability.

## Related
- [[Database Concurrency and Atomicity]]
- [[Locking]]
- [[Postgres in Production]]
- [[Data Index]]
