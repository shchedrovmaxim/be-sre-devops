# PromQL essentials — the simple version (the speedometer analogy)

> Read this first. Once the rate-vs-gauge intuition clicks, the [deep-dive](./promql-essentials.md) becomes easy.

This doc explains **one idea**:

> **Counters only go up. PromQL's job is to turn "how high the counter is now" into "how fast it's changing" — because that's what you actually care about.**

---

## Your counter is an odometer, not a speedometer

A car's odometer (total km driven) keeps climbing forever. If you want to know how fast you're going *right now*, you look at the speedometer — which is the odometer's rate of change.

| Odometer / Speedometer | PromQL world |
|---|---|
| Odometer reading (total km) | `http_requests_total` — a counter, only goes up |
| Speedometer (km/h right now) | `rate(http_requests_total[5m])` — requests/second |
| "How many km in the last hour?" | `increase(http_requests_total[1h])` — total increase over a window |
| Your odometer reset to 0 when you rented a new car | Counter reset — Prometheus handles this automatically |

The **5m** in `rate(...[5m])` is the **range vector** — the window of past samples you average over.

---

## The 5 PromQL idioms you need

### 1. Rate of a counter

```promql
rate(http_requests_total[5m])
```

Requests per second, averaged over the last 5 minutes. Use this for dashboards and alerting rules. Never graph a raw counter.

### 2. Aggregate with sum

```promql
sum by (job) (rate(http_requests_total[5m]))
```

Total request rate per `job`, summed across all instances. The `by (job)` keeps the `job` label; all other labels are collapsed.

### 3. p99 latency from a histogram

```promql
histogram_quantile(0.99,
  sum by (le, job) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

The `le` label is the bucket boundary. You must keep `le` in the aggregation or `histogram_quantile()` has nothing to work with. This gives you the 99th percentile latency per `job`.

### 4. Error ratio

```promql
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (job) (rate(http_requests_total[5m]))
```

`status=~"5.."` is a regex match (5xx responses). The `/` divides two instant vectors, resulting in the error fraction.

### 5. Alert when a series disappears

```promql
absent(up{job="my-api"})
```

Returns 1 if no `up{job="my-api"}` series exists. Use this to alert when a service stops exporting metrics entirely.

---

## rate() vs irate() — when to use each

| | `rate()` | `irate()` |
|---|---|---|
| How it works | Average rate across the whole window | Rate using only the last 2 samples |
| Best for | Dashboards, alerts (smooth, less noisy) | Debugging spikes (more responsive) |
| Sensitive to | Low-traffic services (fewer samples → noisy average) | Time gaps between samples |

**Default**: use `rate()`. Switch to `irate()` only when you're debugging and want the most up-to-date reading.

---

## Instant vector vs range vector

This trips people up:

- **Instant vector**: a single value per time series at a moment in time. Example: `http_requests_total` — one number per label set.
- **Range vector**: a window of values over time. Example: `http_requests_total[5m]` — all samples from the last 5 minutes per label set.

Most functions (`rate()`, `sum()`, `avg()`) return instant vectors. Functions like `rate()` take a range vector as input and return an instant vector.

---

## Self-test (out loud)

> **"Walk me through computing a p99 latency from a histogram metric using PromQL."**

**Reference answer:**

"Histograms expose three series: `_bucket` with cumulative counts per bucket boundary (`le` label), `_sum` (total observed values), and `_count` (total observations). To get p99:

```promql
histogram_quantile(0.99,
  sum by (le, job) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

I take the rate of the bucket counters over a 5-minute window — that converts cumulative counts into per-second rates. I sum by `le` and `job` to aggregate across replicas. Then `histogram_quantile(0.99, ...)` interpolates within the bucket containing the 99th percentile. The result is the latency in seconds where 99% of requests are faster.

The key gotcha: you must keep `le` in the `sum by` aggregation, otherwise `histogram_quantile()` can't find the bucket boundaries."

---

## Further reading

- [Prometheus query docs](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [histogram_quantile reference](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)
- [Robust Perception — rate vs irate](https://www.robustperception.io/irate-graphs-are-better-graphs/)

---

## Next: the deep-dive

When rate/histogram/sum feel natural, jump to [`promql-essentials.md`](./promql-essentials.md). The deep-dive covers:

- `topk`, `_over_time` aggregators, `offset`, `without`
- The staleness marker and why 5m is a magic number
- Gotchas: the range vector must be > scrape interval; histogram bucket ordering
- `predict_linear()` for capacity planning
- 4 drills, the 60-second interview answer, and the 4-dimensions framing
