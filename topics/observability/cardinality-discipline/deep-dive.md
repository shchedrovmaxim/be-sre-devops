# Cardinality discipline — the deep-dive

> **Goal**: by the end you can answer — **"Your Prometheus is OOMing. What does 'cardinality explosion' mean and how would you find the culprit?"** — covering detection via TSDB API, the topk trick, remediation via relabeling, and the Loki parallel.

> Start with [cardinality-discipline-simple.md](./simple.md) first. The filing-cabinet analogy is the spine.

---

## The senior framing — cardinality is the silent killer

Cardinality problems are insidious because they don't cause an error — they cause a slow OOM. The metric count grows, memory climbs, queries slow, and then one day Prometheus is OOMKilled and immediately OOMs again on restart because it restores from WAL.

The pattern: a developer adds a new label to track something useful ("let's tag by customer_id so we can debug per-customer"). Passes code review. Deploys Friday. By Monday Prometheus is at 2M series and climbing. By Wednesday it's dead.

Senior engineers build **defensive systems**: cardinality limits, CI checks, and relabeling guardrails so this can't happen silently.

---

## The math: what "cardinality" actually means

**Cardinality** of a metric = the number of distinct time series produced by that metric.

A time series is uniquely identified by: `metric_name + {label1=val1, label2=val2, ...}`.

Cardinality = product of distinct value counts per label:

```
http_requests_total with:
  - job: 10 services
  - status: 6 values (200, 201, 400, 404, 500, 503)
  - method: 4 values (GET, POST, PUT, DELETE)

Cardinality = 10 × 6 × 4 = 240 series
```

Add `region: 5 values` → 240 × 5 = 1,200 series. Still fine.

Add `customer_id: 50,000 values` → 1,200 × 50,000 = **60 million series**. Prometheus is dead.

### Memory cost per series

Each active time series in Prometheus head (the in-memory TSDB head block) costs approximately:
- **~1–3 KB** for the label set storage
- **~768 bytes** for the raw sample chunk
- Total: roughly **~2 KB per series** in practice (varies by label cardinality per series)

| Series count | Memory footprint (approx) | Status |
|---|---|---|
| 100,000 | ~200 MB | Fine — single Prometheus |
| 500,000 | ~1 GB | OK with 4+ GB allocation |
| 1,000,000 | ~2 GB | Starting to strain |
| 2,000,000 | ~4 GB | Dangerous — OOM likely on a default install |
| 5,000,000 | ~10 GB | Dead without serious sizing |
| 10,000,000 | ~20 GB | Requires Thanos/Mimir/VictoriaMetrics |

The default `kube-prometheus-stack` resource limits are 2 GiB. At 1M series you're at the edge. This is why cardinality discipline matters.

---

## Detection — finding the culprit

### Method 1: PromQL (the first stop)

```promql
# Total active series in the TSDB head
prometheus_tsdb_head_series

# Top 10 metric families by series count
topk(10,
  count by (__name__) ({__name__=~".+"})
)
```

**Warning**: the `count by (__name__)` query itself is expensive at >1M series. In a degraded Prometheus, it may time out. Run it during a quieter period or use the API directly.

If you find the offending metric family, drill into it:

```promql
# What labels does the top offender have, and how many distinct values?
count by (customer_id) (http_requests_total)

# How many distinct values for all labels of a metric?
count(count by (customer_id) (http_requests_total))
```

If `count(count by (customer_id) (http_requests_total))` returns 500,000 — you have 500,000 distinct `customer_id` values. That's your bomb.

### Method 2: TSDB Admin API

Prometheus exposes a TSDB introspection API (requires `--web.enable-admin-api` flag):

```bash
# Get series count per metric
curl http://prometheus:9090/api/v1/label/__name__/values

# TSDB stats — top 10 series by label count
curl http://prometheus:9090/api/v1/tsdb/status | jq '.data.seriesCountByMetricName[:10]'
```

The `tsdb/status` endpoint returns:
- `seriesCountByMetricName` — top metrics by series count
- `seriesCountByLabelName` — which *label name* contributes most series
- `seriesCountByFocusLabelValue` — for a specific label, which value has the most series
- `memoryInBytesByLabelName` — memory cost per label name

This is the fastest route during an incident:

```bash
curl -s http://prometheus:9090/api/v1/tsdb/status \
  | jq '.data.seriesCountByMetricName[:5]'
# {value: 2400000, name: "http_requests_total"}
# {value: 800000, name: "grpc_server_handled_total"}
# ...

curl -s http://prometheus:9090/api/v1/tsdb/status \
  | jq '.data.seriesCountByLabelName[:5]'
# {value: 2000000, name: "customer_id"}
# {value: 500000, name: "user_email"}
# ...
```

### Method 3: Grafana Mimirtool (for Mimir/Cortex deployments)

```bash
# Analyze cardinality per tenant
mimirtool analyze grafana --address=http://mimir:8080

# Top cardinality metrics
mimirtool cardinality list --address=http://mimir:8080
```

---

## The common cardinality anti-patterns

### 1. Unbounded identifier labels

```python
# DON'T DO THIS
metrics.counter("http_requests_total",
  labels={"user_id": user_id, "request_id": request_id})

# DO THIS
metrics.counter("http_requests_total",
  labels={"job": "checkout-api", "status": str(response.status_code)})
```

Rule: if the label value comes from user input, a UUID, a timestamp, or a database ID, it's not a label. It's a log field.

### 2. High-cardinality URL paths

```python
# BAD: path includes dynamic segment
labels={"path": "/api/v1/users/a3f9b2c1/orders/9182736"}

# GOOD: template the path
labels={"path": "/api/v1/users/{id}/orders/{id}"}
```

Most instrumentation libraries (OpenTelemetry, Prometheus client) allow you to configure a route template instead of the raw URL path.

### 3. Putting log data in metrics

```python
# BAD: error message as a label
labels={"error": "connection timeout to db-host-17.internal:5432"}

# GOOD: error type only
labels={"error_type": "db_connection_timeout"}
```

Error messages are unbounded. Error *types* are a handful of values.

### 4. Label explosion from external metadata

Kubernetes labels and annotations can have thousands of distinct values. Relabeling rules like `action: labelmap, regex: __meta_kubernetes_pod_label_(.+)` promote all pod labels to metric labels. If pods carry labels like `git_sha` (one per commit), you've just created a new label with a new value on every deployment.

Fix: be selective about which Kubernetes labels you promote.

---

## Remediation — dropping the bad label

### Option 1: `metric_relabel_configs` in the scrape config

Drop a label from all metrics returned by a target before they enter TSDB:

```yaml
scrape_configs:
  - job_name: 'checkout-api'
    ...
    metric_relabel_configs:
      # Drop the user_id label entirely
      - action: labeldrop
        regex: "user_id"

      # Drop the entire metric family if it's the culprit
      - source_labels: [__name__]
        action: drop
        regex: "debug_internal_.*"
```

### Option 2: ServiceMonitor `metricRelabelings` (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: checkout-api
spec:
  endpoints:
    - port: metrics
      metricRelabelings:
        - action: labeldrop
          regex: "customer_id|user_email|session_id"
```

### Option 3: Delete series from the TSDB (emergency)

Prometheus admin API allows deleting series (requires `--web.enable-admin-api`):

```bash
# Delete all series with a specific label value
curl -X POST http://prometheus:9090/api/v1/admin/tsdb/delete_series \
  --data-urlencode 'match[]={customer_id=~".+"}'

# Then clean tombstones
curl -X POST http://prometheus:9090/api/v1/admin/tsdb/clean_tombstones
```

This is a last resort during an incident. The proper fix is dropping the label at the scrape level so it never comes back.

---

## Proactive cardinality governance

### PR-level checks

Add to your instrumentation library's CI:

```bash
# Count series that would be created by a metric definition
# (pseudo-code for a custom check)
promtool tsdb analyze ./data/ | grep "highest cardinality"

# Or: assert that no metric has more than 10,000 series
# (using mimirtool or a custom script against a test Prometheus)
```

### Cardinality limits in Prometheus

```yaml
# prometheus.yml global section
global:
  # Refuse to ingest samples that would exceed per-scrape cardinality limit
  sample_limit: 10000

# Per-scrape-config limit
scrape_configs:
  - job_name: 'my-service'
    sample_limit: 5000  # refuse if this target returns >5000 time series
    label_limit: 30     # refuse if any series has >30 labels
    label_name_length_limit: 64
    label_value_length_limit: 128
```

`sample_limit` is a hard brake — if a target returns more series than the limit, the entire scrape is dropped (not just the excess). You'll get `up=0` and a log line explaining why. Better than silently accepting a cardinality bomb.

---

## The Loki parallel — same lesson, different system

Loki indexes **stream labels**, not log content. A stream is a unique combination of label values. This is identical to Prometheus cardinality.

```yaml
# BAD: Loki streams with request_id label
{app="checkout", request_id="a3f9b2c1-..."}
# Each request creates a new stream. 1M requests/day = 1M streams.
# Loki index explodes. Queries slow to a crawl.

# GOOD: Loki streams with coarse labels only
{app="checkout", namespace="production", env="prod"}
# Handful of streams per deployment. Cheap index.
# Filter by request_id at query time using LogQL pattern matching.
```

LogQL extracts fields at query time, not at index time:

```logql
{app="checkout"} | logfmt | request_id="a3f9b2c1"
```

The label `request_id` is *in the log line* (structured JSON or logfmt), not in the Loki stream label. The query filters on it at read time — slower than a label match, but the index stays small.

The rule: **Loki stream labels are for grouping. Log fields are for filtering.** Same as Prometheus label discipline.

---

## The 60-second interview answer

> "Cardinality is the number of time series in Prometheus — each unique combination of metric name plus label values is one series stored in memory. A cardinality explosion happens when a label with an unbounded value set gets added. Classic example: `user_id` on an HTTP request counter. With 500K users × 4 HTTP methods × 6 status codes, you're at 12 million series. At ~2KB per series that's 24 GB — Prometheus dies.
>
> To find the culprit I'd start with the TSDB status API:
> ```bash
> curl http://prometheus:9090/api/v1/tsdb/status | jq '.data.seriesCountByLabelName[:5]'
> ```
> That shows which label name contributes the most series. Then confirm with PromQL:
> ```promql
> topk(10, count by (__name__) ({__name__=~".+"}))
> ```
> Once I've identified the metric and the offending label, the fix is `labeldrop` in `metric_relabel_configs` — it drops the label before the metric enters the TSDB. For the underlying need (per-user debugging), I'd route that to Loki as a structured log field — query-time filtering without index cost.
>
> Proactively I'd set `sample_limit: 5000` per scrape config and add cardinality budget checks to CI. Cardinality is a silent killer — the bomb is usually planted weeks before Prometheus dies."

---

## Self-test drills

### Drill 1 — cardinality math

A metric has labels: `job` (20 services), `status` (6 values), `method` (4 values), `region` (5 regions), and `user_id` (2 million users). What's the cardinality?

**Answer:** 20 × 6 × 4 × 5 × 2,000,000 = **4,800,000,000 series**. Four billion. That's not a typo. Prometheus would need terabytes of RAM. The fix: drop `user_id` entirely.

### Drill 2 — detection

You're paged for Prometheus OOM. Name 3 concrete steps you'd take in the first 10 minutes.

**Answer:**
1. Check `prometheus_tsdb_head_series` — is it above 1M? Is it still growing?
2. Run `curl .../api/v1/tsdb/status | jq '.data.seriesCountByLabelName[:5]'` to find which label name is the top offender.
3. Drill into the offending metric: `count by (offending_label) (metric_name)` to confirm it's unbounded.

### Drill 3 — remediation

The culprit is `customer_id` on `grpc_server_handled_total`. Write the ServiceMonitor `metricRelabelings` stanza to drop it.

**Answer:**
```yaml
metricRelabelings:
  - action: labeldrop
    regex: "customer_id"
```

### Drill 4 — governance

A PR adds `request_id` as a Prometheus label to a high-traffic metric. How do you catch this before it reaches production?

**Answer:** 
1. Code review: the label name `request_id` should be an immediate red flag — it's a per-request identifier.
2. CI gate: spin up a test Prometheus against the new instrumentation and assert `prometheus_tsdb_head_series` stays below a threshold after 60 seconds of synthetic traffic.
3. `sample_limit` per scrape config: if the new metric produces >5000 series, the scrape is rejected and generates an alert before it can OOM.

---

## Further reading

- [Prometheus best practices — instrumentation and labels](https://prometheus.io/docs/practices/instrumentation/#labels)
- [Prometheus TSDB admin API](https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis)
- [Grafana blog — cardinality spikes](https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/)
- [Robust Perception — you can have too many time series](https://www.robustperception.io/you-can-have-too-many-time-series/)
- [Loki label best practices](https://grafana.com/docs/loki/latest/get-started/labels/best-practices/)

---

## The 4 dimensions

- **Tech**: cardinality = product of distinct label-value counts. ~2KB per series in TSDB head. Detection via TSDB status API and `topk(10, count by (__name__) ...)`. Remediation via `labeldrop` in `metric_relabel_configs`. Defense via `sample_limit` per scrape config. The Loki parallel: stream labels are for grouping, not for filtering.
- **People**: cardinality bombs are almost always added by developers who don't know the model. The fix is education (explain the filing-cabinet mental model) and guardrails (PR checklists, `sample_limit`). Platform team needs to own cardinality as a service — it's as much a safety gate as memory limits or CPU quotas.
- **CI/CD**: add cardinality checks to the instrumentation library CI. A synthetic-traffic test that spins up Prometheus for 60 seconds and asserts `head_series < N` catches most bombs before they ship. Review any PR that adds a new label with a high entropy value (IDs, emails, URLs, hashes).
- **Operations**: `prometheus_tsdb_head_series` is the cardinality canary — dashboard it, alert when it crosses 500K, page when it approaches the memory limit. Monthly cardinality audits using the TSDB status API. When a new exporter or new instrumentation rolls out, watch the series count for the first 30 minutes.
