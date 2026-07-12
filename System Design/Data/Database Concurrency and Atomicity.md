---
tags: [system-design, database, concurrency]
---

# Database Concurrency and Atomicity

The flagship problem of my [E-Commerce-System](https://github.com/1Kyryll/E-Commerce-System): **10,000 users buying a product with 10 units in stock — no overselling, no lost payments, no dropped orders.** The whole design hangs on three patterns; this note covers the first two (the third is [[Transactional Outbox]]).

Authoritative sources: [ADR 001](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/001-inventory-atomic-decrement.md), [ADR 002](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/002-reservation-table-payment-window.md), [ADR 009](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/009-inventory-count.md), [system-design.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/system-design.md).

## Pattern 1 — Atomic conditional decrement

The oversell check and the decrement must be **one atomic operation**, or two concurrent buyers both pass the check. One SQL statement does it:

```sql
UPDATE products
   SET inventory_available = inventory_available - $1
 WHERE id = $2 AND inventory_available >= $1
RETURNING id;
```

Row returned → you got the stock. No row → sold out → `409 Conflict`. Postgres MVCC serializes concurrent updates on the row; 10K concurrent attempts resolve with *exactly* N successes where N = remaining stock.

The production version goes further — decrement **and** reservation insert in a single statement via CTE, so neither can exist without the other ([queries/order/reservations.sql](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/queries/order/reservations.sql)):

```sql
WITH decremented AS (
    UPDATE products
       SET inventory_available = inventory_available - $1
     WHERE id = $2 AND inventory_available >= $1
    RETURNING id
)
INSERT INTO reservations (id, idempotency_key, product_id, user_id, quantity, status, expires_at)
SELECT $3, $4, $2, $5, $1, 'active', now() + INTERVAL '15 minutes'
  FROM decremented
RETURNING ...;
```

Empty CTE → zero rows inserted → the app translates to "insufficient inventory."

### Alternatives and why they lose (straight from ADR 001)
- **`SELECT ... FOR UPDATE` (pessimistic locking)** — correct but serializes all buyers behind row locks: high contention, waiting customers. Right for banking, wrong for e-commerce.
- **Optimistic locking (version column + retry)** — forces retry loops for what is logically a single decision.
- **Redis distributed lock (SETNX)** — new failure point (lock holder crashes, TTL races); the DB already solves this natively.
- For true flash-sale scale: a Redis token-queue in front — know it exists, don't build it prematurely.

## Pattern 2 — Time-bound reservations for the payment window

Between "clicked buy" and "payment settled" the stock must be held without blocking anyone else. Reservation rows with a 15-minute TTL and an explicit lifecycle: `active → consumed` (payment ok) or `active → released` (declined/expired, inventory restored **in the same transaction**):

```sql
WITH released AS (
    UPDATE reservations SET status = 'released', released_at = now()
     WHERE id = $1 AND status = 'active'
    RETURNING product_id, quantity
)
UPDATE products p
   SET inventory_available = p.inventory_available + r.quantity
  FROM released r WHERE p.id = r.product_id;
```

A [cleanup worker](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/cleanup.go) releases expired actives in batches; per-row failures are logged but don't abort the batch. Partial indexes keep the "find expired actives" scan cheap:

```sql
CREATE INDEX idx_reservations_active_expiring
    ON reservations (expires_at) WHERE status = 'active';
```

**Payment timeout is the hard case**: if the network dies mid-charge, you don't know whether money moved — you must reconcile against the provider's status API (or wait for a webhook) before releasing. Never assume "timeout = failed."

The full purchase flow (two anchoring transactions, everything between them retryable) is diagrammed in [system-design.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/system-design.md); the orchestration lives in [place_order.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/place_order.go) — reserve each line item, charge once for the total, finalize order + outbox in one transaction, releasing everything on any failure.

## Interview sound bites
- "Check-then-act is a race; make the check part of the write." (`WHERE inventory_available >= $1`)
- "A single SQL statement is atomic — no explicit transaction needed for the decrement+insert CTE."
- "Reservations turn 'hold stock during payment' from a locking problem into a data-modeling problem."
- Isolation levels: this design works at Read Committed *because* atomicity is pushed into single statements — no need for Serializable and its retry storms.

## Related
- [[Idempotency]] — the retry side of the same flow
- [[Denormalization and Invariants]] — why `inventory_available` is materialized
- [[Transactional Outbox]]
- [[Data Index]]
