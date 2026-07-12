---
tags: [system-design, operations, performance, testing]
---

# Load Testing with k6

Load testing answers "does the design survive contact with concurrency?" — for my E-Commerce system, literally the design goal (*no oversell under 10K contending users*) is only provable under load.

Scripts: [load-tests/](https://github.com/1Kyryll/E-Commerce-System/tree/main/load-tests) · Docs: [load-testing.md](https://github.com/1Kyryll/E-Commerce-System/blob/main/docs/load-testing.md)

## The test suite (scenario per traffic shape)

| Script | Models |
|---|---|
| [smoke.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/smoke.js) | Minimal sanity — does the deploy work at all (run first, always) |
| [browse.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/browse.js) | Read-heavy catalog browsing (the cacheable path) |
| [cart.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/cart.js) | Cart mutations |
| [checkout.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/checkout.js) | Full purchase funnel — the contention path |

## What a realistic scenario looks like ([checkout.js](https://github.com/1Kyryll/E-Commerce-System/blob/main/load-tests/checkout.js))

```js
export const options = {
  scenarios: {
    checkout: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m',  target: TARGET_VUS },  // ramp up
        { duration: '3m',  target: TARGET_VUS },  // steady state
        { duration: '30s', target: 0 },           // ramp down
      ],
    },
  },
};

export default function () {
  ensureAuthenticated(data.runId);        // each VU is a logged-in user
  const page = listProducts('', 20);      // browse
  sleep(0.5 + Math.random() * 1.5);       // think time!
  addItem(p.id, 1);                       // fill cart (1–2 items)
  const result = placeOrder();
  check(result, { 'checkout ok or 409': (r) => r.ok || r.status === 409 });
  ...
}
```

Details that separate a real load test from `ab -n 10000`:
- **Ramp stages**, not instant full load — find *where* it degrades, not just *that* it does.
- **Think time** (randomized sleeps) — real users pause; without it you model a DDoS, not traffic.
- **Full user journeys** — auth → browse → cart → checkout → confirmation view, so load hits every service including session handling.
- **`409 is a pass`** — under contention, "sold out" is correct behavior. Asserting only 2xx would mark the system's *best* property as failure. Load tests must encode the domain's definition of correct.
- ~20% of payments are declined by the fake payment client — failure paths are part of the load profile (they trigger reservation releases, exercising the cleanup path).

## Closing the loop: k6 metrics → Grafana

k6 pushes client-side metrics into Mimir via Prometheus remote-write ([README](https://github.com/1Kyryll/E-Commerce-System#k6-metrics--grafana)):

```bash
K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9009/api/v1/push \
k6 run --out experimental-prometheus-rw load-tests/checkout.js
```

Client-observed latency (k6) and server-side traces/metrics (OTel) land on **one dashboard** — when p95 spikes you immediately see whether it's the gateway, the order service, or Postgres. See [[Observability]].

## Vocabulary to have ready
- **VU** (virtual user), **RPS**, open vs closed workload models (ramping-vus is closed; arrival-rate executors are open — better for modeling "requests arrive regardless of how slow you respond")
- **p95/p99 latency** — averages lie; tail latency is what users feel
- Smoke → load → stress → soak (long-duration, catches leaks) test ladder
- Little's Law: `concurrency = arrival_rate × latency` — sanity-checks any load test's numbers

## Related
- [[Observability]]
- [[Database Concurrency and Atomicity]] — what the checkout test proves
- [[Operations Index]]
