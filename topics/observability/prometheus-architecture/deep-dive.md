# Prometheus architecture + scrape config + relabeling — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through what Prometheus does when it scrapes a target — including how relabeling works."** — covering service discovery, the two relabeling phases, the 4 metric types, Kubernetes SD patterns, and how Prometheus Operator's ServiceMonitor abstracts the raw config.

> Start with [prometheus-architecture-simple.md](./simple.md) first if you haven't. The newspaper-reporter analogy is the spine.

---

## The senior framing — pull model as an operational choice

Mid-level engineers see Prometheus as "the metrics tool." Senior engineers see it as **an architectural choice about control and observability contracts**.

The pull model puts Prometheus in control: it decides when to scrape, how often, and what to keep. This means:
- **Targets are stateless** about where their metrics go. They expose `/metrics` and don't care who reads it.
- **Prometheus decides health**, not the target. If a scrape fails, Prometheus records `up=0`. The target doesn't get to declare itself healthy.
- **Configuration lives centrally**. You can change what you scrape without touching the application.

The flip side: pull model needs network access to every target. In multi-cluster or cross-VPC setups, this gets complicated — which is part of why Thanos and Victoria Metrics support remote-write (push-compatible) as a second channel.

---

## The scrape pipeline — every step

```
Kubernetes API
    │
    ▼
Service Discovery  ─── discovers pods/services with __meta_kubernetes_* labels
    │
    ▼
relabel_configs    ─── filter + rewrite the TARGET (address, port, job, instance)
    │
    ▼
HTTP GET /metrics  ─── Prometheus fetches the exposition text
    │
    ▼
metric_relabel_configs ─── filter + rewrite the METRICS (drop labels, rename)
    │
    ▼
TSDB storage       ─── each unique {name + label set} → one time series
```

### Step 1: Service discovery

Prometheus supports many SD mechanisms. In Kubernetes you'll use `kubernetes_sd_configs`:

```yaml
scrape_configs:
  - job_name: 'pods'
    kubernetes_sd_configs:
      - role: pod          # also: node, service, endpoints, endpointslice
    relabel_configs:
      # ... see below
```

The SD mechanism fills in `__meta_kubernetes_*` labels for each discovered object. For a pod:

```
__meta_kubernetes_pod_name="my-api-6f4b8d-xkpzq"
__meta_kubernetes_pod_namespace="production"
__meta_kubernetes_pod_label_app="my-api"
__meta_kubernetes_pod_annotation_prometheus_io_scrape="true"
__meta_kubernetes_pod_container_port_name="metrics"
__meta_kubernetes_pod_container_port_number="9090"
```

These are *temporary* labels used only in the relabeling phase — they don't end up in the stored time series unless you explicitly promote them.

### Step 2: `relabel_configs` — rewriting the target

This phase runs before Prometheus touches the network. Typical use cases:

**Keep only pods that opt in via annotation:**
```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: "true"
```

**Set the scrape port from a pod annotation:**
```yaml
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: (.+)
    replacement: "${__meta_kubernetes_pod_ip}:$1"
```

**Set the scrape path from an annotation:**
```yaml
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
```

**Promote pod labels to metric labels:**
```yaml
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
```

**Drop targets from a specific namespace:**
```yaml
  - source_labels: [__meta_kubernetes_pod_namespace]
    action: drop
    regex: "kube-system"
```

### Step 3: Metric exposition format

The target's `/metrics` endpoint responds with plain text (OpenMetrics or the classic Prometheus format):

```
# HELP http_requests_total Total HTTP requests received
# TYPE http_requests_total counter
http_requests_total{handler="/api/v1/users",method="GET",status="200"} 84291
http_requests_total{handler="/api/v1/users",method="GET",status="500"} 17
http_requests_total{handler="/healthz",method="GET",status="200"} 3122093

# HELP http_request_duration_seconds Request duration histogram
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{handler="/api/v1/users",le="0.05"} 71204
http_request_duration_seconds_bucket{handler="/api/v1/users",le="0.1"}  79801
http_request_duration_seconds_bucket{handler="/api/v1/users",le="0.5"}  84231
http_request_duration_seconds_bucket{handler="/api/v1/users",le="1"}    84289
http_request_duration_seconds_bucket{handler="/api/v1/users",le="+Inf"} 84291
http_request_duration_seconds_sum{handler="/api/v1/users"} 5412.7
http_request_duration_seconds_count{handler="/api/v1/users"} 84291
```

### Step 4: `metric_relabel_configs` — rewriting the metrics

This phase runs after the scrape, on each metric line. Typical use cases:

**Drop a noisy high-cardinality metric entirely:**
```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    action: drop
    regex: "go_gc_.*"
```

**Drop a specific label that's too high cardinality:**
```yaml
  - action: labeldrop
    regex: "user_id"
```

**Keep only specific metrics from a verbose exporter:**
```yaml
  - source_labels: [__name__]
    action: keep
    regex: "nginx_(http_requests_total|connections_active|up)"
```

**Rename a label to match your conventions:**
```yaml
  - source_labels: [exported_job]
    target_label: job
    action: replace
```

### Step 5: TSDB storage

Each unique `{metric_name + all label key-value pairs}` becomes one **time series** with a numeric ID. Prometheus appends samples (timestamp + float64 value) to that series in the WAL (write-ahead log) before compacting to 2-hour chunks. Local retention defaults to 15 days.

---

## The 7 relabeling actions

| Action | What it does |
|---|---|
| `replace` | Captures a regex match from `source_labels`, writes the replacement to `target_label`. The default action. |
| `keep` | Keep only targets/metrics where the regex matches. Drops everything else. |
| `drop` | Drop targets/metrics where the regex matches. Keeps everything else. |
| `labelmap` | Copies labels matching a regex to new labels, renaming them. Used to promote `__meta_*` labels. |
| `labeldrop` | Removes labels whose names match a regex from the metric. |
| `labelkeep` | Removes all labels whose names do NOT match a regex. |
| `hashmod` | Computes hash of a label value and assigns to target_label. Used for sharding scrapes across multiple Prometheus instances. |

---

## The 4 metric types — senior nuances

### Counter

- Monotonically increasing. Only resets to 0 on process restart.
- Always use `rate()` or `increase()` to make it useful. Never graph a raw counter.
- Naming convention: suffix `_total`.

```promql
rate(http_requests_total{status=~"5.."}[5m])
```

### Gauge

- Any value, any direction. Memory usage, queue depth, active connections.
- Can be graphed directly. No `rate()` needed.

```promql
container_memory_working_set_bytes{pod=~"my-api-.*"}
```

### Histogram

- Tracks distribution of values in configurable buckets.
- Exposes three series per `{label set}`: `_bucket` (cumulative counts per bucket), `_sum`, `_count`.
- Buckets are cumulative and must include `+Inf`.
- **Aggregate across replicas correctly** — use `histogram_quantile()` on the bucket series.

```promql
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

### Summary

- Pre-computes quantiles **in the application process**, not in Prometheus.
- Cannot be aggregated across replicas (each replica computes its own quantile).
- **Senior rule**: prefer histograms unless you have good reason to use summaries (very fine-grained quantile accuracy requirements, can't choose buckets in advance).

---

## Kubernetes SD patterns — the patterns you'll actually use

### Pattern 1: Annotation-based opt-in (most common for simple cases)

Applications add annotations; a single catch-all scrape config reads them:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_ip,
                        __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: (.+);(.+)
        replacement: "$1:$2"
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_pod_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### Pattern 2: ServiceMonitor (Prometheus Operator — the production standard)

The Prometheus Operator watches for `ServiceMonitor` CRDs and auto-generates the equivalent scrape configs. You never write raw `relabel_configs` for Kubernetes targets — you write ServiceMonitors:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-api
  namespace: production
  labels:
    release: kube-prometheus-stack   # must match Prometheus CR's serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: my-api
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
      metricRelabelings:
        - source_labels: [__name__]
          action: drop
          regex: "go_.*"
```

`PodMonitor` is the pod-level equivalent when there's no Service in front.

---

## Prometheus Operator — what it does for you

| Raw Prometheus | Prometheus Operator |
|---|---|
| Edit `prometheus.yml` + send SIGHUP or POST /-/reload | Create/update a `ServiceMonitor` CRD |
| Manage Prometheus as a StatefulSet manually | Declare a `Prometheus` CR; the operator manages the StatefulSet |
| Write recording rules in `rules.yml` | Create a `PrometheusRule` CRD |
| Configure Alertmanager manually | Create an `Alertmanager` CR + `AlertmanagerConfig` CRDs |

The operator watches CRDs and reconciles the running Prometheus config. It handles config reloads, rolling restarts, and shard scaling.

---

## The 60-second interview answer

> "Prometheus is pull-based: it maintains a list of targets from service discovery — in Kubernetes that's the `kubernetes_sd_config` watching pods, services, or endpoints. Each scrape cycle it applies `relabel_configs` to filter and rewrite the target list — keeping only pods with the right annotation, setting the scrape address and path, promoting pod labels to metric labels. Then it fetches `/metrics`, which responds in the Prometheus exposition format. After the scrape, `metric_relabel_configs` runs on each metric line — typically to drop high-cardinality labels or noisy metric families. The surviving samples go into the TSDB as time series, each uniquely identified by the metric name plus its label set.
>
> In Kubernetes, we almost never write raw `relabel_configs` — we run the Prometheus Operator and write `ServiceMonitor` CRDs. The operator translates ServiceMonitors into scrape configs and handles reloads. The same config pattern — ServiceMonitor selects a Service, the Service selects pods — means adding a new service to Prometheus is one YAML file, no manual config editing."

---

## Self-test drills

### Drill 1 — relabeling

You have a pod annotation `prometheus.io/scrape: "true"` and `prometheus.io/port: "9091"`. The pod IP is `10.0.1.5`. Write the relabeling stanza that sets `__address__` to `10.0.1.5:9091`.

**Answer:**
```yaml
- source_labels: [__meta_kubernetes_pod_ip,
                  __meta_kubernetes_pod_annotation_prometheus_io_port]
  action: replace
  regex: (.+);(.+)
  replacement: "$1:$2"
  target_label: __address__
```

### Drill 2 — metric type selection

Your team is building a latency SLI. Should you use a histogram or a summary? Why?

**Answer:** Histogram. Summaries compute quantiles in the application process per replica — you can't aggregate across replicas (the p99 of two replicas can't be derived from their individual p99s). Histograms store bucket counts that sum correctly, so `histogram_quantile()` over the aggregated bucket series gives you the true population quantile across all replicas. The trade-off is that you must define bucket boundaries upfront — they should match your SLO thresholds (e.g., 50ms, 100ms, 200ms, 500ms, 1s, +Inf).

### Drill 3 — Prometheus Operator

A new microservice team wants their metrics scraped. They have a `Service` with port `metrics: 9100` and label `app: inventory`. Write the ServiceMonitor.

**Answer:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: inventory
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: inventory
  endpoints:
    - port: metrics
      interval: 30s
```

### Drill 4 — relabeling actions

What's the difference between `labeldrop` and `drop`?

**Answer:** `labeldrop` removes a *label* from a metric's label set (reduces cardinality, e.g., removes the `user_id` label from every time series). `drop` removes the entire *metric* (or target, depending on which phase) where the regex matches the source labels. `labeldrop` is about removing a dimension; `drop` is about removing the whole observation.

---

## Further reading

- [Prometheus configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Prometheus exposition formats](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [Prometheus Operator documentation](https://prometheus-operator.dev/docs/getting-started/introduction/)
- [Robust Perception blog — relabeling](https://www.robustperception.io/life-of-a-label/) — Brian Brazil's "Life of a Label" is the canonical relabeling walkthrough
- [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) — the standard install; reading its defaults is the fastest way to understand production Kubernetes Prometheus setup

---

## The 4 dimensions

- **Tech**: pull model + kubernetes_sd_config + two-phase relabeling + TSDB. Histograms over summaries for aggregatable quantiles. ServiceMonitor as the Kubernetes-native abstraction over raw scrape configs.
- **People**: Prometheus Operator means app teams write ServiceMonitor YAMLs — a minimal surface, no Prometheus internals needed. Platform team owns the Prometheus CR and the cluster-level relabeling defaults. New engineers onboard to "add a ServiceMonitor to your chart."
- **CI/CD**: ServiceMonitor and PrometheusRule CRDs are Kubernetes manifests — they live in the app's Helm chart, reviewed in PRs, applied by the same GitOps pipeline as everything else. Linting: `promtool check rules` in CI. Alert routing config changes go through Alertmanager CR PRs.
- **Operations**: `up` metric is the health signal — alert on `up == 0` for any scraped target. `scrape_duration_seconds` flags slow targets (high-cardinality exporters that take >10s are a cardinality smell). `prometheus_tsdb_head_series` is the cardinality counter — alert when it passes 1M series on a single Prometheus instance. See [`cardinality-discipline.md`](../cardinality-discipline/deep-dive.md) for the full story.
