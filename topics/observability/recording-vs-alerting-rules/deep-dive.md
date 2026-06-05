# Recording rules vs alerting rules — the deep-dive

> **Goal**: by the end you can answer — **"When would you use a recording rule vs an alerting rule, and what's the performance benefit?"** — covering the pre-computation win, naming conventions, the SLO burn-rate pattern, reload mechanics, and the Prometheus Operator abstraction.

> Start with [recording-vs-alerting-rules-simple.md](./simple.md) first. The sous-chef analogy is the spine.

---

## The senior framing — rules as a correctness and performance contract

Mid-level engineers think of recording rules as an optimization. Senior engineers know they're also a **correctness contract**: if your alerting rule computes a rate over 1 hour inline, every evaluation re-scans the raw samples. If data is dropped mid-window (scrape failure, remote-write lag), the inline query might produce a misleading result. A recording rule written to a new series at a known interval gives you a **stable, auditable intermediate representation** you can inspect, backfill, and reason about.

The performance angle is real too. A `rate(high_cardinality_counter[5m])` summed by job across 500 instances scans every sample in a 5-minute window for every series. At 30-second scrape intervals that's 10 samples × 500 instances × however many label combinations. For a 5-instance service it's trivial. At cluster scale it's 100K+ samples per query, and if 10 dashboards are open and refreshing every 30 seconds, you're doing 20 scans/second of that data.

The recording rule does that scan **once per interval**, writes one float64 per result series, and every dashboard reads those tiny pre-computed series.

---

## Recording rules — the full anatomy

```yaml
groups:
  - name: slo.recording.rules
    interval: 60s           # how often to evaluate; defaults to global evaluation_interval
    rules:
      - record: job:http_error_ratio:rate5m
        expr: |
          sum by (job) (
            rate(http_requests_total{status=~"5.."}[5m])
          )
          /
          sum by (job) (
            rate(http_requests_total[5m])
          )
        labels:
          # optional extra labels attached to the resulting series
          slo: checkout-api
```

The `record:` field becomes the **metric name** of the new time series. The `expr:` result is stored with that name at each evaluation. Dashboards and alerting rules query `job:http_error_ratio:rate5m` directly.

### Naming convention — `level:metric:operations`

The canonical convention (from the Prometheus docs):

```
level:metric:operations
```

| Segment | Meaning | Example |
|---|---|---|
| `level` | The label set you aggregated to | `job`, `instance`, `cluster`, `svc` |
| `metric` | What you measured | `http_error_ratio`, `request_rate`, `cpu_utilization` |
| `operations` | Transformation applied | `rate5m`, `avg1h`, `irate1m` |

Examples:
- `instance:cpu_utilization:avg5m` — per-instance average CPU, 5-minute window
- `job:http_error_ratio:rate5m` — per-job error ratio, 5-minute rate
- `cluster:http_request_rate:rate1m` — cluster-wide request rate, 1-minute rate

This convention allows building **multi-tier hierarchies**:

```
# Tier 1: per-instance (from raw counters)
- record: instance:http_requests:rate5m
  expr: rate(http_requests_total[5m])

# Tier 2: per-job (from tier-1 recording rule)
- record: job:http_requests:rate5m
  expr: sum by (job) (instance:http_requests:rate5m)

# Tier 3: cluster-wide (from tier-2)
- record: cluster:http_requests:rate5m
  expr: sum(job:http_requests:rate5m)
```

Each tier aggregates the tier below. Prometheus only runs the raw counter scan once (for tier-1). Everything above that reads pre-computed series.

---

## Alerting rules — the full anatomy

```yaml
groups:
  - name: slo.alerting.rules
    rules:
      - alert: CheckoutHighErrorBurnRate
        expr: |
          job:http_error_ratio:rate5m{job="checkout-api"} > 0.01
        for: 5m
        labels:
          severity: page
          team: checkout
        annotations:
          summary: "{{ $labels.job }} error ratio above 1%"
          description: |
            Current error ratio: {{ $value | humanizePercentage }}
            Runbook: https://wiki.internal/runbooks/checkout-high-error
          dashboard: "https://grafana.internal/d/abc123"
```

### The `for:` duration — why it matters

`for: 5m` means the expression must be continuously true for 5 minutes before the alert fires. During those 5 minutes the alert is in the **PENDING** state. After 5 minutes it transitions to **FIRING** and is sent to Alertmanager.

Without `for:`, a single-evaluation spike (30 seconds) pages someone. That's alert fatigue. The right `for:` value is a function of:
- How long the condition must persist to be a real problem (not a blip)
- How much delay you can tolerate in getting paged

Common values: `for: 1m` (fast, low tolerance), `for: 5m` (default for most pages), `for: 15m` (for slow degradation tickets).

### The `absent()` pattern

For alerting when a service disappears entirely:

```yaml
- alert: ServiceMissing
  expr: absent(up{job="checkout-api"})
  for: 2m
  labels:
    severity: page
  annotations:
    summary: "checkout-api has disappeared from Prometheus scraping"
```

`absent()` returns a single sample with value 1 if the selector matches nothing. This catches "service stopped exporting metrics" — which can happen without any 5xx error (e.g., the pod is just gone).

---

## The SLO burn-rate pattern

This is where recording rules and alerting rules work together most powerfully. A full multi-window multi-burn-rate alert stack for a 30-day SLO at 99.9% availability:

```yaml
groups:
  - name: slo.recording
    interval: 30s
    rules:
      # Error ratio over various windows
      - record: job:http_error_ratio:rate5m
        expr: |
          sum by (job)(rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job)(rate(http_requests_total[5m]))

      - record: job:http_error_ratio:rate1h
        expr: |
          sum by (job)(rate(http_requests_total{status=~"5.."}[1h]))
          / sum by (job)(rate(http_requests_total[1h]))

      - record: job:http_error_ratio:rate6h
        expr: |
          sum by (job)(rate(http_requests_total{status=~"5.."}[6h]))
          / sum by (job)(rate(http_requests_total[6h]))

      - record: job:http_error_ratio:rate1d
        expr: |
          sum by (job)(rate(http_requests_total{status=~"5.."}[1d]))
          / sum by (job)(rate(http_requests_total[1d]))

  - name: slo.alerting
    rules:
      # SLO: 99.9% availability. Error budget = 0.1%. 
      # Burn rate thresholds derived from 30d window.

      # 14.4x burn = exhaust budget in 2 days. Page fast.
      - alert: SLOBudgetBurnHigh
        expr: |
          job:http_error_ratio:rate5m{job="checkout-api"}  > (14.4 * 0.001)
          and
          job:http_error_ratio:rate1h{job="checkout-api"}  > (14.4 * 0.001)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "checkout-api burning SLO budget at 14.4x rate"

      # 6x burn = exhaust budget in 5 days.
      - alert: SLOBudgetBurnMedium
        expr: |
          job:http_error_ratio:rate1h{job="checkout-api"}  > (6 * 0.001)
          and
          job:http_error_ratio:rate6h{job="checkout-api"} > (6 * 0.001)
        for: 15m
        labels:
          severity: page

      # 3x burn = exhaust budget in 10 days. Ticket.
      - alert: SLOBudgetBurnLow
        expr: |
          job:http_error_ratio:rate6h{job="checkout-api"} > (3 * 0.001)
          and
          job:http_error_ratio:rate1d{job="checkout-api"} > (3 * 0.001)
        for: 1h
        labels:
          severity: ticket
```

The `and` condition is the two-window guard. Both windows must exceed the threshold, preventing single-spike pages. The recording rules pre-compute each window — the alerting rules just compare two floats.

> **Shortcut**: use [Sloth](https://github.com/slok/sloth) or [Pyrra](https://github.com/pyrra-dev/pyrra) to generate all of this from a simple SLO spec. Don't hand-write it; the math is error-prone.

---

## Reload mechanics

### Raw Prometheus

Prometheus watches for rule file changes. You can trigger a reload two ways:

```bash
# Send SIGHUP
kill -HUP <prometheus-pid>

# Or POST to the lifecycle API (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload
```

### Prometheus Operator

You create a `PrometheusRule` CRD:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-rules
  labels:
    release: kube-prometheus-stack   # must match Prometheus CR's ruleSelector
spec:
  groups:
    - name: slo.recording
      rules:
        - record: job:http_error_ratio:rate5m
          expr: |
            sum by (job)(rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job)(rate(http_requests_total[5m]))
```

The operator watches PrometheusRule objects and automatically reconciles the Prometheus config. No manual reload needed.

### Linting in CI

Always validate rules before applying:

```bash
promtool check rules rules/*.yml
```

This catches: invalid PromQL syntax, duplicate rule names, missing `record:` fields. Add it to your CI pipeline before any merge to main.

---

## The 60-second interview answer

> "Recording rules pre-compute expensive PromQL on a schedule and store the result as a new time series. The performance benefit is concrete: a `rate()` over a high-cardinality counter scans every sample in the window. If 10 dashboards each query that every 30 seconds, you're doing 20 full scans per second. The recording rule runs the scan once per minute and stores one float per result series — dashboards read that instantly.
>
> Alerting rules fire when an expression is true for the `for:` duration. Without `for:`, brief spikes cause false pages. The standard SLO pattern combines both: recording rules compute burn rates at multiple windows (5m, 1h, 6h, 24h), alerting rules fire when two windows both exceed a threshold simultaneously — that dual-window guard separates real incidents from noise.
>
> In Kubernetes with the Prometheus Operator, rules are PrometheusRule CRDs. They live in Git, go through PR review, and the operator reconciles config automatically. Linting with `promtool check rules` runs in CI."

---

## Self-test drills

### Drill 1 — naming

You're computing the p99 latency per job, averaged over 10 minutes. What's the correct recording rule name?

**Answer:** `job:http_request_duration_seconds_p99:avg10m`

Level: `job`. Metric: `http_request_duration_seconds_p99`. Operations: `avg10m`.

### Drill 2 — for: duration

An alerting rule fires on `job:http_error_ratio:rate5m > 0.05` with no `for:` clause. What's the problem?

**Answer:** Without `for:`, a single evaluation where the expression is true (a 30-second spike, a scrape hiccup) will immediately page someone. That's alert fatigue. Adding `for: 5m` requires the condition to be sustained for 5 minutes before firing — which eliminates the vast majority of transient spikes while still catching real incidents.

### Drill 3 — SLO burn rate

Your SLO is 99.9% over 30 days (error budget = 0.1%). At what error ratio does a 14.4× burn rate alert fire?

**Answer:** `14.4 × 0.001 = 0.0144` — i.e., 1.44% error ratio. At that rate you'd exhaust the 30-day budget in `30d / 14.4 ≈ 2 days`. That's the justification for paging urgently.

### Drill 4 — absent()

Why use `absent(up{job="my-service"})` instead of `up{job="my-service"} == 0`?

**Answer:** `up{job="my-service"} == 0` only fires if Prometheus is actively scraping the target and it returns unhealthy. If the pod disappears entirely (deleted, namespace gone, NetworkPolicy blocking), there are no `up` time series at all — the selector matches nothing and `== 0` never fires. `absent()` returns 1 specifically when the selector matches nothing, catching the "target vanished" case that `== 0` misses.

---

## Further reading

- [Prometheus recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Sloth](https://github.com/slok/sloth) — SLO-to-recording+alerting-rules generator
- [Pyrra](https://github.com/pyrra-dev/pyrra) — Kubernetes-native SLO management with UI
- [Google SRE Workbook Chapter 5](https://sre.google/workbook/alerting-on-slos/) — the canonical multi-window multi-burn-rate reference

---

## The 4 dimensions

- **Tech**: recording rules pre-compute expensive expressions; alerting rules fire on pre-computed or raw expressions with a sustained duration guard. The `level:metric:operations` naming enables multi-tier aggregation hierarchies. The SLO burn-rate pattern (recording rules + dual-window alerting rules) is the production standard.
- **People**: PrometheusRule CRDs mean app teams write rules in their own Helm charts. Platform team owns the Prometheus CR and the `ruleSelector` that picks them up. New engineers onboard to "add a PrometheusRule to your Helm chart." Runbooks and dashboard links go in alert annotations — not in someone's head.
- **CI/CD**: `promtool check rules` in CI catches syntax errors before deploy. Sloth/Pyrra generate SLO rules from a spec — PR review is on the spec, not hand-computed thresholds. Alert routing config (which team gets which severity) goes through Alertmanager CR PRs, not click-ops in the Alertmanager UI.
- **Operations**: PrometheusRule resources appear in `kubectl get prometheusrules` — you can see every rule in the cluster. `ALERTS` and `ALERTS_FOR_STATE` are built-in metrics that show what's pending or firing — use them in dashboards. Alert annotation runbooks must be kept current; a runbook link that 404s is worse than no link (it wastes the first 3 minutes of an incident).
