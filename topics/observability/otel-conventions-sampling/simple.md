# OTel semantic conventions + sampling — the simple version (the passport analogy)

> Read this first. Once the passport analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

This doc explains **two related ideas**:

> 1. **Semantic conventions are a shared passport format.** Every country (observability vendor) knows how to read your passport (trace attributes) because they all agreed on the same fields.
> 2. **Sampling is about deciding which travelers (traces) to keep records of.** Head-based sampling decides at the border (trace start). Tail-based sampling decides after you've seen the whole trip.

---

## Semantic conventions — the shared passport format

Without semantic conventions, every service team invents their own attribute names:

```
# Team A                          # Team B
http_method = "GET"               method = "get"
url = "/api/users"                endpoint = "/api/v1/users"
user_id = 12345                   userId = "12345"
```

You can't write a single dashboard or alert that works across both teams. You can't switch from Datadog to Grafana Tempo without renaming attributes everywhere.

OTel semantic conventions are the agreed-upon names:

```
http.request.method = "GET"
url.path = "/api/users"
user.id = "12345"
service.name = "checkout-api"
```

Every OTel SDK defaults to these names. Every vendor that supports OTel knows how to read them. A dashboard built on `http.request.method` works against Datadog, Grafana Tempo, Jaeger — any OTel-compatible backend.

### The namespace migration you'll hit

The HTTP conventions changed in OTel 1.21 (2023):

| Old (deprecated) | New (current) |
|---|---|
| `http.method` | `http.request.method` |
| `http.url` | `url.full` |
| `http.status_code` | `http.response.status_code` |
| `http.target` | `url.path` |

If your dashboards and alerts use the old names, they'll break when SDKs upgrade. Most vendors support both during a migration window — but you need to know this exists.

---

## Sampling — deciding which traces to keep

Distributed tracing can generate enormous data volumes. A service at 10,000 req/s with p99 latency of 20ms generates 10,000 traces per second — that's millions per day. Keeping all of them costs a fortune. Sampling keeps a representative subset.

### Head-based sampling — decide at the border

The sampling decision is made **at the very start of the trace**, before any spans are collected.

```
Request arrives → [Sample? Yes/No] → If yes: record all spans → export
                                     If no: don't record anything
```

**Simple and fast** — the decision is one random number comparison. No buffering needed.

**Limitation**: you can't keep "interesting" traces. If you sample 10% randomly, 10% of your error traces are kept and 90% are discarded — including the ones that would have shown you exactly what went wrong.

### Tail-based sampling — decide at the end of the trip

The sampling decision is made **after the trace is complete**, when you can see all the spans.

```
Request arrives → Record all spans → [After 30s: was this trace interesting?]
                                     If error → keep
                                     If slow → keep
                                     If normal → keep 10%
```

**Can keep 100% of errors and slow traces** — exactly the ones you want for debugging.

**Requires buffering**: all spans of a trace must be held in memory until the trace is "complete" (within a timeout). More complex, more resource-intensive, requires routing all spans of a trace to the same collector instance.

---

## The cost-vs-fidelity trade-off

| | Head-based | Tail-based |
|---|---|---|
| Complexity | Simple | Complex (needs central collector) |
| Error trace coverage | Probabilistic | 100% |
| Resource cost | Minimal | High (buffering) |
| Best for | Low-volume or debug environments | Production with cost constraints |

**The senior recommendation**: always use tail-based sampling in production. The whole point of distributed tracing is debugging failures — if you're randomly discarding 90% of your error traces, you've defeated the purpose.

---

## Self-test (out loud)

> **"Why do OTel semantic conventions matter, and what's the difference between head-based and tail-based sampling?"**

**Reference answer:**

"Semantic conventions are standardized attribute names — `service.name`, `http.request.method`, `db.system`. They matter because they make telemetry portable: dashboards, alerts, and tools built on standard attribute names work against any OTel-compatible backend without changes. Without them, every team invents its own schema and you end up with 5 different names for the HTTP method.

Head-based sampling decides at trace start — simple, no buffering, but you're sampling randomly. 90% of your error traces get discarded. Tail-based sampling waits until a trace is complete and then decides — 100% of error and slow traces are kept, the rest are downsampled. The cost is complexity: you need a central collector with buffering and sticky routing (all spans of a trace must land on the same replica)."

---

## Further reading

- [OTel semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [OTel HTTP semantic conventions migration](https://opentelemetry.io/docs/specs/semconv/http/migration-guide/)
- [OTel sampling docs](https://opentelemetry.io/docs/concepts/sampling/)

---

## Next: the deep-dive

When the passport and border analogies are obvious, jump to [`otel-conventions-sampling.md`](./deep-dive.md). The deep-dive covers:

- The full semantic convention namespaces (http, db, rpc, messaging, cloud, k8s)
- The http → http.request migration in detail with migration strategies
- Head-based sampling: AlwaysSample, TraceIDRatioBased, ParentBased
- Tail-based sampling: the tail_sampling processor config with multiple policies
- The W3C TraceContext propagation standard
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
