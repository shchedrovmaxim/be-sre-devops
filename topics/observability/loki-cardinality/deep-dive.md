# Loki + label cardinality discipline — the deep-dive

> **Goal**: by the end you can answer — **"Why is Loki's promise of 'just shove logs in cheaply' contingent on label discipline, and what happens if you screw it up?"** — covering the storage model, cardinality detection, LogQL extraction patterns, structured logging, and Loki ingestion limits.

> Start with [loki-cardinality-simple.md](./simple.md) first. The library analogy is the spine.

---

## The senior framing — Loki's model is a trade, not a free lunch

Loki's cost advantage over Elasticsearch comes from one architectural choice: **index labels, not content**. The index is tiny. Logs are stored compressed in object storage (S3/GCS). Queries filter on labels to select streams, then scan the actual log chunks for content matches.

This trade has a hard constraint: if the label set grows without bound, the index still grows without bound. The cost advantage evaporates. At high cardinality, Loki's query performance degrades worse than Elasticsearch because Loki wasn't designed to do index lookups across millions of streams — it was designed to do a few coarse lookups and then filter in-stream.

The senior understanding: **Loki's scalability is not about total log volume — it's about stream count (cardinality).** A system ingesting 10 TB/day with 1,000 streams is easy. A system ingesting 100 GB/day with 10 million streams is a disaster.

---

## Loki storage model — streams, chunks, index

### Streams

A **stream** is a unique combination of label key-value pairs:

```
{app="checkout", namespace="production", env="prod"}
```

Every log line belongs to exactly one stream. Loki writes log lines from the same stream into sequential chunks.

### Chunks

A **chunk** is a compressed block of log lines from a single stream, covering a time range (typically minutes to hours). Chunks are stored in object storage. Loki reads chunks to answer log queries (the "pulling books off the shelf" step).

### Index

The **index** maps stream label sets to the list of chunks containing their log lines. For a query `{app="checkout", namespace="production"}`, Loki:
1. Looks up the stream in the index → gets a list of chunk locations.
2. Fetches those chunks from S3.
3. Optionally filters the log lines within chunks by log-level filters or LogQL expressions.

### Why cardinality breaks this model

If you have 10 million streams:
- The index has 10 million rows.
- Writing to a new stream requires an index write.
- A query like `{app="checkout"}` now matches potentially hundreds of thousands of streams, each with their own chunks. Loki has to fetch many chunk lists from the index and fan out reads across many chunks.
- Ingest queues back up because ingester memory is consumed managing too many open chunks (one per active stream).
- Object storage request rates spike (many small chunk flushes instead of few large ones).

### The `chunks_per_stream` canary

```logql
# Via Loki API (pseudo-query)
sum by (stream) (count_over_time({app="checkout"}[24h]))
```

A healthy stream has steady log output and few chunks per stream per day. An exploding cardinality shows as:
- `loki_ingester_chunks_created_total` growing faster than log volume
- `loki_ingester_memory_chunks` growing without bound
- Many one-or-two-line chunks (because each request gets its own stream that immediately flushes)

Loki exposes `loki_ingester_streams_created_total` — alert when the rate of stream creation exceeds your expected deployment churn rate.

---

## What Loki's index actually contains

```
# Each row is approximately:
stream_fingerprint (hash of label set) → [chunk_ref_1, chunk_ref_2, ...]
                                          (start_time, end_time, S3 key)
```

For Loki with boltdb-shipper or TSDB store (the modern backends), the index is stored in object storage too, but cached on ingesters and queriers for fast access. The in-memory cache size limits how many streams can be efficiently queried.

At 1 million streams, the index is ~1–2 GB in memory. At 10 million streams, it's 10–20 GB — and many query fanouts across millions of stream metadata entries become IO-bound.

---

## The label discipline rules

### Rule 1: Labels are for stream selection, not search

A label is correct if you'd say "give me all logs from the checkout service in production" — `{app="checkout", env="prod"}`. One query, one stream.

A label is wrong if you'd say "give me the log where request_id equals this UUID" — that's a search, not a stream selection.

### Rule 2: Maximum ~10–20 labels per stream

```bash
# Common Kubernetes deployment label set (reasonable):
{
  "app": "checkout",
  "namespace": "production",
  "env": "prod",
  "container": "checkout",
  "node": "ip-10-0-1-5"    # ← marginal: changes on pod reschedule
}
```

Labels beyond ~20 are almost certainly log fields masquerading as stream labels.

### Rule 3: Pod name is a cardinality trap

```yaml
# Promtail / OTel filelog receiver common config
pipeline_stages:
  - labels:
      app:
      namespace:
      # pod: __meta_kubernetes_pod_name  ← DON'T do this
```

A deployment with 50 replicas rotating every hour → 50 × 24 = 1,200 pod names per day × all your services. This is a common cause of Loki index bloat in Kubernetes.

If you need per-pod filtering, use `| logfmt | pod_name="checkout-6f4b8d-xkpzq"` at query time.

---

## LogQL — extracting fields at query time

The power of Loki's model is that you don't need labels for everything — LogQL can extract fields from log content at query time.

### `| json` — parse JSON log lines

```logql
{app="checkout"} | json
```

If log lines are JSON objects, `| json` parses them and makes all top-level keys available as labels for further filtering.

```logql
{app="checkout"} | json | level="error" | user_id="12345"
```

This selects the `checkout` stream (one index lookup), reads the chunks, parses each line as JSON, and filters by `level` and `user_id` — all without indexing either field.

### `| logfmt` — parse key=value log lines

```logql
{app="checkout"} | logfmt
```

Parses lines in the format `key=value key=value ...`:

```
2026-06-05T12:34:56Z level=error msg="payment failed" user_id=12345 trace_id=abc
```

After `| logfmt`: `level`, `msg`, `user_id`, `trace_id` are all filterable.

### `| pattern` — extract with named patterns

```logql
{app="nginx"} | pattern `<_> "<method> <path> <_>" <status> <_>`
```

Extracts `method`, `path`, `status` from NGINX access logs without regex. Fast.

### `| regexp` — extract with named capture groups

```logql
{app="checkout"} | regexp `(?P<level>\w+): (?P<msg>.*)`
```

For log lines that aren't JSON or logfmt.

### Metric queries from logs

```logql
# Request rate from log lines (no Prometheus required)
sum by (status) (
  rate({app="checkout"} | json | __error__="" [5m])
)

# Error ratio from logs
sum(rate({app="checkout"} | json | level="error" [5m]))
/
sum(rate({app="checkout"} [5m]))

# p99 latency from logs (if duration_ms is a log field)
quantile_over_time(0.99,
  {app="checkout"} | json | unwrap duration_ms [5m]
) by (method)
```

`unwrap` converts a log field to a numeric value for aggregation. This is how you get metric-style aggregations from logs without Prometheus.

---

## Structured logging — the practical prerequisite

Loki's `| json` and `| logfmt` only work if your applications write structured logs.

### Node.js (pino)

```javascript
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({ user_id: req.userId, trace_id: req.traceId, duration_ms: elapsed }, 'order created');
// Output: {"level":30,"time":1717569296000,"user_id":"12345","trace_id":"abc","duration_ms":42,"msg":"order created"}
```

### Go (slog)

```go
import "log/slog"

slog.Info("order created",
    "user_id", userID,
    "trace_id", traceID,
    "duration_ms", elapsed.Milliseconds(),
)
// Output: time=2026-06-05T12:34:56Z level=INFO msg="order created" user_id=12345 trace_id=abc duration_ms=42
```

### Python (structlog)

```python
import structlog
log = structlog.get_logger()

log.info("order_created", user_id=user_id, trace_id=trace_id, duration_ms=elapsed)
# Output: {"event": "order_created", "user_id": "12345", "trace_id": "abc", "duration_ms": 42, "level": "info"}
```

The key: **`trace_id` and `user_id` are JSON fields, not Loki stream labels**. They're searchable with `| json | trace_id="abc"` — no index cost.

---

## Loki ingestion limits — the guardrails

Loki has configurable limits to prevent cardinality explosions. In a Kubernetes Loki deployment (via Helm), these go in `values.yaml`:

```yaml
loki:
  limits_config:
    # Max number of active streams per tenant
    max_streams_per_user: 50000    # default 0 (unlimited) — set this!

    # Reject log lines with more labels than this
    max_label_names_per_series: 30

    # Max length of a label value
    max_label_value_length: 2048

    # Ingestion rate limit
    ingestion_rate_mb: 16          # MB/s per tenant
    ingestion_burst_size_mb: 32

    # Reject streams that produce too many tiny chunks
    # (sign of high-cardinality ingest)
    per_stream_rate_limit: 3MB     # max 3 MB/s per individual stream
```

`max_streams_per_user: 50000` is the critical one. When exceeded, new streams are rejected with a 429. This is painful but better than silent index explosion.

---

## Detecting cardinality issues

### Via the Loki API

```bash
# Number of active streams per tenant
curl http://loki:3100/loki/api/v1/labels

# Count labels (a growing label list is a smell)
curl http://loki:3100/loki/api/v1/label/pod/values | jq '.data | length'

# Loki ring/ingester status
curl http://loki:3100/ring
```

### Via Loki's own metrics

```promql
# Streams created per minute (should be low and stable after initial deployment)
rate(loki_ingester_streams_created_total[5m])

# Active streams in memory
loki_ingester_memory_streams

# Chunks currently held in memory
loki_ingester_memory_chunks

# Ingestion errors (stream limit exceeded)
rate(loki_discarded_samples_total{reason="stream_limit_exceeded"}[5m])
```

Alert on `loki_discarded_samples_total{reason="stream_limit_exceeded"} > 0` — that means you're dropping logs.

---

## The 60-second interview answer

> "Loki indexes stream labels, not log content. That's the cost win — a tiny index, logs stored compressed in cheap object storage. But the model has a hard constraint: stream cardinality. Each unique label combination is a stream. Add `request_id` as a label and you get one stream per request — millions of streams per day. The index explodes, ingesters spend all their memory managing millions of open chunks, queries fan out across millions of stream metadata entries, and ingest queues back up.
>
> The fix is label discipline: labels only for what you'd use to select a whole stream — `app`, `namespace`, `env`. Everything else — `request_id`, `trace_id`, `user_id`, `duration_ms` — lives in the log line body as structured JSON or logfmt fields. LogQL's `| json` or `| logfmt` extracts them at query time without indexing cost.
>
> The canary metric is `loki_ingester_streams_created_total` — if stream creation rate is growing faster than your deployment churn rate, you have a cardinality problem. Loki's `max_streams_per_user` limit is the hard brake — set it explicitly so cardinality explosions cause 429s (noisy, fixable) rather than silent index bloat (quiet, expensive, hard to recover from)."

---

## Self-test drills

### Drill 1 — label discipline

A developer wants to add `http_status_code` as a Loki stream label for a high-traffic API. The API handles 5,000 req/s and returns 5 distinct status codes. Should they add it?

**Answer:** 5 distinct status codes sounds low, but consider the multiplication effect: 5 status codes × 10 apps × 5 namespaces × 3 environments = 750 streams, which is acceptable. But if there are also `method` (6 values) and `region` (5 values) labels, that's 5 × 6 × 5 × 10 × 5 × 3 = 22,500 streams — already high. And status codes sometimes expand (custom codes, etc.). Better to keep `status_code` as a log field and filter with `| json | status_code="500"`. The performance hit of in-stream filtering at 5K req/s is negligible for debugging queries; the cardinality risk is not.

### Drill 2 — LogQL

Write a LogQL query to find all error logs for user `12345` from the checkout service in production, where log lines are JSON.

**Answer:**
```logql
{app="checkout", namespace="production"}
| json
| level="error"
| user_id="12345"
```

### Drill 3 — metrics from logs

Write a LogQL metric query to compute the 5-minute rate of error log lines in the `checkout` service, broken down by error type (assuming `error_type` is a JSON field in the log line).

**Answer:**
```logql
sum by (error_type) (
  rate(
    {app="checkout", namespace="production"} | json | level="error" [5m]
  )
)
```

### Drill 4 — cardinality detection

`loki_discarded_samples_total{reason="stream_limit_exceeded"}` starts incrementing. What's the immediate mitigation and the root cause investigation?

**Answer:**
- **Immediate mitigation**: temporarily raise `max_streams_per_user` in Loki's limits config to stop dropping logs. This buys time but doesn't fix the root cause.
- **Root cause investigation**: look at `loki_ingester_memory_streams` — how many streams are active? Use `curl .../loki/api/v1/label/<label_name>/values | jq '.data | length'` to count distinct values per label and find which label has an unbounded value set. Common culprits: `pod` (from pod churn), `trace_id` (incorrect label), `node` (too granular).
- **Fix**: remove the high-cardinality label from the log shipper config (Promtail `pipeline_stages.labels`, OTel Collector filelog receiver relabeling). Once removed, old streams age out naturally (Loki drops streams with no ingestion after the retention period).

---

## Further reading

- [Loki label best practices](https://grafana.com/docs/loki/latest/get-started/labels/best-practices/)
- [Loki limits configuration](https://grafana.com/docs/loki/latest/configure/#limits_config)
- [LogQL reference](https://grafana.com/docs/loki/latest/query/)
- [Loki metrics reference](https://grafana.com/docs/loki/latest/operations/loki-canary/)
- [Grafana blog — Loki's storage model](https://grafana.com/blog/2020/06/25/how-grafana-loki-stores-logs-with-the-ability-to-query-across-all-their-logs/)

---

## The 4 dimensions

- **Tech**: Loki indexes stream labels only — tiny index, cheap storage. Cardinality = number of unique label combinations = number of streams. The killers: `request_id`, `user_id`, `trace_id`, `pod_name` as stream labels. The fix: coarse stable labels + structured JSON log lines + LogQL extraction at query time. Key metrics: `loki_ingester_streams_created_total`, `loki_ingester_memory_streams`, `loki_discarded_samples_total{reason="stream_limit_exceeded"}`. Guard: `max_streams_per_user` in limits config.
- **People**: developers instinctively want high-resolution labels ("I want to filter by user_id!"). The education is: structured logging lets you filter by user_id at query time without paying the indexing cost. Show them the LogQL query — it's just as easy to write. The discipline is in the log shipper config (Promtail, OTel filelog), which platform team controls, not application code.
- **CI/CD**: validate Promtail / OTel filelog configs in CI for label cardinality. A linter that flags any label with unbounded values (IDs, emails, hashes). Application log format standardization: enforce JSON output via the language's default logger configuration (pino, slog, structlog) in service templates. Structured logging is a prerequisite; you can't do `| json` on free-form text.
- **Operations**: Loki ring status at `/ring` for cluster health. `loki_ingester_memory_chunks` for chunk pressure. Monthly label audit: `curl .../loki/api/v1/labels` and count values per label for the top streams — any label with >1000 distinct values in a 24h period is a candidate for removal. The `max_streams_per_user` limit should be set explicitly (default is unlimited — a footgun). Set it to 2× your expected stream count so you get 429 warnings before things are truly broken.
