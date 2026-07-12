---
tags: [system-design, communication, graphql]
---

# GraphQL

A query language where the **client declares the shape of the data it wants** and the server resolves it field by field. One endpoint, typed schema, no over/under-fetching.

## Where I built it

The [Restaurant System gateway](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/internal/services/gateway) is a full GraphQL API built with **gqlgen** (schema-first Go codegen, config in [`gqlgen.yaml`](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/gqlgen.yaml)):

- **Queries** — `order(id)`, `orders(first, status)`, `menuItems(first, category)`, `search(query)` ([query.resolvers.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/query.resolvers.go))
- **Mutations** — `register`, `login`, `createOrder`, `updateOrderStatus`, `cancelOrder` ([mutation.resolvers.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/mutation.resolvers.go))
- **Subscriptions** — `orderCreated`, `orderStatusChanged(orderId)` over `graphql-transport-ws` ([subscription.resolvers.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/services/gateway/graph/subscription.resolvers.go)) — see [[Real-Time Communication]]
- Resolvers are **role-scoped**: customers can only query their own orders; kitchen staff sees all (see [[Authentication and Authorization]]).

## The N+1 problem and DataLoaders (the killer interview topic)

Naive resolver design: query 50 orders, then for each order resolve `order.user` → 1 + 50 database/gRPC calls. GraphQL makes this failure mode *structural* because nested resolvers run per parent object.

**Fix: DataLoader** — batch all the individual `load(id)` calls that happen within one request tick into a single `WHERE id = ANY($1)` query, and cache per-request.

My implementation: [`gateway/dataloaders/`](https://github.com/1Kyryll/Restaurant-System-Microservices/tree/main/server/internal/services/gateway/dataloaders) — loaders for `UserByID`, `MenuItemByID`, `TicketByOrderID`, `OrderItemsByOrderID`. Two details that matter:

```go
// dataloaders.go — loaders are created PER REQUEST and injected via context
func Middleware(queries *sqlc.Queries, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        loaders := NewLoaders(queries)   // fresh per request
        ctx := context.WithValue(r.Context(), loadersKey, loaders)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

- **Per-request, not global**: a global loader's cache would leak data across users and go stale. Per-request scope makes the cache trivially correct.
- The batch functions issue one `GetOrderItemsByOrderIDs(ctx, []int32{...})` style query for the whole batch (same pattern as OneTube's "JOIN to avoid N+1" feed rule).

## Tradeoffs vs REST
- **For**: client-shaped responses (mobile vs desktop ask for different fields), one round trip for nested data, schema introspection = living docs, subscriptions built into the spec.
- **Against**: caching is harder (every query potentially unique — no URL-level HTTP caching), query cost is unbounded without depth/complexity limits, N+1 unless you engineer around it, file uploads are awkward.
- **My rule of thumb**: GraphQL shines at the edge when clients need flexible reads over aggregated services (exactly the Restaurant gateway's job); REST is simpler for narrow, well-known contracts; gRPC for internal typed calls.

## Related
- [[API Gateway Pattern]]
- [[REST API Design]]
- [[Real-Time Communication]]
- [[Communication Index]]
