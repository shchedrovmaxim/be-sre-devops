# OTel semantic conventions + sampling — the deep-dive

> **Goal**: by the end you can answer — **"Why do OTel semantic conventions matter, and what's the difference between head-based and tail-based sampling?"** — covering convention namespaces, the HTTP migration, head/tail sampling mechanics, and collector-level tail sampling config.

> Start with [otel-conventions-sampling-simple.md](./otel-conventions-sampling-simple.md) first. The passport and border analogies are the spine.

---

## The senior framing — conventions + sampling as cost control

Observability data is expensive. At 10K req/s, a single service generates ~864M spans per day if you keep everything. At ~$1.50 per GB ingested for a typical SaaS backend (Datadog, Honeycomb, Grafana Cloud), full-fidelity tracing of a medium-sized system costs tens of thousands of dollars per month.

The combination of:
1. **Semantic conventions** — so you can write portable queries across any sampling/storage strategy
2. **Tail-based sampling** — keeping the *right* 10% (errors + slow traces), not a random 10%

is how mature teams get **high debugging fidelity at reasonable cost**. The goal isn't to sample less — it's to sample smarter.

---

## Semantic conventions — the full picture

OTel semantic conventions define attribute names for common operations. They're organized by namespace:

### Core resource attributes

Every piece of telemetry should carry these (usually set once by the SDK + resource detection):

| Attribute | Description | Example |
|---|---|---|
| `service.name` | The logical service name | `checkout-api` |
| `service.version` | Deployed version | `1.4.2` |
| `service.namespace` | Organizational grouping | `ecommerce` |
| `deployment.environment` | Environment | `production` |
| `k8s.pod.name` | Pod name (from k8sattributes) | `checkout-6f4b8d-xkpzq` |
| `k8s.namespace.name` | Kubernetes namespace | `production` |
| `k8s.deployment.name` | Deployment name | `checkout` |
| `cloud.provider` | Cloud provider | `aws` |
| `cloud.region` | Cloud region | `us-east-1` |

### HTTP conventions (the migrated ones)

The HTTP conventions changed in OTel semconv v1.21 (OTel SDK v1.20+). Both are in active use:

| Old (deprecated) | New (current) | Notes |
|---|---|---|
| `http.method` | `http.request.method` | GET, POST, etc. |
| `http.url` | `url.full` | Full URL |
| `http.target` | `url.path` | Path + query |
| `http.route` | `http.route` | Still the same |
| `http.status_code` | `http.response.status_code` | 200, 404, 500 |
| `http.scheme` | `url.scheme` | http or https |
| `http.host` | `server.address` | Hostname |
| `http.server_name` | `server.address` | Same target |
| `net.peer.ip` | `network.peer.address` | Upstream IP |

**Migration strategy**: during the transition, most backends and dashboards support both. Use `|| (OR)` queries in Grafana when building dashboards:

```
# Grafana / Tempo TraceQL
{ span.http.request.method = "GET" || span.http.method = "GET" }
```

Update recording rules and SLO queries when you control the SDK version and are on the new convention.

### Database conventions

| Attribute | Description | Example |
|---|---|---|
| `db.system` | Database type | `postgresql`, `redis`, `mongodb` |
| `db.name` | Database name | `users` |
| `db.operation.name` | SQL verb | `SELECT`, `INSERT` |
| `db.query.text` | Query text (sanitized) | `SELECT * FROM users WHERE id = ?` |
| `server.address` | DB host | `db.internal` |
| `server.port` | DB port | `5432` |

### Messaging conventions

| Attribute | Description | Example |
|---|---|---|
| `messaging.system` | Broker | `kafka`, `rabbitmq`, `sqs` |
| `messaging.destination.name` | Topic/queue | `orders` |
| `messaging.operation.type` | `publish`, `receive`, `settle` | `publish` |
| `messaging.message.id` | Message ID | `abc123` |

### RPC conventions

| Attribute | Description | Example |
|---|---|---|
| `rpc.system` | `grpc`, `jsonrpc` | `grpc` |
| `rpc.service` | Service name | `checkout.v1.CheckoutService` |
| `rpc.method` | Method name | `CreateOrder` |
| `rpc.grpc.status_code` | gRPC status | `0` (OK), `5` (NOT_FOUND) |

---

## Trace context propagation

For distributed tracing to work, the trace ID and span ID must be passed between services. The W3C TraceContext standard defines the `traceparent` HTTP header:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ver  trace-id (128-bit)                span-id (64-bit)  flags
```

The `flags` byte: `01` = sampling flag set (keep this trace), `00` = not sampled.

**Baggage**: the `tracestate` and `baggage` headers allow passing key-value pairs across service boundaries without instrument-specific logic. Useful for propagating `user.id` or `feature.flag` without putting them in every span manually.

OTel SDKs propagate `traceparent` (and optionally `baggage`) automatically in outbound HTTP/gRPC calls. You don't write this code.

---

## Head-based sampling — mechanics and config

The sampling decision is made at span creation time in the SDK.

### Sampler types

**AlwaysSample** — keep everything:
```python
# Python SDK
from opentelemetry.sdk.trace.sampling import ALWAYS_ON
tracer_provider = TracerProvider(sampler=ALWAYS_ON)
```

**TraceIDRatioBased** — keep a percentage by hashing the trace ID:
```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
sampler = TraceIdRatioBased(0.1)  # keep 10%
```

All spans of a trace that's sampled are kept (deterministic — same trace ID always produces the same decision). Traces that are dropped generate no spans at all — zero cost.

**ParentBased** — respect the sampling decision from the parent span:
```python
from opentelemetry.sdk.trace.sampling import ParentBased, TraceIdRatioBased
sampler = ParentBased(root=TraceIdRatioBased(0.1))
```

If the inbound `traceparent` says "sampled = yes", keep it. Otherwise use the root sampler. This ensures that a trace started by service A is either fully sampled or not sampled across all downstream services — no partial traces.

**The production head-based pattern:**

```python
# Service entrypoint (e.g., API gateway)
sampler = ParentBased(root=TraceIdRatioBased(0.1))  # 10% of new traces

# Downstream services
sampler = ParentBased(root=ALWAYS_OFF)  # follow parent's decision
```

### Head-based limitations

You **cannot** use head-based sampling to keep 100% of error traces — the error hasn't happened yet at sampling decision time. For that, you need tail-based.

You also can't sample based on a user attribute (`user.tier == "premium"` → always sample) because that attribute might not be available at trace start.

---

## Tail-based sampling — mechanics and config

Tail-based sampling happens in the **OTel Collector gateway** (not the SDK). The `tail_sampling` processor buffers all spans of a trace until the trace is "complete" (within `decision_wait`), then applies policies.

```yaml
processors:
  tail_sampling:
    decision_wait: 30s        # buffer spans for up to 30s after last span received
    num_traces: 100000        # max traces in memory at once; ~500MB at 5KB/trace
    expected_new_traces_per_sec: 1000
    policies:
      # Policy 1: Keep all error traces
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Policy 2: Keep slow traces (p99 > 1s)
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000

      # Policy 3: Always keep traces from premium users
      - name: premium-users
        type: string_attribute
        string_attribute:
          key: user.tier
          values: ["premium", "enterprise"]

      # Policy 4: Keep 10% of everything else
      - name: baseline-sample
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

      # Policy 5: Composite — AND/OR logic
      - name: complex-policy
        type: composite
        composite:
          max_total_spans_per_second: 1000
          policy_order: [errors, slow-traces, premium-users, baseline-sample]
          composite_sub_policy:
            - name: and-example
              type: and
              and:
                and_sub_policy:
                  - name: span-count-low
                    type: span_count
                    span_count:
                      min_spans: 5
                  - name: slow
                    type: latency
                    latency:
                      threshold_ms: 500
```

### Policies evaluated in order

Policies are evaluated in the order listed. The first policy that matches determines the sampling decision. A trace not matched by any policy is dropped (configure a catch-all `probabilistic` policy to avoid this).

### Resource cost

At 1K new traces/s with 10 spans/trace × 5KB/span × 30s buffer window:
```
100,000 traces × 50KB/trace = ~5 GB memory in the gateway
```

This is why the gateway needs significant memory. Size accordingly.

### The routing requirement

With multiple gateway replicas, you must ensure all spans for a trace land on the same replica. In agents, use the `loadbalancing` exporter:

```yaml
exporters:
  loadbalancing:
    routing_key: traceID
    protocol:
      otlp:
        timeout: 5s
    resolver:
      dns:
        hostname: otelcol-gateway-headless.monitoring.svc.cluster.local
        port: 4317
```

The loadbalancing exporter hashes the trace ID to consistently route to the same gateway instance.

---

## Sampling in the SDK vs the Collector

| | SDK (head-based) | Collector (tail-based) |
|---|---|---|
| Decision point | At span creation | After trace completes |
| Can keep error traces | No | Yes |
| Resource cost | Minimal | High (buffering) |
| Config location | Application code / env var | Collector config |
| Recommended for | Dev/staging or when full fidelity is needed | Production |

**The hybrid pattern**: use `ParentBased(TraceIdRatioBased(1.0))` in the SDK (keep all spans) and do the actual sampling in the collector. This lets you change sampling policy without redeploying application code.

---

## The 60-second interview answer

> "Semantic conventions are standardized attribute names — `service.name`, `http.request.method`, `db.system`, `k8s.pod.name`. They matter because they make telemetry portable: dashboards, SLO queries, and alerts built on standard names work against any OTel-compatible backend without modification. There's an ongoing HTTP namespace migration from the old `http.method` / `http.status_code` to `http.request.method` / `http.response.status_code` — worth knowing if you're upgrading SDKs.
>
> Head-based sampling decides at trace start — `TraceIdRatioBased(0.1)` keeps 10% by trace ID hash. Simple and cheap, but you can't selectively keep error or slow traces.
>
> Tail-based sampling happens in the OTel Collector gateway. The `tail_sampling` processor buffers all spans of a trace for up to 30 seconds, then evaluates policies: keep all ERROR status spans, keep all traces over 1s latency, keep 10% of the rest. This is the production pattern — 100% of errors, 100% of slow traces, 10% of normal. The constraint: all spans of a trace must land on the same gateway replica, so agents use the `loadbalancing` exporter with `routing_key: traceID`. The buffer costs memory — size the gateway for your trace volume."

---

## Self-test drills

### Drill 1 — conventions

A teammate writes a Grafana alert using `http.status_code`. Their colleague's SDK was just upgraded to OTel 1.22+. What happens?

**Answer:** The newer SDK emits `http.response.status_code` instead of `http.status_code`. The alert query stops matching and the alert stops firing — a false "all clear" during real errors. Fix: update the alert to query both: `{span.http.response.status_code OR span.http.status_code}` during the migration window, then drop the old attribute once all SDKs are upgraded.

### Drill 2 — head vs tail

Your service processes 5,000 req/s. You're getting paged about high latency but can't find the slow traces in your tracing backend. Why?

**Answer:** You're likely using head-based sampling at 1% (50 req/s sampled). With 5,000 req/s at, say, 0.1% slow traces (5 slow req/s), you're sampling 0.05 slow req/s — one every 20 seconds. Most of your slow traces are discarded before they reach the backend. Switch to tail-based sampling with a policy to keep all traces over your SLO latency threshold.

### Drill 3 — routing

You deploy 3 OTel gateway replicas with `tail_sampling`. A trace has 50 spans across 10 services. After an incident, you look for the trace in Tempo and find only 20 spans — the rest are missing. What happened?

**Answer:** Different spans of the trace were routed to different gateway replicas. Each replica had an incomplete view of the trace — some policies (like `span_count: min_spans=5`) may have made different decisions, and even if all replicas sampled it, only partial spans were exported per replica. The fix is the `loadbalancing` exporter in agents with `routing_key: traceID` to ensure all spans land on one replica.

---

## Further reading

- [OTel semantic conventions reference](https://opentelemetry.io/docs/specs/semconv/)
- [OTel HTTP semantic conventions migration guide](https://opentelemetry.io/docs/specs/semconv/http/migration-guide/)
- [OTel sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [tail_sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)
- [W3C TraceContext spec](https://www.w3.org/TR/trace-context/)

---

## The 4 dimensions

- **Tech**: semantic conventions (`service.name`, `http.request.method`, `db.system`, `k8s.pod.name`) are the portability contract. W3C TraceContext (`traceparent`) is the propagation wire format. Head-based: `TraceIdRatioBased` + `ParentBased` in SDK, zero collector cost. Tail-based: `tail_sampling` processor in gateway, policies by status_code/latency/attribute, `loadbalancing` exporter in agent for sticky routing. Hybrid pattern: SDK keeps all spans, collector samples.
- **People**: teams don't need to know the sampling policy — they just instrument to OTel conventions and send to the local agent. Platform team controls the sampling policy centrally in the collector config. A PR to change "keep 10% baseline" to "keep 20% baseline" is a one-line collector config change, reviewed and deployed without touching application code.
- **CI/CD**: collector config changes that affect sampling are high-blast-radius (they change what you can see in incidents). Gate behind staging canary. Monitor `otelcol_processor_dropped_items` after rollout — a jump means sampling is tighter than expected. Semantic convention migrations are a two-phase process: update dashboard queries first (to accept both old and new attribute names), then upgrade SDKs, then clean up.
- **Operations**: `otelcol_processor_tail_sampling_count_traces_sampled` vs `count_traces_not_sampled` is the sampling efficiency metric — you want the ratio to match your policy intent. If errors are present but `errors` policy isn't firing, the `status_code` attribute may be missing (SDK not setting span status). Monitor `tail_sampling_late_release_decisions` (spans arriving after decision_wait) — a sign that `decision_wait` is too short or the service is slow to complete traces.
