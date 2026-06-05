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
| SLI / SLO / Error budgets — simple version (monthly data-plan analogy) | [`slos-simple.md`](./slos-simple.md) | 2026-06-05 |
| SLI / SLO / SLA / Error budgets — deep-dive (CUJ, budget policy, multi-window multi-burn-rate alerts, 3 worked examples, common mistakes) | [`slos.md`](./slos.md) | 2026-06-05 |

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
