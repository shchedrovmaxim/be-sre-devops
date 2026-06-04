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

This is the modern, more expressive alternative to anti-affinity for HA spread. Worth a deeper look because senior interviewers probe it hard.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway   # or DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-api
```

#### The three fields you need to know

| Field | What it controls |
|---|---|
| `maxSkew` | Max allowed difference between the most-loaded and least-loaded topology bucket. |
| `topologyKey` | What defines a "bucket" — node label name. Common: `topology.kubernetes.io/zone`, `kubernetes.io/hostname`. |
| `whenUnsatisfiable` | `DoNotSchedule` (hard — pod stays Pending if it would violate skew) or `ScheduleAnyway` (soft — schedule but try to minimize skew). |

#### `maxSkew` — the math, with diagrams

`maxSkew` is **(count in most-loaded domain) − (count in least-loaded domain)**. Domains here = the set of distinct values of `topologyKey` across nodes the pod is eligible to land on.

Worked examples — 3 zones (`a`, `b`, `c`), `maxSkew: 1`:

```
4 replicas, allowed distributions:        4 replicas, NOT allowed:
  zone-a: ██                                zone-a: ████          (skew = 4 - 0 = 4)
  zone-b: █                                 zone-b:               
  zone-c: █                                 zone-c:               
  skew = 2 - 1 = 1 ✅                       
                                            zone-a: ███           (skew = 3 - 0 = 3)
                                            zone-b: █             
                                            zone-c:               

5 replicas, allowed:                       7 replicas, allowed:
  zone-a: ██                                zone-a: ███
  zone-b: ██                                zone-b: ██
  zone-c: █                                 zone-c: ██
  skew = 2 - 1 = 1 ✅                       skew = 3 - 2 = 1 ✅
```

Note the **5 replicas case**: there's no way to land at perfectly even 2/2/1; that 1-pod gap is allowed because `maxSkew: 1` permits it. If you set `maxSkew: 0` you'd force perfectly even — which means 5 replicas could never fit into 3 zones at all (`5 / 3` isn't integer). That's almost never what you want; `maxSkew: 0` is mostly a trap.

#### `whenUnsatisfiable` — DoNotSchedule vs ScheduleAnyway

| | `DoNotSchedule` | `ScheduleAnyway` |
|---|---|---|
| What happens if constraint would be violated | Pod stays `Pending` | Pod schedules anyway; scheduler picks the node that minimizes skew |
| Use when | Hard HA requirement: "I will not run 4 of 5 replicas in one zone" | Best-effort spread: "I prefer even spread but never block a deploy" |
| Failure mode under capacity pressure | Deploy stalls | Skew temporarily violated, alert |

The senior heuristic: **`ScheduleAnyway` for most production workloads.** A blocked deploy is a worse outcome than a temporary skew that you can detect and rebalance. Reserve `DoNotSchedule` for truly critical HA (etcd, control plane components).

#### Layered constraints — node AND zone

This is the pattern you actually deploy. You want pods spread across zones for AZ-failure resilience AND spread across nodes within a zone for node-failure resilience.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: my-api
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: my-api
```

Two constraints, both evaluated, both must be satisfied. With 9 replicas + 3 zones × 3 nodes per zone you get 1 pod per node, perfectly spread. Beautiful. This single pattern replaces 90% of pod anti-affinity configs you'll see in older charts.

#### The rolling-update trap — and `matchLabelKeys`

This is the gotcha that distinguishes senior candidates. **Anti-affinity, naively configured, deadlocks rolling updates.**

Scenario: Deployment with 3 replicas. Pod anti-affinity says "never more than 1 per node." You have 3 nodes. Steady state: 1 pod per node ✅.

Now you push a new image. K8s tries to start a 4th pod (new version) before terminating an old one. **The 4th pod can't schedule** — every node already has a matching pod. Rollout is stuck.

The fix used to be `maxSurge: 0, maxUnavailable: 1` on the Deployment — terminate first, then schedule. That works but hurts availability during rollouts.

K8s 1.27+ adds **`matchLabelKeys`** to `topologySpreadConstraints`, which solves this cleanly:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-api
  matchLabelKeys:        # ← the magic
  - pod-template-hash
```

`matchLabelKeys` says "when counting matching pods, **only count pods that share the same value for these label keys as me.**" Every Deployment auto-labels its pods with `pod-template-hash` (one hash per ReplicaSet). So during a rollout:

- New-version pods spread among themselves (3 of them, 3 nodes — fits).
- Old-version pods spread among themselves (also 3, 3 nodes — fits).
- The two groups don't constrain each other.

**Result**: surge replicas don't deadlock. No more `maxUnavailable: 1` workaround.

If your cluster is on K8s 1.27+ — and most managed K8s is by now — **always set `matchLabelKeys: [pod-template-hash]` on Deployment topology spread constraints.** It's free correctness.

#### `minDomains` — force scheduling into N domains

K8s 1.25+. Use when you need a *minimum* number of domains, not just balanced spread.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  minDomains: 3                    # require pods spread across ≥ 3 zones
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-api
```

Without `minDomains`, if zone-c is briefly empty (no nodes), the scheduler considers only zones a and b, and 5 replicas might distribute 3/2 across them — looks balanced but you're not actually multi-AZ. `minDomains: 3` says "no, refuse to schedule unless 3 zones are in play."

Only works with `whenUnsatisfiable: DoNotSchedule`. Useful for compliance/SLA requirements that mandate multi-AZ presence.

#### `nodeAffinityPolicy` and `nodeTaintsPolicy` — what counts as a "domain"?

K8s 1.26+. Both default to `Honor` in modern versions, but worth knowing what they mean.

When the scheduler counts pods per domain, **should it count nodes the pod can't even use** (because of taints or nodeAffinity mismatch)?

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-api
  nodeAffinityPolicy: Honor    # only count nodes matching pod's nodeAffinity
  nodeTaintsPolicy: Honor      # only count nodes whose taints the pod tolerates
```

**Honor** = the scheduler ignores nodes the pod can't land on. This is what you want 99% of the time — "spread across zones the pod can actually use."

**Ignore** = the scheduler counts all nodes regardless. Subtle but breaks things: imagine 3 zones, 2 are GPU-tainted, your CPU pod can only use 1. With `Ignore`, the scheduler thinks "3 domains exist, I should spread to all of them" and refuses to schedule because 2 are unreachable.

**Interview soundbite**: "Honor means the scheduler asks 'where *can* this pod go?' before spreading. Ignore means it spreads first and asks questions later — usually wrong."

#### Cluster-wide default constraints

You can set defaults in the scheduler's config (`KubeSchedulerConfiguration`) so every pod without explicit constraints gets sensible spread:

```yaml
# In the scheduler's profile config:
pluginConfig:
- name: PodTopologySpread
  args:
    defaultConstraints:
    - maxSkew: 3
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
    defaultingType: List   # use these defaults (not "System" auto-defaults)
```

Pods that don't specify their own constraints inherit these. Great for cluster operators who want soft cluster-wide HA without asking app teams to opt in. App teams can still override per-pod.

This is rarely visible to app developers — it's a platform-team move. Worth knowing it exists for "how would you make every pod in the cluster zone-aware?" interview questions.

#### Real-world patterns you'll actually deploy

**Pattern 1 — Production web service, K8s 1.27+:**

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector: { matchLabels: { app: my-api } }
  matchLabelKeys: [pod-template-hash]
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector: { matchLabels: { app: my-api } }
  matchLabelKeys: [pod-template-hash]
```

Zone + node spread, soft, rollout-safe. **This is the default for stateless services.**

**Pattern 2 — Stateful workload requiring hard zone diversity:**

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  minDomains: 3
  whenUnsatisfiable: DoNotSchedule
  labelSelector: { matchLabels: { app: kafka } }
```

Refuse to come up unless we're in ≥ 3 AZs. Right for quorum-based systems (etcd, Kafka, Zookeeper).

**Pattern 3 — DaemonSet-ish "1 per node max":**

Use anti-affinity here — it's literally what it was designed for. Topology spread with `maxSkew: 1` works but is more verbose.

#### Why use topology spread over anti-affinity?

- **More expressive** — "max 2 per zone" is impossible with anti-affinity, trivial here.
- **Scales better** — anti-affinity's O(pods²) evaluation hurts at scale.
- **Soft mode** (`ScheduleAnyway`) — anti-affinity has `preferred` but it's clunky for skew.
- **`matchLabelKeys`** — solves the rolling-update deadlock cleanly.
- **`minDomains`** — anti-affinity can't express "must span ≥ N zones."

#### The senior answer to "anti-affinity vs topology spread"

> "I default to topologySpreadConstraints — it's more expressive, scales better, and `matchLabelKeys` resolves the rolling-update deadlock that anti-affinity has. I layer two constraints: one for zone spread, one for node spread, both with `ScheduleAnyway` so deploys never stall. I reserve pod anti-affinity for two cases: when I want a hard 'never more than 1 per node' for things like a DaemonSet-style deployment, or when I'm working in an older chart that already uses it. For co-location — which anti-affinity can't do anyway — pod affinity is still the right tool."

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
- Topology spread by default — more expressive (can do `maxSkew > 1`), scales better, supports `ScheduleAnyway` cleanly, and has `matchLabelKeys` which **resolves the rolling-update deadlock** that naive anti-affinity creates.
- Layer two constraints in practice: one for zone spread, one for node spread, both `ScheduleAnyway`. Always set `matchLabelKeys: [pod-template-hash]` on K8s 1.27+ for rollout safety.
- Use `minDomains` when you need a hard "must span ≥ N AZs" guarantee (quorum systems like etcd, Kafka).
- Anti-affinity only for hard "never more than 1 per node" cases or when integrating with older charts that already use it.
- For co-location (which anti-affinity can't do anyway): pod affinity is the only option.

### 4. A Deployment with 3 replicas and pod anti-affinity (`maxReplicas: 1 per node`, 3 nodes) deadlocks on rolling updates. Why? How would you fix it?

**Reference answer:**
- Anti-affinity counts *all* matching pods regardless of version. During a rollout, K8s wants to surge a 4th pod (new version) before terminating an old one — but every node already holds a matching pod, so the new one can't schedule. Stuck.
- Quick fix: `maxSurge: 0, maxUnavailable: 1` on the Deployment — terminate first, schedule second. Costs availability during rollouts.
- Better fix: switch to `topologySpreadConstraints` with `matchLabelKeys: [pod-template-hash]` (K8s 1.27+). That makes the constraint apply *only within the same ReplicaSet*, so the new-version pods spread among themselves without being blocked by old-version pods. Clean rollout, no surge limit needed.

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
