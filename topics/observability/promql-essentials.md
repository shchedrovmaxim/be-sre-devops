# PromQL essentials — the deep-dive

> **Goal**: by the end you can answer — **"Walk me through computing a p99 latency from a histogram metric using PromQL."** — and handle the follow-ups: `rate` vs `irate`, staleness markers, the `without` clause, `topk` for cardinality hunting, and `_over_time` aggregators.

> Start with [promql-essentials-simple.md](./promql-essentials-simple.md) first. The odometer/speedometer framing is the spine.

---

## The senior framing — PromQL as an exploration tool

Mid-level engineers know the PromQL they need to answer questions in existing dashboards. Senior engineers use PromQL as an **exploration tool** — finding unknown failure modes, hunting cardinality offenders, validating SLI math live, and writing recording rules that compress expensive queries into cheap ones.

The difference shows up in incidents: a mid-level engineer has a dashboard that shows something is broken. A senior engineer drops into the Prometheus UI, writes 3 exploratory queries, and identifies the offending service, label, and time of regression in under 5 minutes.

---

## Instant vector vs range vector vs scalar

Understanding these three types is prerequisite to everything else.

| Type | What it is | Example |
|---|---|---|
| **Instant vector** | One value per time series, at a single point in time | `http_requests_total` |
| **Range vector** | A window of samples per time series over a time interval | `http_requests_total[5m]` |
| **Scalar** | A single number, no labels | `1.5`, result of `scalar(up)` |

Most functions return instant vectors. `rate()`, `increase()`, `delta()`, `deriv()` take range vectors and return instant vectors. Aggregations (`sum`, `avg`, `max`, `min`, `count`) take instant vectors and return instant vectors with fewer labels.

---

## Counter idioms

### `rate()` — the workhorse

```promql
rate(http_requests_total[5m])
```

Average per-second rate over the last 5 minutes. Handles counter resets automatically (if the counter drops, Prometheus detects a reset and adjusts). The 5-minute window smooths noise; increase it for low-traffic services.

**Gotcha**: the range window must be at least 2× the scrape interval. With `scrape_interval: 30s`, a `[1m]` window might have only 2 samples — producing noisy rates. Use `[2m]` or `[5m]`.

### `irate()` — for spike detection

```promql
irate(http_requests_total[5m])
```

Uses only the last two samples in the window to compute the instantaneous rate. Much more responsive than `rate()` but also much noisier. The `[5m]` window is only used for staleness purposes — it means "if there are no samples in the last 5 minutes, treat as stale."

**When to use**: debugging only. Never alert on `irate()` — it's too noisy. Never use it for recording rules that feed dashboards.

### `increase()` — total increase over a window

```promql
increase(http_requests_total[1h])
```

Total count increase over the window, not rate. Useful for "how many requests did we get in the last hour?" Use `rate()` × duration instead in alerting (it aggregates better).

---

## Aggregation operators

### `sum by` and `sum without`

```promql
# Keep only job label — sum everything else away
sum by (job) (rate(http_requests_total[5m]))

# Drop the instance label — keep everything else
sum without (instance) (rate(http_requests_total[5m]))
```

`by` is an allowlist of labels to keep. `without` is a denylist. For high-cardinality label sets, `without (instance)` is cleaner than listing every label you want to keep.

### `topk` — finding noisy series

```promql
# Top 10 metric families by series count (cardinality hunting)
topk(10, count by (__name__) ({__name__=~".+"}))

# Top 5 jobs by request rate
topk(5, sum by (job) (rate(http_requests_total[5m])))
```

`topk(N, expr)` returns the N highest-value series. Invaluable for cardinality investigation and for finding the noisiest services in an incident.

### `bottomk`, `max`, `min`, `avg`, `count`, `stddev`

```promql
# Average p99 latency across all jobs
avg by (job) (
  histogram_quantile(0.99,
    sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
  )
)

# Count of distinct services reporting
count(count by (job) (up{job=~"my-.*"}))
```

---

## Histogram quantiles — the full picture

A histogram metric exposes three series per unique label set:

| Series | What it contains |
|---|---|
| `http_request_duration_seconds_bucket{le="0.05"}` | Count of requests completed in ≤0.05s |
| `http_request_duration_seconds_bucket{le="0.1"}` | Count of requests completed in ≤0.1s |
| `http_request_duration_seconds_bucket{le="+Inf"}` | All requests (equals `_count`) |
| `http_request_duration_seconds_sum` | Sum of all observed durations |
| `http_request_duration_seconds_count` | Total number of observations |

The bucket counts are **cumulative** — each `le` bucket includes all requests faster than that boundary.

### The correct p99 query

```promql
histogram_quantile(
  0.99,
  sum by (le, job) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Step by step:
1. `rate(http_request_duration_seconds_bucket[5m])` — converts cumulative counts to per-second arrival rates per bucket
2. `sum by (le, job)` — aggregates across instances, keeping `le` (bucket boundary) and `job` labels
3. `histogram_quantile(0.99, ...)` — interpolates within the bucket containing the 99th percentile

**Critical gotcha — you must keep `le` in the aggregation.** If you write `sum by (job)`, the `le` label is dropped and `histogram_quantile()` has no bucket boundaries to work with. It returns NaN.

**Bucket boundary gotcha**: `histogram_quantile()` interpolates linearly within a bucket. If your p99 lands in the `le="+Inf"` bucket (i.e., the request was slower than your highest explicit bucket), the result is `+Inf`. This means your histogram buckets aren't calibrated to your actual latency distribution. You need a higher bucket boundary.

### Computing average latency from a histogram

```promql
sum by (job) (rate(http_request_duration_seconds_sum[5m]))
/
sum by (job) (rate(http_request_duration_seconds_count[5m]))
```

This is the true mean latency per job. Note that the mean is influenced heavily by outliers — p99 is more useful for SLOs.

---

## Label filtering and regex

```promql
# Exact match
http_requests_total{job="checkout-api"}

# Regex match (=~)
http_requests_total{status=~"5.."}          # all 5xx
http_requests_total{method=~"GET|POST"}     # GET or POST

# Negative regex (!~)
http_requests_total{status!~"2.."}          # non-2xx

# Negative exact (!= )
http_requests_total{env!="staging"}
```

Regex anchoring: PromQL regexes are fully anchored. `status=~"5.."` matches exactly three characters: a 5 followed by any two characters. To match "starts with 5", use `"5.*"`.

---

## The `_over_time` functions

These work on **gauge** metrics (not counters) to compute aggregations over a time range:

```promql
# Max memory usage in the last 1 hour
max_over_time(container_memory_working_set_bytes[1h])

# Average CPU over 24 hours
avg_over_time(container_cpu_usage_seconds_total[24h])

# Minimum replicas seen in 30 minutes (was there ever a scale-down?)
min_over_time(kube_deployment_status_replicas_available[30m])
```

These are useful for capacity planning and for "did the service ever fall below N replicas?"

---

## `predict_linear()` — capacity planning

```promql
# Will disk fill up in the next 4 hours?
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
```

`predict_linear(range_vector, seconds)` fits a linear regression to the range vector and predicts the value that many seconds into the future. Alert on `< 0` for "disk will fill in 4 hours."

---

## `absent()` and `absent_over_time()`

```promql
# Alert if no up{job="my-api"} series exists
absent(up{job="my-api"})

# Alert if the series existed but has had no data for 10 minutes
absent_over_time(up{job="my-api"}[10m])
```

`absent()` catches "service never started / stopped emitting". `absent_over_time()` catches "service was scraping fine but disappeared in the last N minutes."

---

## Staleness markers

When a target disappears (pod killed, relabeling changes), Prometheus writes a **staleness marker** — a special NaN value — to all time series that target was producing. Queries treat staleness-marked series as if they don't exist.

The default staleness period is **5 minutes**. If a scrape fails, Prometheus waits 5 minutes before marking the series stale. This is why `rate(...[5m])` is the minimum meaningful window for many metrics — you need the window to be at least as long as the staleness period to avoid gaps.

This is also why `absent(up{job="my-api"})` uses a `for: 2m` guard — the first 5 minutes of an outage the series might still be present but stale.

---

## The `offset` modifier

```promql
# Compare current rate to 1 week ago (same time, same day)
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 1w)
```

`offset` shifts the time of evaluation back. Useful for week-over-week comparisons, anomaly detection, and capacity planning.

---

## PromQL for cardinality investigation

```promql
# Count total active series in TSDB
prometheus_tsdb_head_series

# Top 10 metric families by series count
topk(10,
  count by (__name__) ({__name__=~".+"})
)

# Find the label with the most distinct values for a given metric
count by (user_id) (http_requests_total)   # if this returns 100K+, you have a cardinality problem
```

This is the investigation pattern when Prometheus is OOMing — see [`cardinality-discipline.md`](./cardinality-discipline.md) for the full workflow.

---

## The 60-second interview answer

> "To compute p99 latency from a histogram:
>
> ```promql
> histogram_quantile(0.99,
>   sum by (le, job) (
>     rate(http_request_duration_seconds_bucket[5m])
>   )
> )
> ```
>
> The histogram exposes cumulative bucket counts (the `_bucket` series with an `le` label). I take the `rate()` over 5 minutes to convert cumulative counts to per-second rates, then `sum by (le, job)` to aggregate across replicas. The critical detail is keeping `le` in the aggregation — without it, `histogram_quantile()` has no bucket boundaries and returns NaN. `histogram_quantile(0.99, ...)` then interpolates within the bucket containing the 99th percentile.
>
> On `rate` vs `irate`: `rate()` averages across the window and is the right choice for dashboards and alerting. `irate()` uses only the last two samples — more responsive for incident debugging, too noisy for alerts.
>
> On instant vs range vectors: most PromQL functions return instant vectors (one value per series per moment). `rate()` takes a range vector (a window of samples) and returns an instant vector. Understanding this distinction is what makes you able to compose arbitrary PromQL without guessing."

---

## Self-test drills

### Drill 1 — histogram quantile

Write the PromQL for p95 latency of the `checkout-api` job, bucketed by status code.

**Answer:**
```promql
histogram_quantile(0.95,
  sum by (le, status) (
    rate(http_request_duration_seconds_bucket{job="checkout-api"}[5m])
  )
)
```

### Drill 2 — error ratio

Write a query for the fraction of 5xx requests out of all non-4xx requests for the `checkout-api` job.

**Answer:**
```promql
sum(rate(http_requests_total{job="checkout-api", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="checkout-api", status!~"4.."}[5m]))
```

### Drill 3 — topk investigation

During an incident, OOMKilled pods suggest high memory. Write a PromQL to find the top 5 pods by memory usage in the `production` namespace.

**Answer:**
```promql
topk(5,
  container_memory_working_set_bytes{
    namespace="production",
    container!=""
  }
)
```

### Drill 4 — rate gotcha

A team sets `scrape_interval: 60s` and writes `rate(requests_total[30s])`. What's the problem?

**Answer:** The range window `[30s]` is shorter than the scrape interval `60s`. With a 60-second scrape interval, Prometheus may only have 0 or 1 sample in a 30-second window — not enough to compute a meaningful rate. The rule is: range window must be at least 2× the scrape interval. With `60s` scrape interval, use `[2m]` or `[5m]`.

---

## Further reading

- [Prometheus querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Prometheus query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus query operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [histogram_quantile gotchas — Robust Perception](https://www.robustperception.io/how-does-a-prometheus-histogram-work/)
- [PromQL for humans — Grafana blog](https://grafana.com/blog/2020/02/04/introduction-to-promql-the-prometheus-query-language/)

---

## The 4 dimensions

- **Tech**: `rate()` for counters, `histogram_quantile(0.99, sum by (le,...) rate(_bucket[5m]))` for latency, `absent()` for missing services, `topk(10, count by (__name__)(...))` for cardinality hunting. Range window must be ≥2× scrape interval. Keep `le` in histogram aggregations.
- **People**: PromQL is the shared language for SRE + app teams. The faster everyone can write basic queries, the less "only the Prometheus expert can debug this." Invest in a shared runbook of the 5 most common queries for your services. Grafana's Explore mode is the lowest-friction on-ramp.
- **CI/CD**: recording rules generated from templates (Sloth, Pyrra) rather than hand-written. `promtool check rules` in CI. Dashboard PromQL tested against known sample data using Grafana's `mimirtool analyze` or by running `promtool query` in test environments.
- **Operations**: the exploration workflow — `prometheus_tsdb_head_series` for health, `topk(10, count by __name__)` for cardinality spikes, `rate(prometheus_target_scrape_pool_targets)` for SD changes. On-call should be comfortable with 3-4 exploratory queries, not just reading dashboards. Stale series after pod churn are normal — don't alert on short blips.
