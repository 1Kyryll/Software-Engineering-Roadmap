---
tags:
  - interview-qa
---

# Q: What happens when you put an index on a random UUID column?

## My original answer
> Putting an index on a random UUID is not the best idea. Indexes are meant to optimize reading, usually implemented with B-Tree. But UUIDs are random, the binary search has no condition to compare these PKs, so reads that should have been O(log n) become O(n), plus writing and creating an index occupies space. Therefore it makes no sense to create an index on a random UUID. The optimal solution is time-based/sorted UUIDs, utilizing advantages of both technologies.

## Review — right conclusion, wrong mechanism ⚠️

The recommendation (time-ordered UUIDs) is correct, but the central claim is **not**:

1. **UUIDs are perfectly comparable, and lookups stay O(log n).** A UUID is a 128-bit value; the B-tree compares bytes and doesn't care that they're random. `WHERE id = '...'` on an indexed random UUID column is a normal O(log n) index lookup and works great. If this were O(n), half the production databases in the world (which index UUIDv4 PKs) would be broken. An interviewer will catch this immediately.
2. **The real problem is *write-side insert locality*.** With sequential keys (bigserial, UUIDv7), every insert lands on the rightmost B-tree leaf — one hot page, great cache behavior. With random UUIDs, each insert lands on a *random* leaf: constant **page splits**, index **fragmentation/bloat**, the whole index must stay in the buffer pool (random pages are touched all the time), and **WAL amplification** (more dirtied pages per insert). Invisible on small tables, measurable at scale — this is the actual argument in my own [ADR 007](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/007-primary-keys.md).
3. **Secondary read-side cost**: range scans "give me recent rows" can't use a random-UUID index at all (insertion order isn't encoded), and the 16-byte keys make the index larger than a bigint's — fewer entries per page, deeper tree, more cache pressure. Point lookups still fine; it's the locality-dependent operations that suffer.

## Say this in the interview

"First, the index still *works*. A UUID is just a 128-bit value — the B-tree compares bytes and doesn't care that they're random — so point lookups like `WHERE id = ...` are a perfectly normal O(log n) index descent. Half the databases in production index UUIDv4 primary keys and they're fine functionally.

What suffers is **locality**, mostly on the write path. With a sequential key — a bigserial, or a time-ordered UUID — every new insert lands on the rightmost leaf page of the tree: one hot page, few page splits, a tiny working set that lives happily in cache. With random UUIDs, every insert lands on a *random* leaf page. That means constant page splits and index fragmentation, the *entire* index effectively becomes the working set because any page can be touched at any moment — so it all has to fit in the buffer pool — and you generate more WAL because more pages get dirtied per insert. On a small table this is invisible; on an insert-heavy table at the hundreds-of-millions-of-rows scale it shows up as degraded write throughput and cache pressure. There's a read-side cost too: insertion order isn't encoded anywhere, so 'give me recent rows' can't use this index at all, and 16-byte keys make the tree larger and deeper than an 8-byte integer's.

The fix is to keep UUIDs but make them time-ordered: **UUIDv7** — standardized in RFC 9562 — puts a 48-bit millisecond timestamp in the high bits, so newly generated values sort roughly sequentially and B-tree locality is restored, while you keep everything that made UUIDs attractive: no coordination between writers, and IDs that don't leak business volume the way `/orders/47832` does. That's exactly the decision in my e-commerce system: UUIDv7 primary keys, stored as the native 16-byte `uuid` type — never as text, which more than doubles the storage and slows every comparison."

## Related
- [[Primary Keys - UUIDv7]]
- [[Postgres in Production]]
- [[QA - Composite vs Separate Indexes]]
- [[Interview QA Index]]
