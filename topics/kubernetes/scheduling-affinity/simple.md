# K8s scheduling — the simple version (the wedding seating chart)

> Read this first. Once the 3-question flow clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **When a pod won't schedule, the scheduler filtered it out at one of three checkpoints. If you know the checkpoints in order, you can diagnose any scheduling failure in under 5 minutes.**

That's the whole mental model. Every YAML concept (taints, tolerations, affinity, topology spread) is just a knob on one of those three checkpoints.

---

## The wedding seating chart

You're running a wedding reception. Guests (pods) need tables (nodes). Before anyone sits down, the event manager asks three questions:

**Question 1 — Can this guest sit at this table?**
Some tables are reserved — the VIP family table has a placard: "family members only." If you're not on the family list, you can't sit there even if every other chair is empty. The *table* is setting the rule.

In K8s: this is **taints** on the node, and **tolerations** on the pod. The node says "I have a restriction." The pod opts in with a matching toleration.

**Question 2 — Does this guest *want* this table?**
Even if a guest is allowed at the family table, they might have requested a window seat, or asked to be seated near a friend, or asked to be far from their ex. These are the guest's own preferences.

In K8s: this is **affinity** on the pod. Node affinity says "I want this kind of node." Pod affinity says "put me near these other pods." Pod anti-affinity says "keep me away from those pods."

**Question 3 — Are we spreading guests evenly?**
If 40 guests are all trying to sit at the same 2 tables, the photographer's group shot will look lopsided. The manager tries to distribute people across all available tables for balance.

In K8s: this is **topologySpreadConstraints** — "spread replicas evenly across zones and nodes."

```
Pod needs a node
       │
       ▼
┌────────────────────────────────────────────┐
│ Q1: CAN the pod run here?  (TAINTS)        │
│     "Is the guest allowed at this table?"  │
└────────────────────────────────────────────┘
       │ surviving nodes →
       ▼
┌────────────────────────────────────────────┐
│ Q2: DOES the pod want here?  (AFFINITY)    │
│     "Does the guest want this table?"      │
└────────────────────────────────────────────┘
       │ surviving nodes →
       ▼
┌────────────────────────────────────────────┐
│ Q3: ARE WE SPREAD EVENLY?  (TOPOLOGY)      │
│     "Is the seating balanced?"             │
└────────────────────────────────────────────┘
       │ pick best remaining node
       ▼
   Pod scheduled
```

Memorize this flow. Every "pod stuck in Pending" debug starts at Q1 and walks down.

---

## The one insight that prevents 80% of scheduling confusion

**Toleration ≠ attraction.**

A toleration just says "I'm allowed at the VIP table." It doesn't say "I want to be at the VIP table." If you have a dedicated GPU node with a taint, and your pod has a toleration for it, the pod is *permitted* on the GPU node — but without a matching node affinity, the scheduler might still put it on a regular node.

> **Tolerations open doors. Affinity is what walks through them.**

The pattern for dedicated nodes:
1. Taint the node (exclude everyone without the toleration).
2. Toleration on the pod (permission to enter).
3. Node affinity on the pod (actually steer it there).

Without step 3, your ML training job might end up on a CPU node that happens to have no competing workloads. The toleration opened the door; affinity is the missing step that walks through it.

---

## The trap question — the one that separates candidates

The interviewer will ask something like:

> "GPU node is tainted `nvidia=true:NoSchedule`. Your training pod has a toleration for that taint AND a `nodeAffinity` requiring `arch=amd64`. The GPU node has label `arch=arm64`. Will the pod schedule there?"

Walk the three questions:

- **Q1 — Can it run here (taints)?** Pod has the toleration. Q1 passes. The door is open.
- **Q2 — Does it want to be here (affinity)?** Pod requires `arch=amd64`. Node has `arch=arm64`. Q2 fails. The pod walks up to the door and turns around.
- **Q3 — Skip.** Already eliminated.

**Answer: no, it won't schedule there.** The toleration is necessary but not sufficient.

This is exactly the confusion in real GPU clusters on AWS, where EC2 GPU instances are often arm64 (Graviton). Teams add the toleration, scratch their heads when the pod stays Pending, and eventually notice their node affinity is blocking it.

---

## The "pod won't schedule" runbook — in order

Run these steps in sequence, not in parallel:

### Step 1 — read what the scheduler already tells you

```bash
kubectl describe pod <pod>
```

Scroll to the `Events` section at the bottom. The scheduler writes a message like:

```
Warning  FailedScheduling  0/5 nodes are available:
  3 node(s) had untolerated taint {nvidia: true},
  2 node(s) didn't match Pod's node affinity/selector.
```

That single line has already segmented the failure by checkpoint: 3 nodes failed Q1, 2 nodes failed Q2. Read it carefully before doing anything else.

### Step 2 — check Q1: taints

```bash
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
kubectl get pod <pod> -o json | jq '.spec.tolerations'
```

Match taints to tolerations manually. Common traps:
- Toleration has `operator: Equal` but value is a typo — exact match required.
- Toleration missing `effect` — a toleration without `effect` only matches that specific effect, not all effects.
- `operator: Exists` on a toleration — this matches every taint on the node, which is usually an accident. Your "just make it schedule" pod will happily land on GPU or reserved nodes.

### Step 3 — check Q2: affinity

```bash
kubectl get nodes --show-labels
kubectl get pod <pod> -o yaml | grep -A 20 affinity
```

Match node labels against the pod's `nodeAffinity`. Common traps:
- Label key typo (`zone` vs `topology.kubernetes.io/zone`).
- Using `In` with a values list that doesn't include the actual label value on any node.
- `nodeSelector` and `nodeAffinity` are both set — they must both match (AND, not OR).
- Affinity is `required` (hard constraint) when `preferred` (soft hint) would be more appropriate.

### Step 4 — check Q3: topology and anti-affinity

```bash
kubectl get pods -l app=<your-app> -o wide
```

Count where existing replicas live. If you have anti-affinity saying "max 1 per node" and all 3 nodes already have a replica, a 4th pod can never schedule. If you have `topologySpreadConstraints` with `maxSkew: 1` and zones are unbalanced, the new pod might block on spread.

### Step 5 — if all 3 pass, it's resource pressure

The scheduler will tell you:

```
0/5 nodes are available: 5 Insufficient cpu.
```

Now you're looking at resource requests vs node allocatable. That's a capacity problem, not a scheduling policy problem. Different conversation.

---

## Quick reference: the three effects of taints

| Effect | What it does to pods without a toleration |
|---|---|
| `NoSchedule` | New pods can't land here. Pods already running stay — this is not retroactive. |
| `PreferNoSchedule` | Scheduler tries to avoid this node, but will use it if nothing else fits. Soft version. |
| `NoExecute` | New pods can't schedule AND existing pods without the toleration are evicted. The harshest. This is what K8s uses automatically when a node goes `NotReady`. |

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does a taint do? | Marks a node as "restricted." Only pods with a matching toleration are allowed. |
| What does a toleration do? | Grants *permission* to land on a tainted node. Does not steer the pod there. |
| What does node affinity do? | Steers the pod to nodes with matching labels. Can be hard (required) or soft (preferred). |
| When does affinity check happen? | Only at scheduling time. If a node label changes after the pod is running, the pod stays — it's `IgnoredDuringExecution`. |
| Tolerations vs affinity — when do I need both? | When you have dedicated nodes. Taint + toleration excludes others; affinity actually routes your pod there. |
| What does `topologySpreadConstraints` do? | Spreads replicas evenly across zones or nodes. Better than anti-affinity for most HA patterns — more expressive, scales better. |
| Why use topology spread over anti-affinity? | Topology spread supports `ScheduleAnyway` (won't block deploys), handles `maxSkew > 1`, and has `matchLabelKeys` to avoid rolling-update deadlocks. |
| If pod is Pending, first command? | `kubectl describe pod` — read the `FailedScheduling` event. It tells you which checkpoint failed and how many nodes were filtered there. |

---

## Self-test (one question — the killer one)

Out loud:

> **"A pod won't schedule. Walk me through what to check, in order."**

**Reference answer (intuitive version):**

"I always start with `kubectl describe pod` and read the `FailedScheduling` event at the bottom. The scheduler already tells you how many nodes failed each filter — for example, '3 nodes had untolerated taint, 2 didn't match node affinity.' That segments the problem immediately.

Then I walk the three checkpoints in order. First, Q1 — taints: I compare the node taints to the pod's tolerations, looking for value typos, missing effect fields, or an accidental `operator: Exists` that was added to 'just make it schedule.' Second, Q2 — affinity: I compare node labels against the pod's `nodeAffinity` and `nodeSelector`, looking for label key typos, missing label values, or a `required` constraint that should have been `preferred`. Third, Q3 — topology: I check where existing replicas are distributed and whether anti-affinity or `topologySpreadConstraints` have fully saturated the available domains.

If all three pass, the pod still can't schedule — then it's resource pressure (`Insufficient cpu/memory`), which is a different problem: look at resource requests vs node allocatable capacity.

The senior framing: tolerations open doors, affinity walks through them. Always verify both when a pod has dedicated-node requirements."

---

## Further reading / watching

- **K8s docs — Taints and Tolerations** (kubernetes.io) — clear walkthrough of all three effects and real-world patterns.
- **K8s docs — Assigning Pods to Nodes** — covers nodeSelector, nodeAffinity, pod affinity/anti-affinity.
- **K8s docs — Pod Topology Spread Constraints** — the modern HA spread pattern, including `matchLabelKeys` and `minDomains`.

The tool you'll use every single time:

- **`kubectl describe pod`** — the scheduler writes the entire failure reason into the events. Read it before touching any YAML.

---

## Next: the deep-dive

When the three-question flow feels automatic, jump to [`scheduling-affinity.md`](./deep-dive.md). The deep-dive covers:

- All three taint effects with examples and the `NoExecute` eviction mechanic
- `required` vs `preferred` affinity — the full YAML and when each fails
- Pod affinity and anti-affinity with `topologyKey` — how "near" is defined
- `topologySpreadConstraints` in depth: `maxSkew`, `whenUnsatisfiable`, `minDomains`, and the rolling-update deadlock solved by `matchLabelKeys`
- Why topology spread beats anti-affinity at scale (O(pods²) evaluation cost)
- The `nodeAffinityPolicy: Honor` and `nodeTaintsPolicy: Honor` fields (what counts as a domain)
- Cluster-wide default spread constraints (platform-team level config)
- 4-dimensions framing (Tech, People, CI/CD, Operations)
- 4 self-test drills with reference answers

The deep-dive is the reference. This doc is the mental model. The three-question flow is what you walk through every time a pod goes Pending.
