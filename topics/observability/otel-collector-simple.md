# OpenTelemetry Collector — the simple version (the sorting facility analogy)

> Read this first. Once the intake → sort → dispatch analogy clicks, the [deep-dive](./otel-collector.md) becomes easy.

This doc explains **one idea**:

> **The OTel Collector is the sorting facility between your applications and your observability backends. Applications send telemetry to the collector; the collector cleans, batches, routes, and forwards it.**

---

## Your telemetry data is a package sorting facility

Imagine a city's postal sorting facility. Packages arrive from all over (your apps), go through a sorting conveyor (processors), and get dispatched to the right destinations (your backends — Grafana, Datadog, Jaeger, etc.).

| Sorting facility | OTel Collector |
|---|---|
| Packages arriving | **Receiver** — telemetry coming in (OTLP, Prometheus scrape, Jaeger, etc.) |
| Conveyor belt with scanners | **Processor** — batch, filter, enrich, sample |
| Dispatch dock | **Exporter** — sends to Grafana Cloud, Datadog, Prometheus, Jaeger, etc. |
| Connecting conveyor sections | **Pipeline** — wires receiver → processor(s) → exporter |

Three categories: receivers, processors, exporters. One or more pipelines (traces, metrics, logs) connecting them.

---

## A minimal config

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317   # apps send here
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  otlp:
    endpoint: grafana-tempo:4317   # send traces to Tempo
  prometheusremotewrite:
    endpoint: http://mimir:8080/api/v1/push   # send metrics to Mimir

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
```

---

## Agent vs gateway — the two deployment patterns

This is what interviewers want to hear.

| | Agent | Gateway |
|---|---|---|
| Where it runs | On every node (DaemonSet) or as a sidecar | Centralized (Deployment, 2-3 replicas) |
| What it does | Collects data locally, lightweight enrichment | Heavy processing: tail sampling, routing, fan-out |
| Memory footprint | Small (50–100 MiB target) | Larger (can handle buffering) |
| Best for | Scraping host metrics, receiving from local apps | Tail-based sampling, routing to multiple backends |

**The typical production pattern**: Agent (DaemonSet) receives from local pods → enriches with Kubernetes metadata → forwards to Gateway → Gateway does tail sampling and routes to multiple backends.

```
[Pod A] ─── OTLP ──→ [OTel Agent on Node 1] ─┐
[Pod B] ─── OTLP ──→ [OTel Agent on Node 1] ─┤
                                               ├──→ [OTel Gateway] ──→ [Tempo]
[Pod C] ─── OTLP ──→ [OTel Agent on Node 2] ─┤                   ──→ [Mimir]
[Pod D] ─── OTLP ──→ [OTel Agent on Node 2] ─┘                   ──→ [DataDog]
```

---

## Why the collector exists at all

Without a collector, every application SDK has to know about every observability backend. Change your tracing vendor → update every service. Add a new backend → update every service.

With the collector, applications speak one protocol (OTLP) to one endpoint (the collector). The collector handles all the routing, batching, and backend-specific translation. **Vendor-neutral data layer** — the senior framing.

---

## Self-test (out loud)

> **"Walk me through OTel Collector receivers / processors / exporters and where you'd run it (agent vs gateway)."**

**Reference answer:**

"The OTel Collector has three stages in each pipeline: receivers accept incoming telemetry (OTLP gRPC, Jaeger, Prometheus scrape, host metrics), processors transform it (batch for throughput, memory_limiter for safety, resourcedetection to add host/cloud metadata, tail_sampling to keep interesting traces), and exporters send it out (OTLP to Tempo for traces, prometheusremotewrite to Mimir for metrics, logging for debugging).

The agent vs gateway split: I run a lightweight DaemonSet agent on every node — it receives from local pods, adds Kubernetes metadata, and forwards upstream. The gateway is a small centralized Deployment — it does the heavy processing like tail sampling (which needs to see a full trace before deciding to keep it) and routing to multiple backends. Applications send to the local agent, never directly to backends."

---

## Further reading

- [OTel Collector docs](https://opentelemetry.io/docs/collector/)
- [OTel Collector contrib receivers/processors/exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib)
- [OTel Collector Helm chart](https://github.com/open-telemetry/opentelemetry-helm-charts)

---

## Next: the deep-dive

When the sorting-facility analogy is obvious, jump to [`otel-collector.md`](./otel-collector.md). The deep-dive covers:

- Every major receiver, processor, and exporter with config examples
- The `resourcedetection` processor and Kubernetes metadata enrichment
- Tail sampling processor config in detail
- The `memory_limiter` and why it must be first in the pipeline
- Health check and zPages for debugging the collector itself
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
