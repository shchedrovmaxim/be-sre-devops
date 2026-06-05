# Prometheus architecture — the simple version (the newspaper reporter)

> Read this first. Once the pull-model analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **Prometheus is a reporter, not a mailbox. It goes out and asks targets "what's your metric right now?" on a fixed schedule — targets don't push data to it.**

That pull model is the architectural choice everything else follows from.

---

## Your monitoring is a newspaper reporter

A traditional push-based system is like a city where every business *mails* its daily report to city hall. The mayor gets flooded with mail. If the mail room breaks, reports are lost.

Prometheus works the opposite way — it's a reporter who walks around every 15 or 30 seconds with a clipboard:

> "Hey, `/metrics`, what numbers do you have right now?"

| Reporter analogy | Prometheus world |
|---|---|
| The reporter | Prometheus server |
| Each business on the beat | A **target** (a pod, node, service) |
| The clipboard round | A **scrape** |
| How often the reporter visits | `scrape_interval` |
| The list of businesses to visit | **Service discovery** (Kubernetes, static_configs, file_sd) |
| Skipping a business because it's closed | **Relabeling**: `action: drop` |
| Relabeling the business as "bakery" when the map says "shop" | **Relabeling**: `action: replace` |

The `/metrics` endpoint exposes plain text. A counter looks like this:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200"} 4827
http_requests_total{method="POST", status="500"} 12
```

Prometheus walks up, reads those lines, timestamps them, and stores each `{metric_name + label set}` as a time series.

---

## The 4 metric types — what each one means

| Type | What it does | Example |
|---|---|---|
| **Counter** | Only goes up (or resets to 0 on restart) | `http_requests_total` |
| **Gauge** | Can go up or down | `memory_usage_bytes` |
| **Histogram** | Counts observations into buckets (for latency) | `http_request_duration_seconds_bucket` |
| **Summary** | Like histogram but pre-computes quantiles client-side | `grpc_server_handling_seconds` |

The senior rule: **prefer histograms over summaries**. Histograms aggregate correctly across instances. Summary quantiles don't.

---

## The scrape pipeline in plain English

When Prometheus scrapes a target, it goes through 5 steps:

1. **Discover the target** — from Kubernetes SD, static config, or a file.
2. **Relabel the target** — before scraping. This is where you `keep` or `drop` targets based on their labels, or rename them.
3. **Fetch `/metrics`** — HTTP GET, read the text, parse the lines.
4. **Relabel the metrics** — after scraping. This is where you drop noisy label values, rename labels, or add new ones.
5. **Store** — each unique `{name + label set}` becomes a time series stored in the TSDB (time-series database).

The two relabeling phases trip people up. Think of it as:
- **Before scrape** (`relabel_configs`): filters and rewrites the *target itself* (its address, port, job label).
- **After scrape** (`metric_relabel_configs`): filters and rewrites the *metrics* returned by that target.

---

## Why pull instead of push?

This comes up in interviews. The pull model has three concrete benefits:

1. **You know when a target goes silent.** If a pod crashes, Prometheus sees it as a scrape failure — and you can alert on `up == 0`. With push, a dead sender just stops sending and you might not notice.
2. **Prometheus controls the schedule.** No target can flood Prometheus with data; it only gets data at the interval Prometheus chooses.
3. **Easy integration testing.** You can curl any `/metrics` endpoint manually and see exactly what Prometheus sees.

The push model (used by Graphite, InfluxDB, and the Prometheus **Pushgateway** for batch jobs) requires the sender to know the collector's address, handle retries, and manage its own buffer.

---

## ServiceMonitor — the Kubernetes-native abstraction

In a raw Prometheus config you'd write a `scrape_config` in YAML and reload. In a Kubernetes cluster with the **Prometheus Operator**, you write a `ServiceMonitor` CRD instead:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-api
spec:
  selector:
    matchLabels:
      app: my-api
  endpoints:
    - port: metrics
      interval: 30s
```

The Prometheus Operator watches for ServiceMonitor objects and rewrites the Prometheus configuration automatically. You never touch the raw `prometheus.yml` in Kubernetes again.

---

## Self-test (out loud)

> **"Walk me through what Prometheus does when it scrapes a target — including how relabeling works."**

**Reference answer:**

"Prometheus maintains a list of targets from service discovery — in Kubernetes that's the Kubernetes SD config watching pods and services. Each scrape cycle, it iterates the target list, optionally relabeling targets first (to drop irrelevant ones or rewrite labels). It then does an HTTP GET to the target's `/metrics` endpoint, parses the exposition format, optionally relabels the metrics (to drop noisy labels or rename things), and stores the resulting time series in the local TSDB. Each unique combination of metric name and label set is one time series. Retention defaults to 15 days. For longer retention you'd add Thanos or Mimir in front."

---

## Further reading

- Prometheus docs on [scrape config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- Prometheus docs on [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)
- [Prometheus Operator ServiceMonitor](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#monitoring.coreos.com/v1.ServiceMonitor)
- [Robust Perception blog](https://www.robustperception.io/blog/) — Brian Brazil's writing is the single best Prometheus reference outside the official docs

---

## Next: the deep-dive

When the reporter analogy is obvious, jump to [`prometheus-architecture.md`](./deep-dive.md). The deep-dive covers:

- The full scrape pipeline with all relabeling actions and their semantics
- Kubernetes SD config patterns (`__meta_kubernetes_*` labels)
- How Prometheus Operator's ServiceMonitor / PodMonitor translate to scrape configs
- The TSDB internals (chunks, WAL, compaction)
- Cardinality and why it's load-bearing
- 3 relabeling drills, the 60-second interview answer, and the 4-dimensions framing
