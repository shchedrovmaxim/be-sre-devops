# OpenTelemetry Collector — the deep-dive

> **Goal**: by the end you can answer — **"Walk me through OTel Collector receivers / processors / exporters and where you'd run it (agent vs gateway)."** — covering the pipeline model, key processors, deployment patterns, and tail-sampling configuration.

> Start with [otel-collector-simple.md](./otel-collector-simple.md) first. The sorting-facility analogy is the spine.

---

## The senior framing — vendor-neutral telemetry layer

The OTel Collector is not just "a thing that forwards traces." It's the **vendor-neutral data layer** between your applications and your observability backends. Its existence means:

- Applications speak one protocol (OTLP) to one endpoint. They don't know about Datadog, Grafana Tempo, Jaeger, or whatever comes next.
- Switching backends is a collector config change, not an application change.
- Processing (sampling, enrichment, redaction) happens centrally, not in every SDK.
- A single place to add a new backend (fan-out exporter) or a new processor (PII scrubbing) for all telemetry.

This is the same argument as "use a message bus instead of point-to-point integrations" — applied to observability.

---

## Pipeline model

A pipeline connects: one or more receivers → zero or more processors → one or more exporters.

```yaml
service:
  pipelines:
    traces:                              # pipeline name (traces|metrics|logs)
      receivers:  [otlp, jaeger]        # can have multiple
      processors: [memory_limiter, resourcedetection, batch]  # ordered
      exporters:  [otlp/tempo, logging] # can have multiple (fan-out)
    metrics:
      receivers:  [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp, filelog]
      processors: [memory_limiter, batch]
      exporters:  [loki]
```

**Processor order matters.** The convention:
1. `memory_limiter` — must be first (drops data before processing if memory is exceeded)
2. `resourcedetection`, `k8sattributes` — enrichment
3. `attributes`, `filter`, `transform` — data manipulation
4. `tail_sampling` — in gateway only, needs all spans of a trace
5. `batch` — must be last (buffers for throughput)

---

## Receivers — what you'll use

### `otlp` — the primary receiver

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 4
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins: ["https://myapp.internal"]
```

All modern OTel SDKs default to OTLP. This is what applications send to.

### `prometheus` — scrape Prometheus endpoints

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'my-service'
          static_configs:
            - targets: ['localhost:8080']
```

Useful when you want the collector to scrape and then forward to a remote backend (e.g., Grafana Cloud) without running a full Prometheus server.

### `hostmetrics` — system metrics from the host

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:
      load:
      processes:
```

In a DaemonSet agent, this gives you OS-level metrics without `node_exporter`. The metrics follow OTel semantic conventions (`system.cpu.utilization`, etc.).

### `filelog` — read log files

```yaml
receivers:
  filelog:
    include:
      - /var/log/pods/*/*/*.log
    start_at: beginning
    include_file_path: true
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.time
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
```

Collects container logs from the node's log directory. Parses JSON structured logs.

### `k8s_events` — Kubernetes events as logs

```yaml
receivers:
  k8s_events:
    namespaces: [production, staging]
```

Sends Kubernetes events (pod evictions, OOMKills, node pressure) as log entries to your log backend.

### `jaeger` — legacy Jaeger protocol

```yaml
receivers:
  jaeger:
    protocols:
      thrift_http:
        endpoint: 0.0.0.0:14268
      grpc:
        endpoint: 0.0.0.0:14250
```

For migrating from Jaeger SDK to OTLP. Accept Jaeger-format traces, export via OTLP.

---

## Processors — the critical ones

### `memory_limiter` — first in every pipeline

```yaml
processors:
  memory_limiter:
    limit_mib: 512          # hard limit
    spike_limit_mib: 128    # headroom for GC spikes
    check_interval: 1s      # how often to check memory
```

When memory exceeds `limit_mib`, the collector starts refusing new data (returning errors to receivers). When it exceeds `limit_mib - spike_limit_mib`, it starts dropping data in the pipeline. Without this, the collector can OOM and crash, losing all buffered data.

**Must be the first processor in every pipeline.**

### `resourcedetection` — cloud/host metadata enrichment

```yaml
processors:
  resourcedetection:
    detectors: [env, aws, k8s_node]   # or gcp, azure, system
    timeout: 5s
    override: false   # don't override attributes already set by the SDK
```

Auto-detects and attaches:
- `cloud.provider`, `cloud.region`, `cloud.availability_zone` (from AWS/GCP/Azure metadata API)
- `k8s.node.name` (from Kubernetes Downward API)
- `host.name`, `os.type`

Run this on the **agent** (DaemonSet) — it needs access to the node's metadata endpoint.

### `k8sattributes` — Kubernetes pod metadata

```yaml
processors:
  k8sattributes:
    auth_type: serviceAccount
    extract:
      metadata:
        - k8s.pod.name
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.node.name
      labels:
        - tag_name: team
          key: team
          from: pod
      annotations:
        - tag_name: git_sha
          key: git-sha
          from: pod
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
```

Enriches traces and metrics with Kubernetes labels and annotations from the pod. The collector calls the Kubernetes API to look up the pod by IP. Requires a `ClusterRole` with read access to pods.

### `batch` — throughput optimization

```yaml
processors:
  batch:
    timeout: 5s           # send every 5s even if batch isn't full
    send_batch_size: 1024 # or when batch reaches this many spans
    send_batch_max_size: 2048
```

Groups spans/samples into batches before exporting. Without batching, each span is a separate network call — extremely inefficient. The batch processor is the single biggest throughput improvement for the collector.

**Must be the last processor before exporters.**

### `tail_sampling` — keep the right traces

```yaml
processors:
  tail_sampling:
    decision_wait: 30s      # wait up to 30s for all spans of a trace
    num_traces: 100000      # max traces buffered in memory
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 1000}
      - name: sample-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 10}
```

Tail-based sampling: the collector buffers spans until a trace is complete, then decides whether to keep it. This lets you keep 100% of error traces and slow traces, and sample the rest. Head-based sampling (deciding at trace start) can't do this — you don't know if a trace will be interesting at its start.

**Only run `tail_sampling` in the gateway, not the agent.** The gateway needs to see all spans for a given trace — which means routing by trace ID is required when you have multiple gateway replicas (use the `loadbalancing` exporter to route traces from agents to gateway based on trace ID).

### `attributes` — add, update, delete, hash attributes

```yaml
processors:
  attributes:
    actions:
      # Add a static attribute to all telemetry
      - key: deployment.environment
        value: production
        action: insert
      # Hash a sensitive value (PII protection)
      - key: user.email
        action: hash
      # Delete a label you don't want in your backend
      - key: http.request.header.authorization
        action: delete
```

### `filter` — drop unwanted telemetry

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/healthz"'
        - 'attributes["http.target"] == "/metrics"'
    metrics:
      metric:
        - 'name == "go.gc.heap.frees.by.size"'
```

Drops health check traces (they're noise), specific metrics you don't want to pay for. Use OTTL (OpenTelemetry Transformation Language) expressions.

---

## Exporters — key ones

### `otlp` — OTLP gRPC to a backend

```yaml
exporters:
  otlp:
    endpoint: grafana-tempo:4317
    tls:
      insecure: true   # for internal cluster communication
  otlp/datadog:
    endpoint: https://trace.agent.datadoghq.com:443
    headers:
      DD-Api-Key: ${DD_API_KEY}
```

### `prometheusremotewrite` — push metrics to Prometheus-compatible backend

```yaml
exporters:
  prometheusremotewrite:
    endpoint: http://mimir:8080/api/v1/push
    headers:
      X-Scope-OrgID: my-team
    resource_to_telemetry_conversion:
      enabled: true   # convert resource attributes to metric labels
```

### `loki` — push logs to Loki

```yaml
exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    labels:
      resource:
        k8s.namespace.name: "namespace"
        k8s.pod.name: "pod"
        k8s.container.name: "container"
```

### `loadbalancing` — route by trace ID to gateway replicas

```yaml
exporters:
  loadbalancing:
    routing_key: traceID   # ensures all spans of a trace go to same gateway replica
    protocol:
      otlp:
        timeout: 5s
    resolver:
      dns:
        hostname: otelcol-gateway.monitoring.svc.cluster.local
        port: 4317
```

Required when running multiple gateway replicas with `tail_sampling` — you must route all spans of a trace to the same replica or the sampling decision is split across replicas (and you'd lose context).

---

## Agent vs Gateway — the production pattern

```
┌─────────────────────────────────────────────────────────────┐
│ Node 1                                                      │
│  [Pod A] ──OTLP──→ ┐                                       │
│  [Pod B] ──OTLP──→ ├──→ [OTel Agent DaemonSet]            │
│  [Host metrics]  ──┘     - receivers: otlp, hostmetrics    │
│                           - processors: memory_limiter,     │
│                             resourcedetection, k8sattributes│
│                           - exporter: loadbalancing → GW    │
└──────────────────────────────────────┬──────────────────────┘
                                       │ OTLP (routed by traceID)
                                       ▼
                    ┌──────────────────────────────────────┐
                    │ OTel Gateway (Deployment, 3 replicas) │
                    │  - receivers: otlp                    │
                    │  - processors: memory_limiter,        │
                    │    tail_sampling, batch               │
                    │  - exporters: tempo, mimir, loki      │
                    └──────────────────────────────────────┘
```

The agent is lightweight (targets 50–200 MiB memory), runs on every node, handles local collection. The gateway does the heavy lifting: tail sampling (needs buffering), routing to multiple backends, and centralized filtering.

---

## Kubernetes deployment

### DaemonSet agent

```yaml
# values for opentelemetry-collector Helm chart
mode: daemonset
presets:
  hostMetrics:
    enabled: true
  kubernetesAttributes:
    enabled: true
  logsCollection:
    enabled: true

resources:
  limits:
    memory: 256Mi
    cpu: 200m
```

### Gateway Deployment

```yaml
mode: deployment
replicaCount: 3

resources:
  limits:
    memory: 1Gi
    cpu: 1

# tail_sampling needs sticky routing — use the loadbalancing exporter
# in agents to send to gateway with traceID routing
```

---

## Debugging the collector

```yaml
# Add to service section
service:
  telemetry:
    logs:
      level: debug     # verbose — use only when debugging
    metrics:
      level: detailed
      address: 0.0.0.0:8888   # collector's own Prometheus metrics

extensions:
  health_check:
    endpoint: 0.0.0.0:13133  # liveness/readiness probe
  zpages:
    endpoint: 0.0.0.0:55679  # web UI showing pipeline stats at /debug/tracez
  pprof:
    endpoint: 0.0.0.0:1777   # Go pprof for CPU/memory profiling
```

The collector exposes its own metrics at `:8888/metrics`. Key ones:
- `otelcol_receiver_accepted_spans` — spans successfully received
- `otelcol_exporter_sent_spans` — spans successfully exported
- `otelcol_processor_dropped_items` — items dropped (usually `memory_limiter` or `filter`)
- `otelcol_exporter_queue_size` — backpressure indicator

---

## The 60-second interview answer

> "The OTel Collector has three pipeline stages: receivers accept incoming telemetry (OTLP is the primary; also Jaeger, Prometheus scrape, hostmetrics for system metrics, filelog for log files), processors transform it in order (memory_limiter first — drops before OOM; resourcedetection and k8sattributes for Kubernetes metadata enrichment; tail_sampling in the gateway; batch last for throughput), and exporters dispatch to backends (OTLP to Tempo for traces, prometheusremotewrite to Mimir for metrics, Loki for logs).
>
> The agent vs gateway pattern: I run a DaemonSet agent on every node — lightweight, local collection, Kubernetes metadata enrichment, forward to the gateway. The gateway is a centralized Deployment with 2–3 replicas — it does tail-based sampling (which needs to see a full trace before deciding to keep it), routes to multiple backends, and handles any heavy filtering or enrichment. Agents use the `loadbalancing` exporter with `routing_key: traceID` so all spans of a trace land on the same gateway replica — required for tail sampling to work correctly.
>
> The senior framing: the collector is the vendor-neutral data layer. Applications speak OTLP to one endpoint; the collector handles all backend-specific translation. Switching observability vendors is a config change, not an application change."

---

## Self-test drills

### Drill 1 — processor order

Why must `memory_limiter` be first and `batch` be last in the processor chain?

**Answer:** `memory_limiter` must be first because it applies backpressure before any processing happens — if it's after expensive processors, the collector may OOM before the limiter can act. `batch` must be last because it buffers spans and sends them in groups; if it's before other processors, those processors would receive batched data in the wrong format and the batch would be broken up.

### Drill 2 — tail sampling

Why can't you do tail sampling in the agent (DaemonSet)?

**Answer:** Tail sampling requires seeing all spans of a trace before making a keep/drop decision. A single trace spans multiple services, each running on different nodes. The DaemonSet agent on node 1 only sees spans from pods on node 1 — it will never see a complete trace. The gateway is a central bottleneck that receives all spans from all agents, so it can buffer and complete traces before sampling.

### Drill 3 — loadbalancing exporter

You have 3 gateway replicas all with `tail_sampling`. An agent sends spans from the same trace to different gateway replicas. What goes wrong?

**Answer:** Each gateway replica buffers partial traces. When the `decision_wait` timeout expires, replica A sees 60% of the trace spans (and decides: keep, because there's an error), replica B sees 40% (and decides: drop, because its fragment looks like a fast success). Some spans are exported, some are dropped. You get incomplete, incoherent traces. The fix: use the `loadbalancing` exporter in agents with `routing_key: traceID` — all spans for a given trace ID hash to the same gateway replica.

---

## Further reading

- [OTel Collector configuration](https://opentelemetry.io/docs/collector/configuration/)
- [OTel Collector contrib GitHub](https://github.com/open-telemetry/opentelemetry-collector-contrib) — all receivers, processors, exporters
- [k8sattributes processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor)
- [tail_sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)
- [OTel Helm charts](https://github.com/open-telemetry/opentelemetry-helm-charts)

---

## The 4 dimensions

- **Tech**: receivers (OTLP, hostmetrics, filelog, k8s_events), processors ordered: memory_limiter → resourcedetection/k8sattributes → tail_sampling (gateway only) → batch, exporters (OTLP, prometheusremotewrite, loki). Agent DaemonSet for local collection, Gateway Deployment for tail sampling and fan-out. Loadbalancing exporter for trace-ID-based sticky routing to gateway replicas.
- **People**: app teams add the OTel SDK and set `OTEL_EXPORTER_OTLP_ENDPOINT` to the local agent. They don't know or care about Tempo, Mimir, or Loki. Platform team owns the collector config. Backend migrations (e.g., Jaeger → Tempo) are a platform config change, not a ticket to 30 application teams.
- **CI/CD**: collector config as a Helm chart value, reviewed in PRs. `otelcol validate --config=config.yaml` in CI. Collector version upgrades tested in staging first (breaking changes in contrib processors do happen). Kubernetes liveness probe on the health_check extension — deployment rollout fails fast if the new config is invalid.
- **Operations**: dashboard `otelcol_exporter_queue_size` (backpressure = backend is slow), `otelcol_processor_dropped_items` (limiter or filter dropping data), `otelcol_receiver_refused_spans` (collectors rejecting inbound). Alert on queue depth > 1000 sustained for 5 minutes. The zpages endpoint (`/debug/tracez`) is invaluable for debugging sampling decisions in development.
