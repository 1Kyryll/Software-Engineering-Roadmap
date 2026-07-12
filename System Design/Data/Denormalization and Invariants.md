---
tags: [system-design, database, data-modeling]
---

# Denormalization and Invariants

Denormalization = storing a value that *could* be computed from other data, because computing it on every read is too expensive. The price: the stored copy can drift from the truth. The discipline: **every denormalized value needs (a) transactional maintenance and (b) a way to detect drift.**

## Case study 1 — `inventory_available` (E-Commerce, [ADR 009](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/009-inventory-count.md))

Conceptually, availability is derived:

```
available(product) = inventory_total − Σ active_reservations.quantity
```

Computing that on every product-page render means scanning the reservations table per read, and making the decrement atomic against an aggregate requires serializable isolation. Instead, `inventory_available` is **materialized** as a column, updated in the same transaction/statement as every reservation state change (see [[Database Concurrency and Atomicity]] — the CTEs keep column and reservations in lockstep).

Justified by the access pattern: read on every page render, written only on reservation transitions.

**The drift detector** — a reconciliation query that must always return zero rows:

```sql
SELECT p.id, p.inventory_available AS materialized,
       p.inventory_total - COALESCE(r.reserved, 0) AS computed
  FROM products p
  LEFT JOIN (SELECT product_id, SUM(quantity) AS reserved
               FROM reservations WHERE status = 'active'
              GROUP BY product_id) r ON r.product_id = p.id
 WHERE p.inventory_available <> p.inventory_total - COALESCE(r.reserved, 0);
```

A non-zero result is **an alert about a code bug, not a number to silently fix** — "a system where this drifts is a system that oversells." It's exported as a Prometheus gauge with an alert on non-zero. This "invariant as a monitored metric" idea is one of my favorite interview talking points.

## Case study 2 — `view_count` (OneTube)

`videos.view_count` is denormalized onto the video row and incremented on view — the alternative (a `views` event table + `COUNT(*)`) is exact and auditable but makes every feed query aggregate millions of rows. For a view counter, approximate-but-cheap wins. ([OneTube data model](https://github.com/1Kyryll/OneTube#data-model))

## Supporting technique — partial indexes

Not denormalization, but the same "shape storage to the access pattern" thinking ([ADR 009](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/009-inventory-count.md)):

```sql
CREATE INDEX idx_reservations_active_expiring
    ON reservations (expires_at) WHERE status = 'active';
```

The all-time reservations table is mostly `consumed`/`released` history; only `active` rows need fast lookup. The `WHERE` clause keeps the index tiny and hot. OneTube's feed index is composite for the same reason: `videos (status, visibility, published_at DESC)` matches the exact feed predicate.

## When to denormalize (checklist)
1. The derived value is read far more often than its inputs change ✅ both cases
2. You can maintain it **in the same transaction** as the source change ✅ CTEs
3. You have a reconciliation/repair story ✅ nightly invariant check
4. Approximate is acceptable, or exactness is guaranteed by (2) — know which one you're claiming

## Related
- [[Database Concurrency and Atomicity]]
- [[Caching Strategies]] — caching is denormalization with a TTL
- [[Data Index]]
