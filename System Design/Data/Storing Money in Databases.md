---
tags: [system-design, database, data-modeling]
---

# Storing Money in Databases

The most error-prone data-layer decision. Full analysis: [ADR 008](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/008-money-datatype.md).

## Rule zero: never float
`0.1 + 0.2 == 0.30000000000000004` — IEEE 754 binary floats cannot represent decimal fractions exactly, in *any* language. Accumulated rounding errors surface as failed reconciliations and off-by-a-cent invoices. **No exceptions.**

Also ignore Postgres's built-in `money` type: locale-dependent display, carries no currency info, driver compatibility issues.

## The two correct options

**1. Integer smallest-unit (cents):** `price_amount BIGINT` storing `1999` for €19.99. Exact, fast, universal. Failure mode: forgetting to divide/multiply by 100 at the display/input boundary ("1999 EUR"), plus awkward sub-cent intermediate values (proportional tax).

**2. `NUMERIC` (arbitrary-precision decimal):** what I chose.

```sql
price_amount   NUMERIC(18, 4) NOT NULL,
price_currency CHAR(3)        NOT NULL,   -- ISO 4217
```

Why NUMERIC won (ADR 008): no display-conversion failure mode, natively handles currencies with 0 or 3 decimal places (JPY, BHD), and stores intermediate precision faithfully — 23% VAT on €19.99 = €4.5977 goes in as-is, no premature rounding decision. The 2 extra decimal places beyond cents exist exactly for such intermediates. NUMERIC is slower than BIGINT but still millions of ops/sec — a non-issue.

## Non-negotiable: amount and currency travel together
A `price_amount` without a `price_currency` **in the same row** is a latent bug that detonates the day a second currency appears. In my schema every money column is paired ([data-layer.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/data-layer.md)): `products.price_amount/price_currency`, `orders.total_amount/total_currency`, `order_items.unit_price_amount/unit_price_currency`.

## Application layer — make mixing currencies a compile-time error

[internal/money/money.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/internal/money/money.go), using `shopspring/decimal` (the standard Go arbitrary-precision decimal library, integrates cleanly with pgx/NUMERIC):

```go
type Money struct {
    Amount   decimal.Decimal
    Currency string // ISO 4217
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, ErrCurrencyMismatch
    }
    return Money{Amount: m.Amount.Add(other.Amount), Currency: m.Currency}, nil
}
```

The checkout flow enforces it too — [place_order.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/place_order.go) rejects mixed-currency carts before charging.

On the frontend, money stays a **decimal string** `{ amount, currency }` end to end (never parsed to a JS `number` — same float trap), formatted centrally in `lib/money.ts` ([ADR 010](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/adr/010-frontend-feature-slicing.md)).

## Related
- [[Primary Keys - UUIDv7]]
- [[Postgres in Production]]
- [[Data Index]]
