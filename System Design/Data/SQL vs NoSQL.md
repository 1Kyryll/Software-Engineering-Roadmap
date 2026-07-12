---
tags: [system-design, database]
---

# SQL vs NoSQL

I run Postgres in four projects and MongoDB in one — the split wasn't accidental, and being able to defend both choices is the point.

## Where each lives

| Project | Store | Why it fits |
|---|---|---|
| E-Commerce, OneTube, Restaurant, Finance-Tracker | **Postgres** | Transactions and constraints carry the correctness story: atomic inventory decrement, unique idempotency keys, FKs between orders/items/reservations, outbox in the same transaction |
| [Lychee-Chat](https://github.com/1Kyryll/Lychee-Chat) | **MongoDB** (Mongoose) | Chat messages are self-contained documents; the access pattern is "append message, read recent messages by chat" — no cross-entity transactions needed |

## What actually decided it

**E-Commerce cannot be eventually consistent about money and stock.** The invariants (never oversell, never double-charge) are enforced *by the database*: single-statement atomic CTEs, unique constraints on idempotency keys, FK integrity, all-or-nothing multi-table transactions (order + items + outbox). Relational databases make these guarantees native. Re-implementing them over a document store means distributed-systems homework on every write path. See [[Database Concurrency and Atomicity]].

**Chat messages fit documents naturally.** A message (with embedded media array), a chat, a contact — each is read/written as a unit. Mongoose models in [packages/db/models/](https://github.com/1Kyryll/Lychee-Chat/tree/main/packages/db/models): `User`, `Chat`, `Message`, `Contact`. Even so, indexes still matter in NoSQL:
- `Contact` has a **compound unique index on `(owner, user)`** — the same "uniqueness enforced by the store, not the app" discipline as SQL
- `Message` is indexed on `chat` — the hot query path (load a conversation)

The BullMQ media worker also shows a document-model idiom: updating one element of an embedded array in place with a positional `$set` on `media[index]` — see [[Message Queues and Background Workers]].

## Interview framing (avoid the false dichotomy)
- The real questions: **What are the invariants? What are the access patterns? What consistency do reads need?**
- Relational wins when correctness lives in relationships and multi-row transactions (financial anything, inventory, bookings).
- Document stores win when entities are naturally aggregate-shaped, schema evolves fast, and horizontal write-scaling matters more than cross-document transactions.
- Postgres blurs the line: `jsonb` gives document flexibility inside a transactional store (my outbox `payload` column is exactly that).
- "NoSQL = no schema" is a myth — the schema moves into application code (Mongoose schemas are schemas!), it doesn't disappear.

## Related
- [[Database Concurrency and Atomicity]]
- [[Postgres in Production]]
- [[Data Index]]
