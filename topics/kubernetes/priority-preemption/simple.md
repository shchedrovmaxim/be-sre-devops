# PriorityClass + preemption — the simple version (the airport VIP lounge analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **PriorityClass gives pods a priority number. When a high-priority pod can't be scheduled because the cluster is full, the scheduler evicts lower-priority pods to make room — that's preemption.**

---

## The airport

An airport has a fixed number of seats in the departure lounge. Different ticket classes get different boarding priorities.

| In the airport world | In the K8s world |
|---|---|
| First class ticket | High PriorityClass (e.g., value: 1000000) |
| Economy ticket | Low PriorityClass (e.g., value: 100) |
| Lounge is full, a first-class passenger arrives | Cluster is full, a high-priority pod is pending |
| Gate agent bumps an economy passenger off the plane to make room | Scheduler evicts a lower-priority pod to free up node resources |
| The bumped passenger waits for the next flight | Evicted pod is rescheduled elsewhere (or stays pending) |
| First-class passengers always board before economy | High-priority pending pods get scheduled before lower-priority pending pods |
| "Presidential" lounge — VIPs get their own space, regular VIPs can't take it | `system-node-critical` / `system-cluster-critical` — system pods protected by the highest built-in priorities |

The key insight: priority doesn't help if there's capacity. Priority only matters when the cluster is **full** and the scheduler needs to decide who stays and who goes.

---

## The two confusing concepts

### 1. Priority for scheduling queue vs. preemption for eviction — they're different things

**Priority without preemption**: when multiple pods are pending in the scheduling queue, higher-priority pods are considered first. If there's room for one, the high-priority pod gets it.

**Preemption**: when a high-priority pod can't fit anywhere, the scheduler looks for nodes where it could fit *if* some lower-priority pods were removed. It evicts those pods and schedules the high-priority pod.

You can disable preemption while keeping priority:

```yaml
spec:
  preemptionPolicy: Never    # priority still used for queue ordering; no eviction
```

Use `Never` for batch/analytics workloads that should yield to real-time pods in the queue but shouldn't actively kick others off nodes.

### 2. PDB does NOT protect against preemption

This is the most common interview trap. A PodDisruptionBudget says "don't voluntarily take more than N of my pods offline." But preemption is not voluntary — it's the scheduler forcibly making room.

**PDB does NOT stop preemption.** If the scheduler decides your pod is the right victim, your PDB is not consulted. Your pod gets evicted regardless.

The only protection against preemption is having a high enough PriorityClass value.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is PriorityClass? | A K8s object that assigns a priority integer to pods |
| What does a higher priority number mean? | Higher chance of being scheduled first (queue); lower chance of being evicted (preemption victim) |
| What is preemption? | Scheduler evicting lower-priority pods to make room for a high-priority pod |
| Does PDB protect against preemption? | NO — PDB only protects against voluntary disruptions |
| What is `preemptionPolicy: Never`? | Use priority for queue ordering only; never preempt other pods |
| What are the built-in highest priorities? | `system-node-critical` (2000001000) and `system-cluster-critical` (2000000000) |
| When does preemption happen? | Only when the high-priority pod can't be scheduled anywhere; preemption is a last resort |
| Do evicted pods get their grace period? | Yes — `terminationGracePeriodSeconds` is respected during preemption |

---

## Self-test (one question — the killer one)

Out loud:

> **"Explain how priorityClass + preemption actually work — when would a high-priority pod evict a lower-priority one?"**

**Reference answer (intuitive version):**

"Preemption is triggered when a high-priority pod fails to find a node with enough free resources. The scheduler then looks at each node and asks: if I evicted some lower-priority pods from this node, would the high-priority pod fit? If yes, it picks the node where the impact is smallest — fewest pods evicted, lowest-priority victims — and evicts them.

The evicted pods get their normal termination grace period (`terminationGracePeriodSeconds`) to drain. The high-priority pod then gets scheduled to that node.

Two important gotchas. First: PDB does NOT protect against preemption — this surprises people. PDB only governs voluntary disruptions (node drains, cluster autoscaler). Second: preemption doesn't happen immediately — the scheduler finds the node and marks it, then waits for the evicted pods to terminate, THEN schedules the high-priority pod. So there's a delay between 'preemption decided' and 'high-priority pod running'."

---

## Further reading / watching

- **Kubernetes Docs — Pod Priority and Preemption**: https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/
- **Kubernetes Docs — PodDisruptionBudget**: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/ (to understand what PDB does NOT protect against)
- **System priority classes**: https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#how-to-use-priority-and-preemption

---

## Next: the deep-dive

When the airport analogy clicks and you understand the PDB-doesn't-block-preemption gotcha, jump to [`priority-preemption.md`](./deep-dive.md). The deep-dive covers:

- PriorityClass spec — `globalDefault`, `preemptionPolicy`, `description`
- How the scheduler selects preemption victims (least disruption first)
- The eviction sequence and grace periods during preemption
- Built-in system PriorityClasses and when to use them
- Cascading preemption — a subtle danger
- The distinction between preemption and node-pressure eviction
- 4 self-test drills + 4-dimensions framing
