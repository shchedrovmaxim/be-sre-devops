# PriorityClass + preemption — deep-dive

> **Goal**: by the end you can answer the killer question — **"Explain how priorityClass + preemption actually work — when would a high-priority pod evict a lower-priority one?"** — naming the PriorityClass spec, how the scheduler selects victims, the eviction sequence, the built-in system classes, the PDB-doesn't-block-preemption trap, and cascading preemption risks.

> Start with the [simple version](./simple.md) if you haven't read it. The airport boarding-priority analogy is the spine of this topic.

---

## Mental model: priority for the queue, preemption for full clusters

Two separate but related mechanisms:

1. **Priority**: when multiple pods are in the scheduling queue simultaneously, higher-priority pods are scheduled first. This matters when the cluster has capacity but the queue is long.

2. **Preemption**: when a high-priority pod has no schedulable node, the scheduler proactively evicts lower-priority pods to make room. This is a last resort — the scheduler first exhausts all normal scheduling options.

These are separate. You can use priority without preemption (`preemptionPolicy: Never`). Preemption requires both: a pending pod that can't be scheduled, AND lower-priority pods that could be evicted to make room.

---

## PriorityClass spec

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-production
value: 1000000               # integer; higher = higher priority; no upper bound except 1B
globalDefault: false         # if true, pods without a priorityClassName get this value
                             # only one PriorityClass can have globalDefault: true
preemptionPolicy: PreemptLowerPriority   # or Never
description: "For production serving workloads that must not be preempted by batch jobs"
```

And use it in a pod/Deployment:

```yaml
spec:
  priorityClassName: high-priority-production
  containers:
  - name: app
    ...
```

**Key fields**:

- `value`: the priority integer. Pods with higher values are scheduled first in the queue. The scheduler uses this value when selecting preemption victims (lower value = evicted first).
- `globalDefault: true`: pods without `priorityClassName` get this value. Useful for giving all pods a non-zero baseline. **Only one class can be globalDefault.** Setting a second one errors.
- `preemptionPolicy: Never`: this pod's priority is used for queue ordering only. This pod will never preempt others. Useful for batch jobs that should yield to production but shouldn't actively kick anyone off.
- `description`: human-readable. Put your runbook URL here.

---

## Built-in system PriorityClasses

Kubernetes ships two:

| Name | Value | Used for |
|---|---|---|
| `system-node-critical` | 2000001000 | Critical pods that must run on a node for it to function: kubelet, kube-proxy, CNI agents |
| `system-cluster-critical` | 2000000000 | Critical pods that support the cluster but aren't strictly node-local: CoreDNS, metrics-server |

These are the highest-priority classes in any cluster. No user-defined class should try to exceed 2,000,000,000.

If you're setting up your own priority hierarchy:

```
system-node-critical:       2,000,001,000  (built-in)
system-cluster-critical:    2,000,000,000  (built-in)
high (production serving):  1,000,000      (user-defined)
medium (staging/background): 100,000       (user-defined)
low (batch/analytics):         1,000       (user-defined)
default (no class set):            0       (built-in default)
```

---

## How the scheduler selects preemption victims

When a high-priority pod P is pending and can't be scheduled:

**Step 1**: Find "potential nodes" — nodes where, if some subset of pods were removed, P could fit.

The scheduler evaluates each node: "if I removed the subset of pods with priority < P.priority that frees enough resources, would P fit (including resource requests, affinity, taints)?"

**Step 2**: Rank the potential nodes. The scheduler prefers nodes where preemption causes the least disruption:
- Fewest pods to evict
- Lowest combined priority of evicted pods
- If multiple pods need to be evicted, prefer nodes where evicted pods have already been pending long

**Step 3**: On the chosen node, the scheduler evicts the selected victim pods. Victims get their full `terminationGracePeriodSeconds` to drain.

**Step 4**: While waiting for victims to drain, the scheduler marks the node as "nominated node" for P. P doesn't immediately get scheduled — it waits for victims to terminate and resources to free up.

**Step 5**: After victims terminate and resources are available, the scheduler runs the full scheduling cycle again for P. If no other high-priority pod took the freed resources in the meantime, P gets scheduled.

### What victim selection looks like

Scenario: Node has 4 CPUs. Three pods running:

| Pod | Priority | CPU |
|---|---|---|
| web-backend | 1,000,000 | 1 CPU |
| analytics-job | 1,000 | 2 CPUs |
| batch-import | 500 | 1 CPU |

High-priority pod P (priority 2,000,000) needs 2 CPUs to run. The node only has 0 free.

Scheduler considers evicting:
- Only `analytics-job` (priority 1,000, 2 CPUs) → frees exactly 2 CPUs. P fits. ✓
- Or evicting `analytics-job` + `batch-import` → 3 CPUs freed, more than needed. Less efficient.

The scheduler evicts `analytics-job` only — minimum disruption.

---

## The eviction sequence during preemption

1. Scheduler selects victim pods on the target node.
2. Scheduler sets "nominated node" on the pending pod P (visible in `kubectl get pod P -o yaml → .status.nominatedNodeName`).
3. Victims receive graceful termination: preStop hooks run, SIGTERM sent, `terminationGracePeriodSeconds` honored.
4. Victims terminate, resources free up.
5. Scheduler re-evaluates P. If resources are available on the nominated node (no other pod grabbed them), P is scheduled there.
6. P starts on the node.

The grace period respect is important: **preemption does not bypass `terminationGracePeriodSeconds`**. Evicted pods get to drain. For long-running jobs, this means preemption completion can take minutes.

**Gotcha**: between step 4 and 5, other pods (at any priority level) can grab the freed resources. P has no exclusive claim during the waiting period. The "nominated node" is a hint, not a reservation.

---

## PDB does NOT block preemption — the common interview trap

This is the most important thing to know:

**PodDisruptionBudget only governs voluntary disruptions** — `kubectl drain`, cluster autoscaler, node upgrade tools. These go through the eviction API, which checks PDB.

Preemption does NOT go through the eviction API. The scheduler directly evicts victim pods. PDB is not consulted.

So if your job pods have a PDB saying `minAvailable: 3` and a high-priority pod needs to evict 2 of your job pods — the eviction happens. PDB does not help.

The only protection against preemption is having a high enough PriorityClass value. If your pod must not be preempted, give it a priority higher than any pod that might preempt it.

---

## Cascading preemption — the subtle danger

Scenario: Node A is running three medium-priority pods. A high-priority pod is pending. The scheduler evicts two medium-priority pods from Node A.

Those two evicted medium-priority pods are now pending. They try to reschedule. But the cluster is full. They might now preempt low-priority pods on Node B.

This is **cascading preemption**: one preemption event triggers a chain of secondary preemptions across the cluster. The original intent was to schedule one high-priority pod; the result is significant cluster disruption.

How to avoid it:
- Set `preemptionPolicy: Never` on medium-priority classes that should yield to high-priority but shouldn't cause cascades.
- Keep priority tiers well-separated (don't have many classes at similar values).
- Ensure low-priority (batch) pods are truly OK being evicted — they should be restartable and idempotent.

---

## Preemption vs. node-pressure eviction

These are frequently confused:

| | Preemption | Node-pressure eviction |
|---|---|---|
| Triggered by | A pending pod can't be scheduled | Node memory/disk pressure |
| Who decides | Scheduler | kubelet |
| Victim selection | Based on priority | Based on QoS class + resource overage |
| PDB respected | No | No (node-pressure eviction) |
| Grace period | Yes | Yes |
| Pod after eviction | Rescheduled by controller | Rescheduled by controller |

For node-pressure eviction, see [resource-mgmt-requests-limits.md](../resource-mgmt-requests-limits/deep-dive.md).

---

## Complete annotated YAML

```yaml
# PriorityClasses for a typical production cluster
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 1000000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Core production serving traffic. Preempts batch workloads. Do not use for batch/analytics."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-standard
value: 10000
preemptionPolicy: Never        # yields to production but doesn't kick others off nodes
globalDefault: false
description: "Batch analytics and reporting jobs. Will be preempted by production-critical."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-workload
value: 1000
globalDefault: true            # all pods without priorityClassName get this value
preemptionPolicy: PreemptLowerPriority
description: "Default for all workloads that haven't declared a priority."
---
# Using priorities in a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  template:
    spec:
      priorityClassName: production-critical
      containers:
      - name: app
        image: payment-api:1.0
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            memory: "1Gi"
```

---

## The interview answer in 60 seconds

> "PriorityClass assigns an integer priority to pods. Higher number = higher priority. Two effects: in the scheduling queue, higher-priority pods are considered before lower-priority ones. And when a high-priority pod can't be scheduled anywhere because the cluster is full, preemption kicks in.
>
> The scheduler finds nodes where, if lower-priority pods were removed, the high-priority pod would fit. It selects the node that requires the least disruption — fewest evictions, lowest-priority victims. Victim pods get their full termination grace period. The high-priority pod gets a 'nominated node' hint and is scheduled after victims drain.
>
> The gotcha the interviewer is waiting for: **PDB does NOT block preemption**. PDB governs voluntary disruptions (drain, autoscaler) through the eviction API. Preemption bypasses the eviction API entirely — the scheduler evicts directly. The only protection against preemption is having a higher PriorityClass value.
>
> Secondary gotcha: cascading preemption. Evicting medium-priority pods causes them to become pending; if the cluster stays full, they might preempt low-priority pods on another node. Mitigate with `preemptionPolicy: Never` on intermediate priority classes."

---

## Self-test

### 1. What's the difference between `preemptionPolicy: PreemptLowerPriority` and `preemptionPolicy: Never`?

**Reference answer:** `PreemptLowerPriority` (default): this pod's priority is used for both queue ordering AND for preemption victim selection. If this pod is pending and can't fit, it can evict lower-priority pods. `Never`: priority is used for queue ordering only. If this pod is pending and can't fit, it waits — it does NOT evict other pods. Use `Never` for batch jobs that should yield to production (priority helps queue order) but shouldn't cause cascading evictions if they get bumped off a node.

### 2. A Deployment has a PDB with `minAvailable: 2`. Can preemption still evict its pods?

**Reference answer:** Yes. PDB only blocks voluntary disruptions that go through the eviction API (kubectl drain, cluster autoscaler, node upgrade controllers). Preemption bypasses the eviction API — the scheduler evicts victim pods directly. PDB is not consulted. The only way to prevent preemption is to have a PriorityClass value higher than any pod that might preempt yours.

### 3. What is cascading preemption and how do you prevent it?

**Reference answer:** Cascading preemption: a high-priority pod preempts medium-priority pods. Those medium-priority pods become pending. If the cluster is still full, they in turn preempt lower-priority pods on other nodes. One scheduling event causes a chain of evictions across the cluster. Prevention: set `preemptionPolicy: Never` on intermediate priority classes so evicted pods wait in the queue rather than triggering further preemption. Also: ensure low-priority (batch) workloads are genuinely restartable and idempotent — if they're going to get evicted, they should handle it gracefully.

### 4. When does the high-priority pod actually get scheduled after preemption is decided?

**Reference answer:** Not immediately. Sequence: scheduler marks victims for eviction and sets `nominatedNodeName` on the pending pod. Victims get their full `terminationGracePeriodSeconds` (preStop hooks, SIGTERM, drain). After victims terminate, the scheduler re-runs a full scheduling cycle for the pending pod. If the freed resources are still available (no other pod grabbed them in the meantime), the pod gets scheduled. There is no exclusive reservation between "preemption decided" and "pod scheduled." This means the delay from preemption decision to pod running can be tens of seconds to minutes.

---

## Further reading / watching

- **Kubernetes Docs — Pod Priority and Preemption**: https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/
- **K8s Scheduler source**: the victim selection algorithm is in `pkg/scheduler/framework/plugins/defaultpreemption` — worth a quick read if you want the exact logic
- **"Kubernetes Scheduler Internals" KubeCon talk** — search on YouTube; good for understanding the scheduling cycle
- **PDB vs Preemption clarification**: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budget-limitations

---

## The 4 dimensions (senior framing)

- **Tech**: PriorityClass assigns integer priority; higher = better queue position + lower eviction risk. Preemption = scheduler evicts lower-priority pods to fit a pending high-priority pod. Victim selection: minimum disruption first. Grace period honored. `preemptionPolicy: Never` for non-preempting queue-prioritization. Built-in `system-node-critical` (2000001000) and `system-cluster-critical` (2000000000) for K8s infrastructure pods. PDB does NOT block preemption.
- **People**: Teams running batch jobs alongside production often don't know preemption is happening — their batch jobs mysteriously restart and they think it's a node failure. Surface preemption events in dashboards. Document the priority tier in your runbook. Production teams should know their priority value and whether they can be preempted.
- **CI/CD**: Enforce `priorityClassName` in production namespace via Kyverno or OPA Gatekeeper — production pods without a priority class can be preempted by any other pod that happens to have one. Validate priority class names against the known list in CI — a typo in `priorityClassName` causes the pod to be rejected at admission.
- **Operations**: Alert on `system-node-critical` pods that are pending — these are K8s infrastructure components that couldn't be scheduled, which means the node itself may be degrading. Monitor preemption events (available in the K8s events API: `kubectl get events --field-selector reason=Preempting`). Track the frequency of batch job restarts due to preemption — a sudden spike may indicate production load is stressing the cluster.
