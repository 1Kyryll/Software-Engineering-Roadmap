---
tags:
  - interview-qa
---

# Q: When should you use a composite index instead of two separate indexes?

*(Was open — answered fresh.)*

## Model answer

**Use a composite index when your queries filter/sort on those columns *together*; use separate indexes when they're queried independently.**

A composite index on `(a, b)` is a B-tree sorted by `a`, then `b` within equal `a` — like a phone book sorted by (last name, first name). Consequences:

1. **The leftmost-prefix rule.** `(a, b)` serves queries on `a` alone and on `a AND b` — but **not** on `b` alone (the phone book is useless for finding everyone named "John"). So `(a, b)` ≠ `(b, a)`, and if both columns are also queried solo, you may need `(a, b)` *plus* a separate index on `b`.

2. **Two separate indexes are a weak substitute for one composite.** Postgres *can* combine separate indexes on `a` and `b` with a **bitmap AND**, but that scans both indexes, builds bitmaps, and intersects them — consistently slower than one direct descent into a composite index that lands exactly on the matching range. And critically, **separate indexes cannot provide a combined sort order** — `ORDER BY a, b` needs the composite (or a sort step).

3. **Column order inside the composite: equality first, range last.** A range predicate "uses up" the index — columns after it can only filter, not seek. For `WHERE status = 'ready' AND visibility = 'public' ORDER BY published_at DESC`, the right index is `(status, visibility, published_at DESC)` — equality columns first pin the subtree, the range/sort column last lets rows come out pre-sorted, no sort node.

4. **Covering (index-only scans).** If the index contains every column the query touches, Postgres may skip the heap entirely (`INCLUDE (...)` for extra payload columns).

## Evidence in my own schemas
- **OneTube feed**: `videos (status, visibility, published_at DESC)` — precisely rule 3: two equalities, then the sort column. One index serves the entire feed query with no sort.
- **Cursor pagination**: the composite `(created_at, id)` index is what makes `WHERE (created_at, id) < ($1, $2)` a seek — see [[QA - Paginating 50 Million Rows]].
- **Partial composites**: `reservations (expires_at) WHERE status = 'active'` ([ADR 009](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/009-inventory-count.md)) — when one predicate value is constant in the query, moving it into a partial-index `WHERE` beats putting it in the key: smaller index, same seek.
- **Lychee-Chat (MongoDB)**: compound unique index on `Contact (owner, user)` — composite thinking isn't SQL-specific; it also enforces the uniqueness constraint.

## Say this in the interview

"A composite index is the right choice when queries filter or sort on those columns *together*; separate indexes are for genuinely independent query patterns. The mental model: an index on `(a, b)` is a B-tree sorted by `a` first, then by `b` within equal values of `a` — like a phone book sorted by last name, then first name.

That model gives you the **leftmost-prefix rule** immediately: the phone book is great for 'find Smith' and 'find Smith, John', but useless for 'find everyone named John.' So `(a, b)` serves queries on `a` alone and on `a` and `b` together — but not `b` alone, and `(a, b)` is not the same index as `(b, a)`.

Why not just create two single-column indexes and let the database combine them? Postgres *can* — it's called a bitmap AND: scan both indexes, build a bitmap of matching rows from each, intersect them. But that's consistently slower than one descent into a composite index that lands directly on the matching range. And the bigger limitation: two separate indexes can never produce a combined **sort order** — `ORDER BY a, b` needs the composite, otherwise you pay for an explicit sort.

Column order inside the composite follows one rule: **equality predicates first, the range or sort column last** — because a range 'uses up' the index; anything after it can only filter, not seek. My concrete example: my video platform's feed queries `WHERE status = 'ready' AND visibility = 'public' ORDER BY published_at DESC`, and the index is exactly `(status, visibility, published_at DESC)` — the two equalities pin the subtree, and rows come out already sorted, no sort step at all.

Two refinements worth adding: if one predicate is a *constant* in every query — like `status = 'active'` — a **partial index** with that in the WHERE clause beats putting it in the key: same seek, much smaller index; I use that on my reservations table. And if the index covers every column the query touches, Postgres can do an **index-only scan** and skip the heap entirely. So the summary: index for the query shape, not the columns — equality first, range last, and remember the leftmost prefix rule."

## Related
- [[Pagination - Cursor vs Offset]]
- [[Denormalization and Invariants]] — partial indexes
- [[QA - Indexing Random UUIDs]]
- [[Interview QA Index]]
