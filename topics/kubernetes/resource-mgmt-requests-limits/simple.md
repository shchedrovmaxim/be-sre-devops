# Requests, limits, and QoS — the simple version (the restaurant reservation analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **Requests are what you're guaranteed. Limits are the most you can use. QoS class is determined by how these compare — and under memory pressure, the lower your QoS class, the sooner you get evicted.**

---

## The restaurant reservation

Think of a node as a restaurant with a fixed number of seats and kitchen capacity.

| In the restaurant world | In the K8s world |
|---|---|
| You book a table for 2 → guaranteed a spot | `requests.cpu/memory` → guaranteed on a node with that capacity |
| You can actually seat 4 if tables are free ("burst") | Pod can use more than its request if the node has idle resources |
| The restaurant's max capacity (fire code) | `limits.cpu/memory` → pod cannot exceed this |
| A reservation-only customer has a confirmed seat | **Guaranteed QoS** — both request = limit |
| A customer with a reservation but allowed to stay longer | **Burstable QoS** — has a request, limit > request |
| Walk-in with no reservation | **BestEffort QoS** — no requests or limits set at all |
| Restaurant fills up → walk-ins leave first | Memory pressure → BestEffort pods evicted first |
| Then those who booked but went over | Then Burstable pods evicted (highest over their request) |
| Reserved customers last to leave | Guaranteed pods evicted last |

---

## The two confusing concepts

### 1. CPU limits throttle, memory limits kill

**CPU is compressible**: if a pod tries to use more than its CPU limit, the kernel throttles it (slows it down). The pod stays running, just slower. This is why CPU throttling can be invisible — the pod looks healthy but requests are slow.

**Memory is not compressible**: if a pod uses more memory than its limit, the kernel OOM-kills the process. The pod restarts (if `restartPolicy: Always`). OOMKilled is visible in `kubectl describe pod`.

```bash
# Check if pod was OOMKilled:
kubectl describe pod my-pod
# Containers:
#   app:
#     Last State:  Terminated
#       Reason:   OOMKilled
#       Exit Code: 137
```

This asymmetry is why **CPU limits are controversial** (they cause silent throttling) and **memory limits are essential** (you want to control OOM blast radius).

### 2. The 3 QoS classes — and why they matter for eviction

When the node runs low on memory, kubelet has to evict pods. The order is:

| QoS class | When | Gets evicted |
|---|---|---|
| **BestEffort** | No requests or limits set on any container | First — they promised nothing, they get nothing |
| **Burstable** | Request set, or request < limit (any container) | Second — the one that's farthest over its memory request goes first |
| **Guaranteed** | Every container has request == limit (both CPU and memory) | Last — they reserved exactly what they use |

**The interview insight**: within the Burstable class, eviction order is by how far over the memory request each pod is (as a ratio). A pod using 10× its memory request is evicted before one using 2× its request.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is `requests.cpu`? | What K8s uses for scheduling (fits pods to nodes); sets the cgroup CPU weight |
| What is `limits.cpu`? | Hard ceiling on CPU; throttling happens if exceeded (pod stays running) |
| What is `requests.memory`? | Scheduling guarantee; OOM score baseline |
| What is `limits.memory`? | Hard ceiling on memory; OOMKill if exceeded |
| What happens when CPU limit is exceeded? | Pod is throttled (slower), not killed |
| What happens when memory limit is exceeded? | Container is OOMKilled and restarted |
| What's the eviction order under memory pressure? | BestEffort → Burstable (highest overage first) → Guaranteed |
| Why is setting no memory limit dangerous? | A memory leak can crash the entire node instead of just one pod |
| Why do some people say "don't set CPU limits"? | CPU limits cause throttling even when the node has idle CPU — wasteful |

---

## Self-test (one question — the killer one)

Out loud:

> **"Two pods on the same node, both Burstable, node hits memory pressure. Who gets evicted first and why?"**

**Reference answer (intuitive version):**

"The one that's proportionally furthest over its memory request. Say Pod A requested 500Mi and is using 1Gi — it's at 200% of its request. Pod B requested 256Mi and is using 400Mi — it's at 156% of its request. The kubelet evicts Pod A first because it's more over-provisioned relative to what it promised.

Both are Burstable (they set requests but are exceeding them). Guaranteed pods — where request equals limit — would be last to be evicted. BestEffort pods — with no requests at all — would be first, before either of these Burstable pods."

---

## Further reading / watching

- **Kubernetes Docs — Resource Management**: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- **Kubernetes Docs — Node-pressure Eviction**: https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/
- **"Stop Using CPU Limits" by Tim Hockin** — the case against CPU limits: search for this talk or the associated blog post
- **QoS for Pods**: https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/

---

## Next: the deep-dive

When the restaurant analogy clicks and you understand the three QoS classes, jump to [`resource-mgmt-requests-limits.md`](./deep-dive.md). The deep-dive covers:

- How `requests` maps to Linux cgroup mechanics (cpu.weight, memory accounting)
- How `limits` maps to cgroup `cpu.max` and `memory.max`
- The full eviction order: kube-reserved, system-reserved, eviction thresholds
- `oom_score_adj` — the kernel's OOM priority per container
- The CPU limits antipattern debate in detail
- Node-level vs. cgroup-level enforcement
- 4 self-test drills + 4-dimensions framing
