# K8s scheduling — taints, affinity, anti-affinity, topology

> **Goal**: by the end you can answer the rejection-gap question — **"Walk me through what to check, in order, when a pod won't schedule"** — and explain how taints, node affinity, pod affinity, and topology spread constraints **work together**, not in isolation.

> This was the explicit rejection feedback: the interviewer wanted to hear the **correlation**, not 4 features defined independently.

---

## The whole topic in one diagram

When the scheduler is deciding where to put a pod, it asks three questions in order:

```
Pod needs a node
       │
       ▼
┌───────────────────────────────────────────────────┐
│ Q1: CAN the pod run here?      (TAINTS)           │
│     Node says: "I have rules. Got the toleration?"│
└───────────────────────────────────────────────────┘
       │ filter → only nodes whose taints are tolerated remain
       ▼
┌───────────────────────────────────────────────────┐
│ Q2: DOES the pod want to be here?  (AFFINITY)     │
│     Pod says: "I prefer/require certain nodes     │
│     and certain neighbors."                       │
│       • nodeAffinity      (about the node)        │
│       • podAffinity       (be near friends)       │
│       • podAntiAffinity   (avoid enemies)         │
└───────────────────────────────────────────────────┘
       │ filter → only nodes matching pod's wishes remain
       ▼
┌───────────────────────────────────────────────────┐
│ Q3: ARE WE SPREAD EVENLY?  (TOPOLOGY)             │
│     Cluster says: "Don't pile all replicas        │
│     in one zone/node."                            │
└───────────────────────────────────────────────────┘
       │ score → pick the best remaining node
       ▼
   Pod scheduled
```

Memorize this. Every "won't schedule" debug starts at Q1 and walks down.

---

## The restaurant analogy (the intuition)

You're managing a wedding reception. Guests (pods) need tables (nodes). There are rules.

| Real K8s feature | At the wedding |
|---|---|
| **Taints** on a node | Some tables are reserved — say, the VIP table for family only. If you're not a family member (no toleration), you can't sit there even if it's empty. |
| **Toleration** on a pod | You're on the family list. Now you *can* sit at the VIP table — but it doesn't mean you *want* to. |
| **Node affinity** | You request a window seat. Hard requirement (`required`) or soft preference (`preferred`). |
| **Pod affinity** | "Please seat me with my partner." Two pods want to sit together. |
| **Pod anti-affinity** | "Don't seat me next to my ex." HA: keep replicas apart so a single table going down doesn't take them all. |
| **topologySpreadConstraints** | "Spread the wedding party evenly across all tables for the photo." More expressive than anti-affinity — you can say "max 2 per table, max 4 per section." |

Two things that catch people out:

1. **Toleration ≠ attraction.** It just *allows* the pod onto a tainted node. To actually steer it there, you also need node affinity. Tolerations open doors; affinity is what walks through them.
2. **Pod affinity and anti-affinity reference *other pods*, not nodes.** They use `topologyKey` to define what "near" means: same node, same zone, same rack.

---

## The 4 pieces — in order

### 1. Taints + tolerations — "can the pod run here?"

A taint is a rule **the node owner** puts on a node: "only pods with the matching toleration may land here." Pod owners opt in by adding a toleration to the pod spec.

```yaml
# On the node (set by node admin or cloud provider):
kubectl taint nodes gpu-node-1 nvidia=true:NoSchedule

# On the pod (set by the workload):
tolerations:
- key: nvidia
  operator: Equal
  value: "true"
  effect: NoSchedule
```

**The three effects** — this is the part interviewers probe:

| Effect | What happens to pods without the toleration |
|---|---|
| `NoSchedule` | New pods can't schedule here. **Existing pods stay running** if they were already there before the taint was added. |
| `PreferNoSchedule` | Scheduler tries to avoid this node, but if no other node fits, it'll land here anyway. **Soft version.** |
| `NoExecute` | New pods can't schedule, AND existing pods without the toleration are **evicted immediately** (or after `tolerationSeconds` if you set it). The harshest one. |

**Real-world `NoExecute`**: when a node goes `NotReady`, K8s automatically taints it with `node.kubernetes.io/unreachable:NoExecute`. That's how pods get evicted off a sick node.

**Common taint patterns:**
- Dedicated GPU nodes: `nvidia.com/gpu=true:NoSchedule`
- Dedicated nodes for a team/tenant: `team=payments:NoSchedule`
- Control plane nodes: pre-tainted `node-role.kubernetes.io/control-plane:NoSchedule`

**Gotcha**: A toleration with `operator: Exists` matches *any value* for the key. Forgetting this is how unrelated pods accidentally schedule onto your dedicated GPU nodes.

```yaml
# DON'T do this on a general workload — it'll happily land on tainted nodes:
tolerations:
- operator: Exists   # matches ALL taints, with any key, any effect
```

---

### 2. Node affinity — "does the pod want this kind of node?"

Now the pod says **which nodes it wants**, based on node labels.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a", "us-east-1b"]
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.xlarge"]
```

**Required vs preferred** — second thing interviewers probe:

| | Required | Preferred |
|---|---|---|
| Long name | `requiredDuringSchedulingIgnoredDuringExecution` | `preferredDuringSchedulingIgnoredDuringExecution` |
| Behavior | Hard constraint. No match = pod stays `Pending`. | Soft hint. Scheduler scores nodes and prefers matching ones, but will use a non-matching node if nothing else fits. |
| Use when | Genuine requirement (this app needs GPU; this DB must be in a specific zone). | Preference (this app runs better on m5 but works anywhere). |

**The "IgnoredDuringExecution" part** matters: these constraints are checked **only at scheduling time**. If a node label changes after the pod is running, the pod is NOT moved. (K8s has no stable `requiredDuringExecution` mode for affinity; that was the original plan but it never shipped.)

**`nodeSelector` is the legacy short form.** It's equivalent to a required `In` match on each key. Still valid, still works. Just less expressive. Don't dwell on it in an interview — mention you know it, move on.

---

### 3. Pod affinity + anti-affinity — "where are my friends/enemies?"

Now the pod talks about **other pods**, not nodes. The question is "do I want to sit *near* (or *far from*) pods matching this label selector?"

The key concept is **`topologyKey`**: a node label that defines what "near" means. Common values:

| `topologyKey` | Means "near = on the same…" |
|---|---|
| `kubernetes.io/hostname` | Node |
| `topology.kubernetes.io/zone` | Availability zone |
| `topology.kubernetes.io/region` | Region |

Example — **pod affinity** ("co-locate with cache pods on the same node"):

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: redis
      topologyKey: kubernetes.io/hostname
```

Translation: "place me on a node that already has a pod with `app: redis`." Useful for caches, local agents, sidecars to a co-located service.

Example — **pod anti-affinity** ("don't put two of my replicas in the same zone, for HA"):

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-api
      topologyKey: topology.kubernetes.io/zone
```

Translation: "don't schedule me into a zone that already has a pod with `app: my-api`."

**Gotchas:**

1. **Anti-affinity scales badly.** The scheduler evaluates this against every existing pod on every candidate node. With thousands of replicas it's slow. For large fleets, use `topologySpreadConstraints` instead (next section).
2. **`required` anti-affinity can starve scheduling.** If you say "max 1 per zone" with 3 replicas and 2 zones, the 3rd replica never schedules.
3. **`labelSelector: {}`** (empty) matches **everything**. Easy to footgun.

---

### 4. `topologySpreadConstraints` — "spread evenly"

This is the modern, more expressive alternative to anti-affinity for HA spread.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway   # or DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-api
```

The three fields you need to know:

| Field | What it controls |
|---|---|
| `maxSkew` | Max allowed difference in pod count between any two topology buckets. `maxSkew: 1` over 3 zones → counts like 3/3/2 are OK; 4/3/2 is not. |
| `topologyKey` | Same idea as anti-affinity — what defines a "bucket" (zone, host, region). |
| `whenUnsatisfiable` | `DoNotSchedule` (hard — pod stays Pending if it'd violate skew) or `ScheduleAnyway` (soft — schedule but try to minimize skew). |

**Why use this over anti-affinity?**

- More expressive: "max 2 per zone" is impossible with anti-affinity (it's only "0 or 1"), trivial with topology spread.
- Scales better.
- Lets you set soft (`ScheduleAnyway`) which is hard to express with `required` anti-affinity.
- Combines with `nodeAffinityPolicy` / `nodeTaintsPolicy` to respect the upstream filters.

**The senior answer to "anti-affinity vs topology spread":**

> "I use topology spread by default — it's more expressive and scales better. I use pod anti-affinity only when I genuinely need 'never more than one on the same X' as a hard rule. Pod affinity (co-location) still has its place — topology spread is purely about spreading, not co-locating."

---

## How they combine — the worked-example trap

The interviewer's trap question:

> *"You have a GPU node tainted `nvidia=true:NoSchedule`. Your training pod has the toleration but also a `nodeAffinity` requiring `arch=amd64`. The GPU node has label `arch=arm64`. Will the pod schedule there?"*

Walk the 3 questions:

1. **Can it run here? (Taints)** — Pod has the toleration. ✅ Q1 passes.
2. **Does it want to be here? (Affinity)** — Pod requires `arch=amd64`. Node has `arch=arm64`. ❌ Q2 fails.
3. **Skip Q3 — already filtered out.**

**Result**: pod won't schedule on this node, despite having the toleration. The toleration only opens the door; nodeAffinity is what decides if the pod walks through.

This question separates candidates who memorized features from candidates who understand the flow. Now you understand the flow.

---

## "My pod won't schedule" — the troubleshooting flow

This is the runbook. Walk it in this order, every time.

### Step 1 — read the events

```bash
kubectl describe pod <pod>
```

Look at the bottom for events. The scheduler logs why it failed:

```
Events:
  Warning  FailedScheduling  10s  default-scheduler  0/5 nodes are available:
    3 node(s) had untolerated taint {nvidia: true},
    2 node(s) didn't match Pod's node affinity/selector.
```

That message **already tells you Q1 (3 nodes filtered by taint) and Q2 (2 nodes filtered by affinity)**. Decode it.

### Step 2 — walk Q1 (taints)

```bash
# What taints exist on each node?
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'

# What tolerations does the pod have?
kubectl get pod <pod> -o json | jq '.spec.tolerations'
```

Match them up. Common mistakes:
- `operator: Equal` requires `value` to match exactly. Typo in value → no match.
- Missing the `effect` field → toleration matches *only the matching effect*. If taint is `NoExecute` and toleration omits effect, it might not match.

### Step 3 — walk Q2 (affinity)

```bash
# What labels are on each node?
kubectl get nodes --show-labels

# What does the pod require?
kubectl get pod <pod> -o yaml | yq '.spec.affinity'
```

Common mistakes:
- Label key typo (`zone` instead of `topology.kubernetes.io/zone`).
- `In` operator with values list that doesn't include the actual node value.
- `nodeSelector` + `nodeAffinity` both set; they must *both* match (AND, not OR).

### Step 4 — walk Q3 (topology spread + anti-affinity)

```bash
# Other pods of the same app — where are they?
kubectl get pods -l app=my-api -o wide
```

If anti-affinity or `topologySpreadConstraints` are at play, count how the existing replicas are distributed. If they fully saturate the constraint, your new replica can't fit.

### Step 5 — capacity

If all 3 questions pass and the pod *still* won't schedule, it's capacity. The events will say:

```
0/5 nodes are available: 5 Insufficient cpu.
```

Now you're looking at resource requests vs node allocatable. Different problem.

---

## Self-test

### 1. Walk me through what to check, in order, when a pod won't schedule.

**Reference answer:**
- `kubectl describe pod` — read the FailedScheduling event; it pre-segments the failure by reason ("3 nodes had untolerated taint, 2 didn't match affinity").
- Q1 — taints: compare `node.spec.taints` to `pod.spec.tolerations`. Watch for missing `effect` field or wrong value.
- Q2 — affinity: compare node labels to `pod.spec.affinity.nodeAffinity` (and `nodeSelector` if present — both must match).
- Q3 — anti-affinity/topology: count where existing replicas already are.
- If all three pass, it's resource pressure or a control-plane issue (scheduler down, no nodes at all).

### 2. When would you use pod anti-affinity vs `topologySpreadConstraints`?

**Reference answer:**
- Topology spread by default — more expressive, scales better, supports `maxSkew > 1`, supports `ScheduleAnyway` cleanly.
- Anti-affinity only when you need a hard "never more than 1 per X" guarantee, or when integrating with older charts that already use it.
- For HA spread of replicas across zones: topology spread wins.
- For co-location (which anti-affinity can't do anyway): pod affinity is the only option.

### 3. GPU node tainted `nvidia=true:NoSchedule`. Pod has toleration AND `nodeAffinity` requiring `arch=amd64`. Node has `arch=arm64`. Will it land there?

**Reference answer:**
- No. Q1 (taint) passes — toleration is correct. Q2 (affinity) fails — pod requires amd64, node is arm64.
- The toleration is necessary but not sufficient. It only opens the door; affinity decides whether the pod walks through.
- Common failure mode for GPU workloads when teams forget that GPU nodes are often arm64 (Graviton) on AWS.

---

## The 4 dimensions (senior framing)

- **Tech**: know the 3-question flow cold; know `effect` semantics on taints; know topology spread > anti-affinity for HA in most cases; know that affinity is checked only at scheduling time (`IgnoredDuringExecution`).
- **People**: developers ship pods with `tolerations: - operator: Exists` to "just make it schedule" — that breaks tenant isolation. Educate or block with Kyverno/Gatekeeper. Provide opinionated Helm chart defaults so dev teams don't reinvent affinity rules.
- **CI/CD**: enforce a policy that production workloads must have `topologySpreadConstraints` for HA. Lint pod specs in CI: no blanket `Exists` tolerations, no anti-affinity without a topologyKey, required matches must reference existing node labels.
- **Operations**: when a node goes NotReady, automatic `NoExecute` taints evict pods — make sure your `tolerationSeconds` on critical workloads is sane (default is 5 minutes for the unreachable taint; you might want shorter). Runbook for "pod stuck Pending": always start with `kubectl describe pod`; never guess.

---

## What we did NOT cover (parked for later)

- `priorityClass` + preemption (when a higher-priority pod evicts lower-priority ones to schedule)
- Pod overhead, ephemeral storage scheduling, GPU-specific extended resources
- Multi-scheduler setups (running a custom scheduler alongside the default)
- Scheduler profiles (K8s 1.18+ extension points)

These belong in a follow-up session.
