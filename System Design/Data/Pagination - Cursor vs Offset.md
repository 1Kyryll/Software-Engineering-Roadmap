---
tags: [system-design, database, api]
---

# Pagination — Cursor vs Offset

## Offset pagination (`LIMIT 20 OFFSET 40`)
Simple, supports "jump to page 7", but has two production problems:
1. **Skew under writes**: rows inserted/deleted between page fetches shift everything — users see duplicates or miss rows while infinite-scrolling.
2. **Performance**: `OFFSET 100000` makes Postgres *fetch and discard* 100k rows before returning 20. Cost grows linearly with depth.

## Cursor (keyset) pagination
The client passes an opaque cursor encoding the **last row it saw**; the query seeks past it with an index:

```sql
-- E-Commerce catalog (queries/catalog/products.sql)
SELECT ...
  FROM products
 WHERE (created_at, id) < ($1::timestamptz, $2::uuid)
 ORDER BY created_at DESC, id DESC
 LIMIT $3;
```

- **`(created_at, id)` composite cursor** — the `id` tiebreaker makes ordering total; two rows created in the same millisecond can't cause skipped/duplicated entries. Postgres's row-value comparison `(a, b) < (x, y)` uses the matching composite index.
- Stable under concurrent inserts, and O(log n) regardless of depth.
- Tradeoff: no random page access, and the cursor must round-trip client-side.

## Where I built it

- **E-Commerce catalog** — [ADR 006](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/006-catalog-reads-cache-aside-cursor-pagination.md) (decision + tradeoffs), query above in [products.sql](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/queries/catalog/products.sql), cursor codec in [catalog/domain/cursor.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/catalog/domain/cursor.go):

```go
type Cursor struct {
    CreatedAt time.Time `json:"created_at"`
    ID        uuid.UUID `json:"id"`
}
// Encode: JSON → base64url. Opaque to clients — they can't
// construct or tamper with meaningful cursors, and the encoding
// can change without breaking the API contract.
```

  First page uses a sentinel `(now() + 1 day, uuid_nil)` so all rows match — avoids a special-cased first-page query.

- **OneTube feed** — design rule: "Cursor pagination on `(published_at, id)` or `(created_at, id)`. **Never OFFSET.**" ([README](https://github.com/1Kyryll/OneTube#design-rules)), feed endpoint `GET /videos/feed`, backed by the composite index `videos (status, visibility, published_at DESC)`.

- **Restaurant GraphQL** — `orders(first: Int)` / `menuItems(first: Int)` follow the GraphQL connection-style `first` argument.

## Interview framing
- Default to cursor pagination for any user-facing infinite scroll or feed.
- Offset is acceptable for small, admin-style, rarely-mutated tables where "page 7 of 12" UX genuinely matters.
- Cursor must be over a **unique total ordering** — always add the PK as tiebreaker.
- Make cursors **opaque** (base64) so clients don't couple to their internals.

## Related
- [[REST API Design]]
- [[Caching Strategies]] — ADR 006 pairs pagination with cache-aside
- [[Data Index]]
