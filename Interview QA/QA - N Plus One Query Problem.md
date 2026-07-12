---
tags:
  - interview-qa
---

# Q: What is the N+1 query problem and how do you fix it?

*(Was open — answered fresh.)*

## Model answer

**The problem**: you fetch a list of N parent rows with one query, then loop over them fetching each one's related data with one query per row — `1 + N` round trips where one or two would do.

```
SELECT * FROM orders LIMIT 50;                 -- 1 query
-- then, in a loop:
SELECT * FROM users WHERE id = $order.user_id; -- × 50
```

Each query is fast, so nothing looks wrong locally — the cost is **per-query overhead × N**: network round trips, planning, connection pool churn. 50 orders = 51 queries; nest one level deeper (items per order, product per item) and it multiplies. It's the classic death-by-a-thousand-cuts: pages get slower as data grows, and no single query shows up as slow in monitoring.

**Where it comes from**: any abstraction that makes per-row fetching look free — ORM lazy loading (`order.user` triggers a query on property access), or GraphQL resolvers, where nested resolvers run *per parent object by design* (see [[GraphQL]]).

**The fixes, in order of preference:**

1. **JOIN — one query.** When the shape is "parents with their related row," just join. OneTube's feed rule says it verbatim: *"JOIN to avoid N+1 — the feed query joins `videos` with `users` for the uploader name"* ([design rules](https://github.com/1Kyryll/OneTube#design-rules)).

2. **Batch the second query** — two queries total: collect the IDs, then `WHERE id = ANY($1)`, stitch in memory. Right when a JOIN would duplicate wide parent rows (one parent × many children) or when the "second query" crosses a service boundary and can't be joined.

3. **DataLoader** — batching automated for resolver-style code. Individual `load(id)` calls made during one request tick are coalesced into a single batched query, cached per request. This is my [Restaurant System's dataloaders](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/internal/services/gateway/dataloaders) — per-request loaders for `UserByID`, `MenuItemByID`, `OrderItemsByOrderID`, each backed by one `... = ANY($1)` sqlc query. Per-request scope matters: a global loader's cache would leak data across users and go stale.

4. **ORM eager loading** — the framework-level version of 1/2: declare relations up front (`includes`/`joinedload`/`preload`) instead of lazily faulting them in.

**How you catch it**: query logging or APM showing "one endpoint = 51 near-identical queries" (my kitchen service's enriched order lookup does the two-batched-queries version by hand: `GetOrderItemsByOrderIDs` then `GetMenuItemsByIDs` — [kitchen.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/kitchen/services/kitchen.go)).

## Say this in the interview

"N+1 is when you fetch a list of N rows with one query, then loop over the results issuing one more query per row for related data — fifty orders, then fifty individual 'get the user for this order' lookups: fifty-one queries where one or two would do. What makes it insidious is that every individual query is fast, so nothing shows up as slow in monitoring — the cost is per-query *overhead* multiplied by N: network round trips, planning, connection pool churn. It scales with your data, so the page that was fine at launch degrades quietly. And it usually comes from abstractions that make per-row fetching look free: ORM lazy loading, where accessing `order.user` silently fires a query, or GraphQL, where nested resolvers run once per parent object *by design*.

The fixes, in the order I reach for them. First, a **JOIN** — if the shape is 'parents with a related row each,' one query does it; my video platform's feed joins videos with users for the uploader name, one round trip. Second, **batching** — two queries total: fetch the parents, collect their IDs, then `WHERE id = ANY(that array)` and stitch the results in memory. That's better than a JOIN when joining would duplicate wide parent rows across many children, or when the second fetch crosses a service boundary and can't be joined at all. Third, for resolver-style code where you can't restructure the call pattern, a **DataLoader**: it transparently collects all the individual `load(id)` calls that happen during one request tick, coalesces them into a single batched query, and caches per request. I built exactly this for my restaurant system's GraphQL gateway — per-request loaders for users, menu items, and order items, each backed by one `= ANY` query. The per-request scope matters: a global loader's cache would leak data between users and go stale. And fourth, ORM **eager loading** — declaring the relations up front so the framework fetches them in bulk instead of lazily.

How I'd catch it in the first place: query logging or tracing that shows one endpoint issuing dozens of near-identical queries. The rule that prevents it: never issue a query inside a loop over query results — join it, batch it, or put a DataLoader in front of it."

## Related
- [[GraphQL]] — why N+1 is structural there, and the DataLoader implementation
- [[Postgres in Production]] — sqlc makes the batched queries typed
- [[Interview QA Index]]
