# Requests, limits, QoS classes, and eviction — deep-dive

> **Goal**: by the end you can answer the killer question — **"Two pods on the same node, both Burstable, node hits memory pressure. Who gets evicted first and why?"** — naming the eviction order algorithm, QoS classes and how they're assigned, `oom_score_adj`, the difference between cgroup-level OOM and node-level eviction, and the CPU limits antipattern debate.

> Start with the [simple version](./resource-mgmt-requests-limits-simple.md) if you haven't read it. The restaurant-reservation analogy is the spine of this topic.

---

## Mental model: requests are scheduler hints + cgroup weights. Limits are cgroup hard caps.

When you write:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

You're making two separate contracts:

1. **Requests → the scheduler**. The scheduler finds a node that has at least 250m of allocatable CPU and 256Mi of allocatable memory. The node is considered to have this capacity "consumed" even if the pod isn't using it yet. Requests are the scheduling reservation.

2. **Requests + limits → the kubelet + kernel cgroups**. When the pod runs, kubelet sets cgroup parameters on the pod:
   - CPU: `cpu.weight` (for v2 cgroups) derived from the CPU request. Limits set `cpu.max` (bandwidth throttle).
   - Memory: `memory.limit_in_bytes` (for limits). Requests affect OOM scoring.

---

## CPU mechanics in cgroups

### Requests → `cpu.weight`

In cgroup v2, CPU shares are proportional. If Pod A has `requests.cpu: 1000m` and Pod B has `requests.cpu: 500m`, Pod A gets 2× the CPU time when the node is contended.

Under idle conditions, both pods can use as much CPU as they want (up to their limit). Requests only matter when CPUs are fully saturated — they determine which pod gets priority.

Formula for `cpu.weight`:
```
weight = max(1, round(((cpu_request_millicore) * 1024) / 1000))
```

So 1000m → weight ~1024, 100m → weight ~102.

### Limits → `cpu.max`

`cpu.max` in cgroup v2 enforces a hard upper bound. If a container exceeds its CPU limit over a scheduling period (typically 100ms), it's throttled until the next period.

**Throttling is silent**: the container keeps running, but it's slower. A pod at its CPU limit shows 100% throttle ratio in metrics. This can look like "the app is slow" without any obvious failure.

Check CPU throttling:
```bash
# Via metrics:
container_cpu_cfs_throttled_seconds_total  # Prometheus
container_cpu_cfs_throttled_periods_total

# Via kubectl:
kubectl top pods       # shows current usage; compare to limits
```

### The CPU limits antipattern

**The argument against setting CPU limits** (Tim Hockin, others):

If a node has 4 CPUs and Pod A requests 250m with limit 500m, Pod A can only use 500m even if 3.75 CPUs are idle. Throttled at 500m while the node has massive headroom.

For latency-sensitive workloads (web servers, APIs), CPU throttling at limit causes tail-latency spikes that look like application slowness or timeouts.

**The counter-argument for limits**:

Without limits, a misbehaving pod (bug, memory leak's accompanying CPU burn) can consume all node CPU and starve neighbors. LimitRange can enforce defaults cluster-wide.

**Practical recommendation**: set CPU requests accurately; consider not setting CPU limits for latency-sensitive workloads in controlled environments. Always set memory limits — memory is not compressible and an uncapped leak crashes the node.

---

## Memory mechanics in cgroups

### Limits → `memory.limit_in_bytes`

If a container tries to allocate memory beyond its limit, the kernel OOM-kills the process. This triggers a container restart (if `restartPolicy: Always`). The pod shows `OOMKilled` in `kubectl describe pod`.

Exit code for OOMKill: **137** (128 + SIGKILL signal 9).

```bash
kubectl describe pod my-pod
# Containers:
#   app:
#     Last State: Terminated
#       Reason:   OOMKilled
#       Exit Code: 137
```

### Requests → `oom_score_adj`

The Linux kernel has an OOM scorer. When the system runs low on memory and the kernel's OOM killer fires, it kills the process with the highest OOM score. Kubernetes configures `oom_score_adj` per container to reflect its QoS class:

| QoS class | `oom_score_adj` | Kernel OOM priority |
|---|---|---|
| **Guaranteed** | -998 | Very unlikely to be OOM-killed by kernel |
| **Burstable** | Proportional to usage above request, in range 2–999 | Moderate priority |
| **BestEffort** | 1000 | First to be killed by kernel OOM |

Calculation for Burstable:
```
oom_score_adj = 1000 - (1000 × (memory_request / node_capacity))
```

A pod that requested 1% of node memory has `oom_score_adj ≈ 990` — nearly as likely to be killed as BestEffort. A pod that requested 50% of node memory has `oom_score_adj ≈ 500`.

This means: **within the Burstable class, pods with small requests relative to node memory get higher OOM scores and are killed sooner** — even if two pods are using the same absolute amount of memory.

This is the **cgroup-level OOM** (kernel kills a single container in a pod). It's different from **node-level eviction** (kubelet evicts a whole pod).

---

## Node-level eviction — kubelet's role

The kernel OOM killer reacts after memory is exhausted. The kubelet's eviction manager acts **proactively**, before memory is fully exhausted.

Kubelet monitors node memory and evicts pods when memory drops below configured thresholds. This is preferable to kernel OOM — it's controlled, gives the pod its grace period, and doesn't kill random kernel processes.

### The eviction hierarchy (memory)

Node memory is carved into zones:

```
Total node memory
├── kube-reserved           (set in kubelet config, for K8s system components)
├── system-reserved         (for non-K8s system processes: sshd, systemd, etc.)
└── eviction-hard threshold (the trigger point)
    └── Allocatable memory  (what pods can use)
```

When pod memory usage + kube-reserved + system-reserved exceeds `eviction-hard`, kubelet starts evicting pods.

Example kubelet config:
```yaml
# /etc/kubernetes/kubelet-config.yaml
evictionHard:
  memory.available: "200Mi"     # evict when less than 200Mi free
  nodefs.available: "10%"       # evict when disk is 90% full
  imagefs.available: "15%"
evictionSoft:
  memory.available: "500Mi"     # warn at 500Mi free but wait for evictionSoftGracePeriod
evictionSoftGracePeriod:
  memory.available: "1m30s"     # give pods 90 seconds to self-correct before evicting
kubeReserved:
  cpu: "200m"
  memory: "250Mi"
systemReserved:
  cpu: "200m"
  memory: "250Mi"
```

### Eviction order — the full algorithm

When eviction triggers, kubelet selects victims in this order:

1. **BestEffort pods** — all of them first, ordered by memory usage (highest first)
2. **Burstable pods** — ordered by "memory usage above request as a percentage"
   - Pod using 500% of its request is evicted before one using 200%
3. **Guaranteed pods** — last resort; ordered by memory usage (highest first)

Within each class, pods consuming the most memory relative to their request are evicted first.

The full signal:
```
eviction_score = pod_memory_usage / pod_memory_request
```

Higher score → evicted sooner (within same QoS class).

---

## QoS class assignment

K8s automatically assigns a QoS class based on the pod's resource spec. You cannot set it directly.

### Guaranteed

Every container in the pod has:
- `requests.cpu` set
- `requests.memory` set
- `limits.cpu` == `requests.cpu`
- `limits.memory` == `requests.memory`

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"         # must equal request
    memory: "512Mi"     # must equal request
```

Check:
```bash
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
# Guaranteed
```

### Burstable

At least one container has a request or limit set, but it's NOT Guaranteed.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"          # limit > request → Burstable
    memory: "512Mi"
```

### BestEffort

No containers have any requests or limits.

```yaml
# No resources block at all, or empty
resources: {}
```

BestEffort pods get scheduled wherever space appears to be available (K8s assumes 0 resource usage). They're the first to be killed under pressure.

---

## Complete annotated YAML (best practices)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-api:1.0
        resources:
          requests:
            cpu: "250m"         # matches typical usage; used for scheduling + cpu.weight
            memory: "256Mi"     # accurate estimate; used for scheduling + oom_score_adj
          limits:
            # cpu limit intentionally omitted for latency-sensitive service
            # (prevents silent CPU throttling when node has headroom)
            memory: "512Mi"     # always set memory limit (blast radius control)
```

---

## The interview answer in 60 seconds

> "When the node hits memory pressure, kubelet's eviction manager fires before the kernel OOM killer. It evicts pods in three tiers.
>
> First, BestEffort pods — no requests or limits, they promised nothing. All evicted first, ordered by memory usage.
>
> Second, Burstable pods — they have requests set but are using more than they requested. Eviction order within Burstable is by 'how far over their request are they' — the pod using 500% of its memory request is evicted before the one using 200%.
>
> So between our two Burstable pods: the one with the higher ratio of actual-usage-to-requested-memory gets evicted first.
>
> Third and last, Guaranteed pods — where request equals limit. They reserved exactly what they use.
>
> This is kubelet-level eviction. At the kernel level, the OOM killer uses `oom_score_adj` which kubelet sets per-container. Guaranteed pods get -998 (nearly immune), BestEffort get 1000 (first killed). Burstable are in between, proportional to how small their request is relative to the node."

---

## Self-test

### 1. Two Burstable pods. Pod A: requested 200Mi, using 600Mi. Pod B: requested 500Mi, using 800Mi. Which gets evicted first?

**Reference answer:** Pod A. It's using 300% of its request. Pod B is using 160% of its request. The eviction score is usage/request — higher score = evicted first. Even though Pod B is using more absolute memory, Pod A is more over-provisioned relative to its promise.

### 2. What's the difference between a kernel OOM kill and a kubelet eviction?

**Reference answer:** Kernel OOM kill: happens when the kernel's memory subsystem runs out of memory. It uses `oom_score_adj` to rank processes and kills the highest-scored one (typically a single container). Reactive, no warning. Kubelet eviction: kubelet monitors memory proactively and evicts whole pods (with grace period) when free memory drops below `evictionHard` threshold. Kubelet eviction is preferable — it gives pods time to drain, is ordered by QoS class, and prevents kernel OOM from killing arbitrary processes.

### 3. Why do some engineers argue you shouldn't set CPU limits?

**Reference answer:** CPU limits in K8s map to cgroup `cpu.max`, which enforces a hard bandwidth cap per scheduling period (100ms). When a pod hits its CPU limit, it gets throttled even if the node has plenty of idle CPU — the unused headroom isn't shared. For latency-sensitive workloads (web servers, APIs), this creates tail-latency spikes that look like application slowness. The node is at 30% CPU but pods are throttled. The counter-argument: without limits, a buggy pod can monopolize node CPU and starve neighbors. Recommendation: set requests accurately; evaluate whether limits are necessary per workload type.

### 4. How does `kube-reserved` affect scheduling and eviction?

**Reference answer:** `kube-reserved` (and `system-reserved`) carve out memory and CPU from the node's "Allocatable" capacity. The scheduler places pods based on Allocatable, not total capacity. So if a 16Gi node has `kube-reserved: memory: 1Gi` and `system-reserved: memory: 500Mi`, only ~14.5Gi is allocatable to pods. This means requests across all pods on the node will never sum above ~14.5Gi, leaving headroom for K8s system components. Eviction thresholds (`evictionHard`) are applied on top of this — kubelet evicts pods when free memory (from Allocatable) drops below the threshold.

---

## Further reading / watching

- **Kubernetes Docs — Resource Management**: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- **Kubernetes Docs — Node-pressure Eviction**: https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/
- **Kubernetes Docs — Pod QoS**: https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/
- **"Stop Using CPU Limits" by Tim Hockin** — search this title; the blog post and KubeCon talk are both worth reading
- **cgroup v2 CPU weight**: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html

---

## The 4 dimensions (senior framing)

- **Tech**: Requests → scheduler accounting + cgroup weight (CPU) + oom_score_adj baseline (memory). Limits → cgroup hard cap (cpu.max for CPU throttle, memory.limit_in_bytes for OOMKill). QoS classes: Guaranteed (request==limit), Burstable (request<limit or partial), BestEffort (none). Eviction order: BestEffort → Burstable (highest usage/request ratio) → Guaranteed. `kube-reserved` + `system-reserved` + `eviction-hard` define the allocatable boundary. Kernel `oom_score_adj` handles cgroup-level OOM independently.
- **People**: Developers often set requests to 0 (or leave them unset) "to avoid waste." Educate: unset requests → BestEffort QoS → first evicted under pressure. Set requests close to actual p99 usage. Use VPA recommendations to calibrate. Also explain: memory limit is not "the amount your app uses" — it's the OOM blast radius. Set it 2× the typical usage.
- **CI/CD**: Enforce `LimitRange` in every namespace with sensible defaults so pods without resource specs get sane values. Kyverno policies that reject pods without `requests.memory` — BestEffort pods are production hazards. Add VPA in recommendation mode to non-production namespaces so devs see what requests should be.
- **Operations**: Alert on OOMKilled containers (exit code 137 in container status). Alert on sustained CPU throttle ratio > 50% (the container is CPU-constrained but not visibly failing). Track per-node memory pressure (kubelet exposes `node_memory_MemAvailable_bytes`). Quarterly audit: `kubectl top pods --all-namespaces` to find pods at/near their limits — potential OOM candidates.
