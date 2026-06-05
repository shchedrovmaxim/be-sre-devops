# Observability

Metrics, logs, traces. The signal you operate the system from. SLO-driven thinking is the senior bar.

## Recommended order

### Metrics
1. **Prometheus** — scrape config, targets, relabeling
2. **Recording rules** vs **alerting rules** — when each
3. **PromQL essentials** — `rate`, `histogram_quantile`, `topk`, `sum by`, `irate`
4. **Cardinality discipline** — the silent Prometheus killer
5. **Long-term storage** — Thanos / Mimir / VictoriaMetrics

### Tracing
6. **OpenTelemetry collector** — receivers → processors → exporters
7. **Semantic conventions** — why standardizing attribute names matters
8. **Sampling** — head-based vs tail-based; what each gets you

### Logs
9. **Loki** — label cardinality discipline (same lesson as Prometheus)
10. Structured logging — JSON > free text

### Dashboards & visualization
11. **Grafana** dashboards as code (Grafonnet, jsonnet)
12. Dashboard layout principles: USE / RED methods

### SLO design
13. **USE method** — Utilization / Saturation / Errors (for resources)
14. **RED method** — Rate / Errors / Duration (for services)
15. **SLI / SLO / SLA** — how each is different
16. **Error budgets** — how they drive engineering decisions
17. **Alert design** — actionable signals, no alert fatigue, the "page only on user-impact" rule

## Files

| Sub-topic | File | Date covered |
|---|---|---|
| SLI / SLO / Error budgets — simple version (monthly data-plan analogy) | [`slos-simple.md`](./slos/simple.md) | 2026-06-05 |
| SLI / SLO / SLA / Error budgets — deep-dive (CUJ, budget policy, multi-window multi-burn-rate alerts, 3 worked examples, common mistakes) | [`slos.md`](./slos/deep-dive.md) | 2026-06-05 |
| Prometheus architecture + scrape config + relabeling — simple version (newspaper reporter analogy) | [`prometheus-architecture-simple.md`](./prometheus-architecture/simple.md) | 2026-06-05 |
| Prometheus architecture + scrape config + relabeling — deep-dive (full pipeline, 7 relabeling actions, Kubernetes SD patterns, ServiceMonitor, 4 drills) | [`prometheus-architecture.md`](./prometheus-architecture/deep-dive.md) | 2026-06-05 |
| Recording rules vs alerting rules — simple version (sous chef analogy) | [`recording-vs-alerting-rules-simple.md`](./recording-vs-alerting-rules/simple.md) | 2026-06-05 |
| Recording rules vs alerting rules — deep-dive (naming convention, SLO burn-rate pattern, reload mechanics, PrometheusRule CRD, 4 drills) | [`recording-vs-alerting-rules.md`](./recording-vs-alerting-rules/deep-dive.md) | 2026-06-05 |
| PromQL essentials — simple version (odometer/speedometer analogy) | [`promql-essentials-simple.md`](./promql-essentials/simple.md) | 2026-06-05 |
| PromQL essentials — deep-dive (rate vs irate, histogram_quantile, topk, _over_time, absent, staleness, offset, 4 drills) | [`promql-essentials.md`](./promql-essentials/deep-dive.md) | 2026-06-05 |
| Cardinality discipline — simple version (filing cabinet analogy) | [`cardinality-discipline-simple.md`](./cardinality-discipline/simple.md) | 2026-06-05 |
| Cardinality discipline — deep-dive (cardinality math, memory cost, TSDB admin API, labeldrop remediation, sample_limit, Loki parallel, 4 drills) | [`cardinality-discipline.md`](./cardinality-discipline/deep-dive.md) | 2026-06-05 |
| Long-term storage (Thanos / Mimir / VictoriaMetrics) — simple version (filing room analogy) | [`long-term-storage-simple.md`](./long-term-storage/simple.md) | 2026-06-05 |
| Long-term storage (Thanos / Mimir / VictoriaMetrics) — deep-dive (architecture, deduplication, downsampling, trade-off matrix, 3 drills) | [`long-term-storage.md`](./long-term-storage/deep-dive.md) | 2026-06-05 |
| OTel Collector — simple version (sorting facility analogy) | [`otel-collector-simple.md`](./otel-collector/simple.md) | 2026-06-05 |
| OTel Collector — deep-dive (receivers, processors, exporters, agent vs gateway, tail sampling, loadbalancing exporter, 3 drills) | [`otel-collector.md`](./otel-collector/deep-dive.md) | 2026-06-05 |
| OTel semantic conventions + sampling — simple version (passport / border analogy) | [`otel-conventions-sampling-simple.md`](./otel-conventions-sampling/simple.md) | 2026-06-05 |
| OTel semantic conventions + sampling — deep-dive (all namespaces, HTTP migration, head/tail sampling, W3C TraceContext, tail_sampling processor, 3 drills) | [`otel-conventions-sampling.md`](./otel-conventions-sampling/deep-dive.md) | 2026-06-05 |
| Grafana dashboards as code — simple version (blueprint analogy) | [`grafana-as-code-simple.md`](./grafana-as-code/simple.md) | 2026-06-05 |
| Grafana dashboards as code — deep-dive (Grafonnet, Foundation SDK, Grafana Operator, file provisioning, testing, access control, 3 drills) | [`grafana-as-code.md`](./grafana-as-code/deep-dive.md) | 2026-06-05 |
| Loki + label cardinality — simple version (library catalogue analogy) | [`loki-cardinality-simple.md`](./loki-cardinality/simple.md) | 2026-06-05 |
| Loki + label cardinality — deep-dive (storage model, chunks_per_stream canary, LogQL extraction, structured logging, ingestion limits, 4 drills) | [`loki-cardinality.md`](./loki-cardinality/deep-dive.md) | 2026-06-05 |

## Why this matters

Observability is where the SRE day-to-day actually happens. A senior engineer reasons about the system through the dashboards and traces, not just the code.

## Hands-on environment

- Local `kind` cluster + kube-prometheus-stack via Helm
- Tempo or Jaeger for traces
- Loki for logs
- Grafana to visualize all three

## Cost discipline

Observability spend can exceed compute spend on smaller teams. Senior signal:
- Cardinality audit monthly
- Sampling strategy explicit, not default
- Retention policies set from day one
- Recording rules for expensive queries
