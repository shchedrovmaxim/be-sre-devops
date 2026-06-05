# HorizontalPodAutoscaler (HPA) — deep-dive

> **Goal**: by the end you can answer the killer question — **"Your HPA isn't scaling under load. Walk me through what to check, in order."** — naming the metrics pipeline, the per-pod metric subtlety, `ScalingActive` conditions, `maxReplicas`, stabilization windows, and KEDA as the event-driven alternative.

> Start with the [simple version](./simple.md) if you haven't read it. The thermostat analogy is the spine of this topic.

---

## Mental model: a controller that adjusts `replicas`

HPA is a Kubernetes controller that watches metrics and writes to `.spec.replicas` on your workload (Deployment, StatefulSet, ReplicaSet). It does nothing special to the workload itself — it just changes the replica count, and the normal Deployment controller takes over from there.

The control loop runs every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period` on the controller manager).

---

## HPA v2 API — the current form

HPA v2 (stable since K8s 1.23) replaces the old v1 which only supported CPU. v2 supports multiple metric types:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50       # target: 50% of cpu.request per pod
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"          # per-pod target
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: "my-queue"
      target:
        type: AverageValue
        averageValue: "30"
```

When multiple metrics are configured, HPA computes the desired replica count for **each metric separately** and takes the **maximum** of all desired counts. This ensures no single metric under-provisions the deployment.

---

## The scaling math

```
desired_replicas = ceil(current_replicas × (current_metric_value / target_metric_value))
```

The "current metric value" for a Resource metric (CPU, memory) is the **average** across all pods in the Deployment.

Examples:

| current pods | avg CPU per pod | target CPU | calculation | desired |
|---|---|---|---|---|
| 4 | 80% | 50% | ceil(4 × 80/50) = ceil(6.4) | 7 |
| 7 | 50% | 50% | ceil(7 × 50/50) = ceil(7.0) | 7 |
| 7 | 30% | 50% | ceil(7 × 30/50) = ceil(4.2) | 5 |

HPA has a tolerance of ±10% before acting. If the ratio is between 0.9 and 1.1, it stays put. This prevents constant small adjustments.

---

## The per-pod average gotcha

The most common "HPA isn't scaling" misunderstanding.

`averageUtilization: 50` means target **50% of the pod's cpu.request per pod, averaged across all pods**.

Scenario: your pod has `requests.cpu: 100m` and the actual CPU usage is 200m (2× the request). HPA sees this as 200% utilization per pod and scales aggressively.

But: your pod has `requests.cpu: 2000m` (2 cores) and actual usage is 400m. HPA sees 20% utilization — well below the 50% target. No scale-up, even if the service is slow, because HPA thinks there's plenty of headroom.

The fix: **set CPU requests accurately** — close to typical usage, not inflated for safety headroom. HPA depends on requests being a meaningful number.

What if you can't use CPU? Use custom metrics (request rate, queue depth) which measure real load independently of resource requests.

---

## Scale-up vs. scale-down behavior

By default:
- **Scale-up**: can scale by any amount immediately. Stabilization window: 0 seconds (fast response).
- **Scale-down**: stabilization window of **300 seconds** (5 minutes). HPA records the highest desired replica count in the last 300 seconds and uses that. Prevents thrashing during brief metric drops.

You can customize with the `behavior` field (HPA v2):

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30      # wait 30s before scaling up
      policies:
      - type: Pods
        value: 4
        periodSeconds: 15                 # max: add 4 pods per 15 seconds
      - type: Percent
        value: 100
        periodSeconds: 60                 # max: double pods per minute
      selectPolicy: Max                   # use whichever policy allows more
    scaleDown:
      stabilizationWindowSeconds: 120     # only scale down if low for 2 minutes
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60                 # max: remove 2 pods per minute
      selectPolicy: Min                   # use whichever policy allows less (safer)
```

Key insight on `scaleDown`: in the stabilization window, HPA tracks the **maximum** desired replica count over the window. It will only scale below the historical max when the window clears. This prevents:
1. HPA scaling down during a momentary dip in traffic
2. Then traffic spikes → cold start delay → user impact

**Real-world calibration**: for web services, the defaults (5min scale-down window) are usually fine. For latency-sensitive services, lower it. For expensive GPU pods, raise it — you want to be sure load is truly gone before removing pods.

---

## Metrics pipelines

### CPU and memory: metrics-server

`metrics-server` aggregates resource usage from the kubelet every 15 seconds. It's the only supported source for `type: Resource` metrics (CPU, memory).

Verify it's working:

```bash
kubectl top nodes
kubectl top pods
```

If these commands fail, metrics-server isn't working.

### Custom pod metrics: Prometheus Adapter

The Prometheus Adapter translates Prometheus metrics into the K8s Custom Metrics API. HPA can then use these with `type: Pods`.

Example: expose `http_requests_per_second` per pod to HPA via Prometheus Adapter. The adapter scrapes your Prometheus for this metric and makes it available at the custom metrics API endpoint.

Install: https://github.com/kubernetes-sigs/prometheus-adapter

Configuration (simplified):
```yaml
# In Prometheus Adapter config
rules:
- seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total"
    as: "${1}_per_second"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

### External metrics and KEDA

For event-driven scaling — Kafka consumer lag, SQS queue depth, Pub/Sub message count, custom metrics from external systems — the native Prometheus Adapter gets complex. **KEDA** (Kubernetes Event-Driven Autoscaler) is the modern alternative.

KEDA acts as a metrics server and creates/manages HPA objects automatically. You define a `ScaledObject`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: my-consumer
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      topic: my-topic
      consumerGroup: my-group
      lagThreshold: "10"         # scale up when lag > 10 messages per replica
```

KEDA handles the metrics API translation under the hood. KEDA is now a CNCF graduated project and is the recommended approach for event-driven HPA.

KEDA can also scale **to zero** (minReplicas: 0) and back — something native HPA doesn't support (HPA minimum is 1 replica).

---

## Troubleshooting: the full `describe hpa` checklist

```bash
kubectl describe hpa my-api
```

Look at each section:

```
Name:           my-api
...
Reference:      Deployment/my-api
Metrics:
  resource cpu on pods  (as a percentage of request):
    current:  43% (as a percentage of request)
    target:   50%
Min replicas:   2
Max replicas:   20
Deployment pods:        4 current / 4 desired
...
Conditions:
  Type              Status  Reason            Message
  ----              ------  ------            -------
  AbleToScale       True    SucceededGetScale Successfully got the scale
  ScalingActive     True    ValidMetricFound  the HPA was able to successfully calculate a replica count
  ScalingLimited    False   DesiredWithinRange the desired replica count is within the acceptable range
Events:             <none>
```

**Conditions to check**:

| Condition | Status | What it means |
|---|---|---|
| `AbleToScale: False` | Bad | Can't read current replica count. Check Deployment exists. |
| `ScalingActive: False` | Bad | Can't compute scaling decision. Metrics unavailable — check metrics-server. |
| `ScalingLimited: True` | Info | HPA wants to scale but is blocked by `minReplicas` or `maxReplicas`. |

**Check the TARGETS column**:

```bash
kubectl get hpa
# NAME      REFERENCE         TARGETS     MINPODS   MAXPODS   REPLICAS
# my-api    Deployment/my-api 43%/50%     2         20        4
# my-api    Deployment/my-api <unknown>   2         20        4   ← bad
```

`<unknown>` in TARGETS: metrics-server is not installed, or the pod has no `requests.cpu` set (can't compute utilization without a request as baseline).

---

## Common issues checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| `TARGETS: <unknown>` | metrics-server not installed or not working | `kubectl get pods -n kube-system | grep metrics-server` |
| CPU shows 0% but app is slow | CPU request is 0 or not set | Set `resources.requests.cpu` on all containers |
| HPA not scaling up despite high load | Already at `maxReplicas` | Check `ScalingLimited: True` condition; increase `maxReplicas` |
| HPA not scaling down after load drops | 5-minute stabilization window | Normal behavior; or check `scaleDown.stabilizationWindowSeconds` |
| HPA and VPA fighting | Both adjusting same resource | Don't use VPA on the same metric HPA is using; use VPA for memory, HPA for CPU |

---

## The interview answer in 60 seconds

> "First thing I check: `kubectl describe hpa my-api`. Look at Conditions. Is `ScalingActive: True`? If False, HPA can't make decisions — usually because metrics aren't available.
>
> Second: the TARGETS column. `<unknown>` means metrics-server isn't running or the pods have no `resources.requests.cpu` set. HPA calculates utilization as actual usage divided by the requested amount. No request = no ratio = HPA can't compute anything.
>
> Third: is `maxReplicas` already reached? HPA will show `ScalingLimited: True` if it wants more replicas but can't go above the max.
>
> Fourth: the stabilization window. Scale-up has a 0-second window by default (fast). Scale-down has a 5-minute window. If load just spiked briefly and came back, HPA is waiting for sustained high load.
>
> If none of these apply and I need event-driven scaling (Kafka lag, SQS depth), I'd look at KEDA rather than the native HPA — KEDA supports external metrics natively and can scale to zero."

---

## Self-test

### 1. Your HPA target is CPU 50%, you have 4 pods, average CPU is 80%. What replica count does HPA target?

**Reference answer:** `ceil(4 × 80/50) = ceil(6.4) = 7`. HPA scales to 7 pods. As load distributes across more pods, CPU per pod drops toward 50%.

### 2. HPA shows `TARGETS: <unknown>`. What's likely wrong and how do you fix it?

**Reference answer:** Either metrics-server isn't installed/running, or the pods in the Deployment have no `resources.requests.cpu` (HPA uses utilization = actual/requested, no request = no denominator). Fix: verify `kubectl top pods` works (metrics-server check); add `resources.requests.cpu` to the pod spec.

### 3. Your HPA shows the right metric value and the right desired replica count, but replicas aren't changing. Why?

**Reference answer:** Two possibilities. First: already at `maxReplicas` — check `ScalingLimited: True`. Second: within the ±10% tolerance band (ratio between 0.9 and 1.1) — HPA deliberately doesn't act on small deviations to prevent constant small adjustments. A third possibility: the Deployment itself is paused (`kubectl rollout pause`).

### 4. When would you use KEDA instead of native HPA?

**Reference answer:** When the scaling signal is event-driven rather than resource-based. Kafka consumer lag: scale workers based on messages behind, not CPU. SQS queue depth: scale processors based on messages waiting, not memory. GitHub Actions runner pool: scale based on queued workflow jobs. KEDA also enables scale-to-zero (minReplicas: 0) which native HPA doesn't support. For CPU/memory-only autoscaling, native HPA with metrics-server is simpler and sufficient.

---

## Further reading / watching

- **Kubernetes Docs — HPA**: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- **HPA v2 behavior field**: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#configurable-scaling-behavior
- **KEDA**: https://keda.sh/ — especially the scalers catalog
- **Prometheus Adapter**: https://github.com/kubernetes-sigs/prometheus-adapter
- **Kubernetes HPA design doc** — if you want to understand the controller internals: search "KEP horizontal pod autoscaling"

---

## The 4 dimensions (senior framing)

- **Tech**: HPA v2 API; ceil-based scaling formula with per-pod average metric; ±10% tolerance dead zone; `behavior` field for scale-up/scale-down velocity and stabilization; metrics-server for CPU/memory; Prometheus Adapter for custom pod metrics; KEDA for external/event-driven metrics and scale-to-zero.
- **People**: Developers setting HPA often don't understand the per-pod average math. "My pods are 80% CPU, why isn't HPA scaling?" — because requests.cpu is set to 4 cores and actual usage is 400m, so utilization is only 10%. Train devs to verify HPA targets and actual metric values, not just look at `kubectl top pods`. Document the requests-vs-limits setting as a prerequisite for HPA.
- **CI/CD**: Validate HPA configuration in CI before deploying. A HPA pointing at a non-existent metric or with `maxReplicas` lower than `minReplicas` will silently fail. In GitOps: verify that `scaleTargetRef` matches the Deployment name in the same PR. Consider adding a smoke test: deploy to a test namespace, check `kubectl get hpa` TARGETS is not `<unknown>`.
- **Operations**: Alert on `ScalingActive: False` — an HPA that can't scale is worse than no HPA (gives false confidence). Alert on replicas frequently hitting `maxReplicas` — either load is growing and maxReplicas should be raised, or there's a traffic anomaly. Track HPA scale events over time (from events API or a metrics-based signal) to understand scaling frequency and velocity.
