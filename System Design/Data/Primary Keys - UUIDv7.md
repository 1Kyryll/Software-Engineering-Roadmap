---
tags: [system-design, database, data-modeling]
---

# Primary Keys — UUIDv7

Full reasoning: [ADR 007](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/007-primary-keys.md) in my E-Commerce repo. Every table there uses **UUIDv7 stored as native Postgres `uuid`** (16 bytes binary). On the Go side: `github.com/google/uuid` → `uuid.NewV7()`.

## Why UUIDv7 specifically

UUIDv7 (RFC 9562, 2024) = 48 high-order bits of Unix-millisecond timestamp + randomness. Newly generated values are **roughly sequential**, so inserts land at the right edge of the B-tree index — preserving index locality. That's the fix for UUIDv4's notorious insert-heavy-table problem: v4 is uniformly random, inserts hit random B-tree pages, fragmenting the index and bloating cache. Invisible on small tables, measurable at 100M rows.

## The decision matrix (interview gold)

| Option | Pros | Why not |
|---|---|---|
| `bigserial` | 8 bytes, fastest index | **Leaks business info** (`/orders/47832` → competitor knows order volume); central coordination breaks with multiple writers |
| UUIDv4 | uncoordinated, unguessable | Random inserts fragment the B-tree |
| **UUIDv7** ✅ | k-sortable + unguessable + uncoordinated | 16 bytes vs 8 — acceptable |
| ULID | functionally identical to v7 | Lost the standardization race; v7 has broader support |
| Snowflake IDs | 8 bytes, sortable | Needs an ID-issuing coordination service — overkill for single-region |

## Storage rule
Native `uuid` type, **never** `text`/`varchar(36)` — string form is 37 bytes (2.3× the storage), and comparisons are much slower. `pgx` + `google/uuid` handle the binary type transparently.

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY,   -- generated app-side with uuid.NewV7()
    ...
);
```

## Refinement worth mentioning
UUIDs are ugly in URLs. Pattern: keep the UUID as the real PK, expose a human-friendly `slug` or short opaque ID (`/products/p_8sNqL2v`) as a presentational layer. Addable later without migration.

(Contrast: my [Restaurant System](https://github.com/1Kyryll/Restaurant-System-Microservices) uses plain `int32` serial IDs — fine for a demo, and a good example of knowing *when* the tradeoff matters: no public enumeration surface + single writer = serial is acceptable.)

## Related
- [[Postgres in Production]]
- [[Storing Money in Databases]] — the other one-way-door data decision
- [[Data Index]]
