# HorizontalPodAutoscaler — the simple version (the thermostat analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **An HPA watches a metric and adjusts the number of pod replicas to keep that metric at a target value. When the metric is above target, add pods. When below, remove them. Think: thermostat for pod count.**

---

## The thermostat

Your house has a thermostat set to 70°F. The furnace (K8s) adds heat (replicas) when temperature drops below 70°F, and cuts heat when it rises above.

| In the thermostat world | In the K8s world |
|---|---|
| Current temperature (65°F) | Current metric value (85% CPU utilization) |
| Target temperature (70°F) | Target metric value (50% CPU per pod) |
| Furnace turns on → adds heat | HPA adds replicas |
| House warms up → furnace turns off | CPU drops → HPA removes replicas |
| Dead zone (69–71°F) before reacting | Stabilization window before scaling (prevents flapping) |
| Thermostat can't control outdoor weather | HPA can't scale below `minReplicas` or above `maxReplicas` |

The key formula:

```
desired replicas = ceil(current_replicas × (current_metric / target_metric))
```

Example: 4 pods, CPU at 80%, target is 50%:

```
desired = ceil(4 × (80 / 50)) = ceil(6.4) = 7
```

HPA scales up to 7 pods. The new pods share the load, and CPU per pod drops toward 50%.

---

## The two confusing concepts

### 1. The metric is per-pod, not cluster-wide

This is the most common misunderstanding.

Say you set `targetCPUUtilizationPercentage: 50`. This means "target 50% CPU on **each pod**" — not "50% of total cluster CPU."

If you have 4 pods and the average CPU across them is 40%, HPA is happy — it's under 50%. If load spikes and average hits 80%, HPA scales up.

The gotcha: if your pod `requests.cpu` is set very low (say 100m), the math might show 800% utilization even though the pod feels fine. CPU metrics are always relative to the pod's CPU request. Set requests accurately.

### 2. Metrics-server must be installed (for CPU/memory HPA)

HPA doesn't scrape metrics itself. It reads from the metrics pipeline. For CPU and memory, that pipeline is `metrics-server` — a cluster addon that aggregates kubelet resource metrics.

If `metrics-server` isn't installed, `kubectl get hpa` shows `<unknown>` in the TARGETS column and HPA does nothing.

```bash
# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Check what HPA sees
kubectl describe hpa my-api
# Look for "AbleToScale: True" and "ScalingActive: True"
```

For custom metrics (request rate, queue depth, etc.), you need an adapter (Prometheus Adapter or KEDA).

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does HPA actually do? | Adjusts `spec.replicas` on your Deployment/StatefulSet based on metrics |
| What metrics can HPA use? | CPU, memory (via metrics-server); custom (via Prometheus Adapter); external (via KEDA) |
| What formula does HPA use? | `ceil(current_replicas × current_metric / target_metric)` |
| Why is HPA slow to scale down? | Stabilization window (default: 5 minutes) prevents flapping |
| Why is HPA slow to scale up? | Less so — scale-up is fast by default (15 seconds stabilization) |
| What does `<unknown>` in TARGETS mean? | Metrics aren't available — check if metrics-server is installed and running |
| What's KEDA? | Event-driven autoscaler; extends HPA with Kafka lag, SQS depth, etc. |
| Can HPA and VPA (Vertical Pod Autoscaler) coexist? | Not on the same metric — VPA adjusts requests, HPA scales replicas; combine carefully |

---

## Self-test (one question — the killer one)

Out loud:

> **"Your HPA isn't scaling under load. Walk me through what to check, in order."**

**Reference answer (intuitive version):**

"I'd check five things in order. **One**: `kubectl describe hpa my-api` — look at the Conditions. Is `ScalingActive: True`? If it says `False`, the HPA can't make scaling decisions at all, usually because metrics aren't available. **Two**: check the TARGETS column. `<unknown>` means metrics-server isn't installed or isn't scraping this pod. **Three**: check the pod's CPU requests. HPA calculates utilization as `actual_cpu / cpu_request`. If requests aren't set (or are set too high), the math never triggers. **Four**: check `maxReplicas` — maybe HPA is trying to scale but is already at the max. **Five**: check the stabilization window. By default, scale-down has a 5-minute stabilization window. If load just spiked and came back down, HPA is waiting for sustained load. For scale-up, 15 seconds is the default — quicker, but still a delay."

---

## Further reading / watching

- **Kubernetes Docs — HorizontalPodAutoscaler**: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- **HPA Walkthrough**: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
- **KEDA** (event-driven autoscaling beyond CPU/memory): https://keda.sh/
- **Metrics Server**: https://github.com/kubernetes-sigs/metrics-server

---

## Next: the deep-dive

When the thermostat analogy clicks, jump to [`hpa.md`](./deep-dive.md). The deep-dive covers:

- HPA v2 — how it differs from the old v1 API
- The exact scaling math
- Scale-up vs. scale-down behavior with the `behavior` field
- Custom metrics via Prometheus Adapter
- External metrics via KEDA
- The "per-pod average" gotcha (why 80% might not trigger scaling)
- Troubleshooting the full `describe hpa` output
- 4 self-test drills + 4-dimensions framing
