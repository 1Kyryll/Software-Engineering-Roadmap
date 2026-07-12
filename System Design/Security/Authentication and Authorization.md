---
tags: [system-design, security, auth]
---

# Authentication and Authorization

AuthN = who are you. AuthZ = what may you do. I've implemented the full stack in every project; the pattern is consistent enough to state as *my* standard.

## My standard pattern (all five projects)

**Stateless JWT sessions in HttpOnly cookies.**

1. Login verifies the password hash, issues an HS256-signed JWT with the user ID in `sub` (+ role claims where RBAC exists)
2. The token is set as a cookie: `HttpOnly; Secure; SameSite=Lax`
3. Middleware verifies signature + expiry on every request and injects the user ID/claims into the request context

E-Commerce implementation ([gateway/auth/token.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/gateway/auth/token.go)):

```go
func VerifyToken(secret []byte, raw string) (uuid.UUID, error) {
    var claims Claims
    _, err := jwt.ParseWithClaims(raw, &claims, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, ErrInvalidToken   // reject alg-substitution attacks
        }
        return secret, nil
    })
    ...
}
```

That signing-method check matters: without it, a token crafted with `alg: none` or an RS256/HS256 confusion can bypass verification — a classic JWT vulnerability, handled explicitly.

### Why cookies, not localStorage (say this verbatim)
`HttpOnly` means JavaScript cannot read the token → XSS can't exfiltrate it. `SameSite=Lax` blocks most CSRF vectors. localStorage is readable by any injected script. OneTube's rule: "**JWT never in localStorage. Always HttpOnly cookie.**"

## Password hashing
- **bcrypt** — E-Commerce ([gateway/auth/hash.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/gateway/auth/hash.go)), Restaurant, Lychee-Chat (bcryptjs)
- **argon2id** — OneTube ([auth/password.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/auth/password.go)) — the current OWASP first choice (memory-hard → GPU-resistant)
- Both are deliberately slow, salted KDFs. Never general-purpose hashes (SHA-256) for passwords; never reversible encryption.

## Authorization — three models I've shipped

**1. Ownership-scoped queries (OneTube).** Authorization folded into the SQL: `WHERE id = ? AND uploader_id = ?`. Zero rows = not yours = 404. No separate check-then-act, no TOCTOU gap.

**2. RBAC (Restaurant System).** JWT carries a `role` claim (`CUSTOMER` / `KITCHEN_STAFF`); middleware extracts it ([authmiddleware.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/middleware/authmiddleware.go), [authorization.go](https://github.com/1Kyryll/Restaurant-System-Microservices/blob/main/server/internal/middleware/authorization.go)) and **every resolver enforces server-side**: customers query only their own orders, kitchen staff sees all, `updateOrderStatus` is staff-only. Data-scoping by role, not just endpoint-gating.

**3. Auth perimeter at the gateway (E-Commerce).** Only the gateway sees credentials or tokens; internal gRPC services receive an already-authenticated user ID. One place to audit ([[API Gateway Pattern]]).

## Edge cases I've actually handled (great interview stories)
- **SSE can't send headers** — `EventSource` has no header API, so the kitchen stream authenticates via `?access_token=<jwt>` query param (Restaurant). Tradeoff: tokens can leak into server logs — acceptable for short-lived tokens on an internal dashboard, worth flagging.
- **`Set-Cookie` across a proxy chain** — Next.js server-side fetches don't auto-relay `Set-Cookie` from the gateway; E-Commerce's frontend explicitly forwards it (`forwardSetCookies()`, [ADR 010](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/010-frontend-feature-slicing.md)), and the route middleware **validates** the cookie against `/me` rather than merely checking its presence.
- JWT tradeoffs to volunteer: stateless = no DB hit per request, but **no server-side revocation** before expiry (mitigations: short TTLs + refresh tokens, or a denylist — reintroducing state).

## Related
- [[API Gateway Pattern]]
- [[REST API Design]]
- [[Real-Time Communication]]
- [[System Design MOC]]
