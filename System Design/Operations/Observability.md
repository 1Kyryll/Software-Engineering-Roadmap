---
tags: [system-design, observability, operations]
---

# Observability — OpenTelemetry + Grafana LGTM

Observability = being able to ask *new* questions about a running system without shipping new code. Three signals: **traces** (one request's journey across services), **metrics** (aggregates over time), **logs** (discrete events). My E-Commerce system wires all three through OpenTelemetry into the Grafana LGTM stack.

Docs: [observability.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/observability.md) · Config: [observability/](https://github.com/1Kyryll/E-Commerce-System/tree/main/observability)

## The pipeline

```
Go services ──OTLP/gRPC──► OTel Collector ──► Tempo   (traces)
                                          ──► Mimir   (metrics, Prometheus-compatible)
                                          ──► Loki    (logs)
                                                └──► Grafana (one UI over all three)
```

- Every binary calls one bootstrap: [internal/observability/observability.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/internal/observability/observability.go) — sets up trace/metric/log providers, OTLP exporters, W3C `traceparent` propagation, runtime metrics. **Graceful degradation**: if `OTEL_EXPORTER_OTLP_ENDPOINT` is unset it installs no-op providers, so unit tests and local runs need no collector.
- The **collector** ([otel-collector-config.yaml](https://github.com/1Kyryll/E-Commerce-System/blob/main/observability/otel-collector-config.yaml)) is the routing layer: services export one protocol to one place; backends can be swapped without touching app code. That's the argument for OTel over vendor SDKs — instrument once, choose backends later.
- Logs: structured `slog` with trace IDs attached ([slog.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/internal/observability/slog.go)) → **logs are correlated to traces**; from a slow trace in Tempo you can jump to its exact log lines in Loki.

## Instrumentation in practice ([place_order.go](https://github.com/1Kyryll/E-Commerce-System/blob/main/backend/services/order/service/place_order.go))

**Spans around meaningful phases**, not every function:

```go
ctx, reserveSpan := tracer.Start(ctx, "PlaceOrder.reserve",
    trace.WithAttributes(attribute.Int("items.count", len(items))))
...
chargeCtx, chargeSpan := tracer.Start(ctx, "PlaceOrder.charge", ...)
...
finalizeCtx, finalizeSpan := tracer.Start(ctx, "PlaceOrder.finalize", ...)
```

A checkout trace shows reserve → charge → finalize as child spans (with `otelgrpc`/`otelhttp` adding the cross-service hops), so "checkout is slow" decomposes instantly into *which phase*.

**Metrics tagged by outcome**, defined next to the business logic:

```go
placeOrderTotal, _ = meter.Int64Counter("order.place_order.total", ...)
// every exit path: recordOutcome(ctx, "ok" | "replay" | "insufficient_inventory"
//                  | "payment_declined" | "mixed_currency" | ...)
paymentOutcomes.Add(ctx, 1, metric.WithAttributes(attribute.String("result", paymentResult)))
```

This is **RED-method** instrumentation (Rate, Errors, Duration) with domain-specific outcome labels — you can graph "payment declines per second" or alert on `insufficient_inventory` spikes without log parsing. The nightly inventory-invariant check exports a gauge the same way ([[Denormalization and Invariants]]) — *business invariants as metrics*.

**Context propagation is the glue**: `ctx` carries the trace through every function and across gRPC hops (W3C traceparent). This is the second reason "context is the first argument everywhere" is a hard rule in my Go code (the first: cancellation/deadlines).

## Load-test integration
k6 pushes its metrics into the same Mimir via Prometheus remote-write, so load-test client-side latency and server-side traces/metrics sit on one Grafana dashboard — see [[Load Testing with k6]].

## Interview framing
- Monitoring answers known questions (dashboards, alerts); observability lets you explore unknown ones (trace exemplars, high-cardinality attributes).
- Sampling: I run `ParentBased(AlwaysSample())` — fine at demo scale; at production volume you'd head- or tail-sample (tail = keep the interesting/slow/error traces).
- LGTM vs alternatives: same shape as Datadog/Honeycomb etc.; OTel keeps you portable between them.

## Related
- [[Load Testing with k6]]
- [[Docker Compose Orchestration]]
- [[Operations Index]]
