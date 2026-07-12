---
tags:
  - interview-qa
---

# Q: How do you paginate through 50 million rows without using OFFSET?

## My original answer
> Such a massive dataset requires more than Offset Pagination. The solution is Cursor Pagination because the database can directly jump to the desired window of rows. With Offset, getting 20 rows at the 1-millionth offset means scanning all rows before it — O(n). With Cursor we store a cursor (usually PK + timestamp); the database compares whether we want rows bigger or smaller than the cursor — O(log n), definitely better.

## Review — right answer, tighten the mechanics ✅

The core is correct (keyset/cursor pagination, and *why* OFFSET degrades). Refinements that turn it from good to airtight:

1. **Precise complexity**: OFFSET is O(offset + limit) — the DB walks the index/heap and *discards* `offset` rows every page; page 1 is fast, page 50,000 is a disaster, and total cost over a full scan of the table is quadratic. Cursor is **O(log n + limit)** per page: one B-tree *seek* to the cursor position, then read `limit` rows along the leaf chain. "Jump directly" happens *because of the index seek* — say that; the cursor without a matching index is still a scan.
2. **The cursor needs a tiebreaker and a matching index.** A timestamp alone isn't unique — two rows in the same millisecond cause skips/duplicates at page boundaries. Use `(created_at, id)` with row-value comparison and a composite index in the same order:
   ```sql
   WHERE (created_at, id) < ($1, $2)
   ORDER BY created_at DESC, id DESC
   LIMIT 20;
   ```
   This is exactly my catalog query ([products.sql](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/queries/catalog/products.sql)) and OneTube's feed rule ("cursor on `(published_at, id)`, never OFFSET").
3. **Two more selling points worth adding**: cursor pagination is **stable under concurrent writes** (OFFSET shifts when rows are inserted/deleted mid-scroll — users see duplicates/gaps), and the cursor should be **opaque** (base64 of `{created_at, id}` — my [cursor.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/catalog/domain/cursor.go)) so clients can't tamper or couple to its structure.
4. **Name the tradeoff before they ask**: no random access ("jump to page 300" is gone) — fine for feeds/infinite scroll, wrong for admin tables that genuinely need numbered pages.

## Say this in the interview

"I'd use keyset pagination — usually called cursor pagination. The problem with OFFSET is that the database can't skip rows it hasn't counted: `OFFSET 1000000 LIMIT 20` walks the index, fetches and *discards* a million rows, then returns twenty. That's O(offset + limit) per page — page one is fast, page fifty-thousand is a disaster — and it has a correctness problem too: if rows are inserted or deleted while a user scrolls, every subsequent page shifts, so they see duplicates or miss rows entirely.

With a cursor, each response includes a token encoding the position of the last row returned — concretely the pair `(created_at, id)`. The next request runs `WHERE (created_at, id) < cursor ORDER BY created_at DESC, id DESC LIMIT 20`, backed by a composite index on exactly those columns in that order. Now the database does one B-tree *seek* to the cursor position — that's the O(log n) part — and reads twenty rows along the leaf chain. Cost is constant no matter how deep you are in the 50 million rows, and it's stable under concurrent writes because the cursor is anchored to a row's values, not to a count.

Three details make it production-grade. The `id` in the cursor is a **tiebreaker**: timestamps aren't unique, and without it two rows in the same millisecond can be skipped or duplicated at a page boundary — the pair makes the ordering total. The composite **index must match** the cursor columns and direction, otherwise the 'seek' silently degrades to a scan. And the cursor should be **opaque** — I base64-encode the JSON — so clients can't tamper with it or couple to its internals.

The tradeoff to volunteer: you lose random access — there's no 'jump to page 300.' For a 50-million-row feed or infinite scroll that's the right trade; for a small admin table where numbered pages genuinely matter, OFFSET is fine. I use exactly this pattern in my e-commerce catalog and my video platform's feed — cursor on `(created_at, id)`, never OFFSET."

## Related
- [[Pagination - Cursor vs Offset]]
- [[QA - Composite vs Separate Indexes]] — why the index must match the cursor columns
- [[Interview QA Index]]
