# Recording rules vs alerting rules — the simple version (the sous chef analogy)

> Read this first. Once the prep-work analogy clicks, the [deep-dive](./recording-vs-alerting-rules.md) becomes easy.

This doc explains **one idea**:

> **Recording rules pre-cook expensive PromQL so dashboards and alerts don't have to cook it fresh every time. Alerting rules watch the result and fire when a threshold is crossed.**

---

## Your monitoring is a restaurant kitchen

Imagine a chef who has to compute "what's the p99 latency for every service in the cluster" every time a customer asks. That query hits millions of data points, does a histogram_quantile, and takes 15 seconds. The customer waits 15 seconds for every dashboard refresh.

A smarter kitchen has a **sous chef** who runs in the background every minute, pre-computes that query, and writes the result to a small cheat sheet. When a customer asks, the head chef reads the cheat sheet — instant.

| Kitchen analogy | Prometheus world |
|---|---|
| Sous chef runs every 60s | **Recording rule** — evaluates a PromQL expression on a schedule |
| Cheat sheet of pre-computed results | **New time series** written by the recording rule |
| Head chef reads cheat sheet for a customer | Dashboard or alert queries the pre-computed series |
| Head chef calls 911 when fridge temp > 8°C | **Alerting rule** — fires when an expression exceeds a threshold |
| Calling 911 only if fridge stays hot for 5 min | The `for: 5m` duration guard on alerting rules |

---

## The two rule types in one sentence each

**Recording rule**: "Every 60 seconds, run this expensive PromQL and store the result as a new metric name."

**Alerting rule**: "If this expression is true for 5 minutes straight, send an alert to Alertmanager."

They often work together: a recording rule computes an SLO burn rate, and an alerting rule fires when that burn rate exceeds a threshold.

---

## A concrete example

You want to know the error ratio per service. The raw query is expensive (it scans millions of counter samples):

```promql
# Expensive — runs on every dashboard load
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

You turn this into a recording rule:

```yaml
groups:
  - name: slo.rules
    interval: 60s
    rules:
      - record: job:http_error_ratio:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
```

Now the dashboard queries `job:http_error_ratio:rate5m` — one small series per service, instant.

And the alerting rule fires on it:

```yaml
      - alert: HighErrorRatio
        expr: job:http_error_ratio:rate5m > 0.01
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "Error ratio above 1% for {{ $labels.job }}"
```

---

## The naming convention — why it looks weird

Recording rule names follow `level:metric:operations`:

- `job:http_error_ratio:rate5m`
  - **level** = `job` (the label set we aggregated by)
  - **metric** = `http_error_ratio` (what we computed)
  - **operations** = `rate5m` (how we computed it)

This isn't decorative — it tells you exactly how to compose rules. A rule at the `cluster` level was probably built from `job`-level rules, which were built from `instance`-level raw metrics.

---

## When to use which

| Situation | Use |
|---|---|
| Dashboard query takes >1s | Recording rule |
| The same expensive expression used in multiple alerts | Recording rule |
| Computing an SLO burn rate | Recording rule |
| "Alert me when error rate > threshold" | Alerting rule |
| "Alert me when a service goes missing" | Alerting rule with `absent()` |
| Computing a ratio for visual display only, cheap query | Just PromQL directly (no rule needed) |

---

## Self-test (out loud)

> **"When would you use a recording rule vs an alerting rule, and what's the performance benefit?"**

**Reference answer:**

"A recording rule pre-computes an expensive PromQL expression on a schedule and writes the result as a new metric. The main benefit is dashboard performance — instead of a query scanning millions of counter samples on every browser refresh, the dashboard reads a single small pre-computed series. I use recording rules for any query that's expensive (rate over high-cardinality counters), reused in multiple alerts, or used to compute SLO burn rates.

Alerting rules watch an expression — typically either raw metrics or a pre-computed recording-rule series — and fire to Alertmanager when it's true for the `for:` duration. The `for:` guard is critical: without it, a 10-second spike pages someone unnecessarily. With `for: 5m`, the condition must be sustained.

The standard SLO pattern: a recording rule computes the burn rate, and an alerting rule fires when that burn rate exceeds the multi-window thresholds."

---

## Further reading

- Prometheus [recording rules docs](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- Prometheus [alerting rules docs](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Sloth](https://github.com/slok/sloth) — generates recording + alerting rules from a simple SLO YAML (see how it uses recording rules to compute burn rates)

---

## Next: the deep-dive

When the sous-chef analogy is obvious, jump to [`recording-vs-alerting-rules.md`](./recording-vs-alerting-rules.md). The deep-dive covers:

- The performance math (how much recording rules actually help)
- The reload mechanics (SIGHUP, /-/reload, Prometheus Operator PrometheusRule CRD)
- The naming convention in full
- How recording rules compose into multi-tier hierarchies
- The SLO burn-rate recording pattern in detail
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
