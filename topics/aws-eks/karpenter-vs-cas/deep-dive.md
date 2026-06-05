# Karpenter vs Cluster Autoscaler — the deep-dive

> **Goal**: by the end you can answer **"What problem does Karpenter solve that Cluster Autoscaler doesn't?"** with precision — naming the ASG-bound model, the direct EC2 Fleet API path, per-workload provisioning, active consolidation, spot interruption handling, and the disruption budget / TTL knobs. You can also say when CAS is still the right call.

> Start with the [simple version](./simple.md) if you haven't — the taxi dispatcher analogy is the mental model.

---

## The senior framing — this is a control-loop architecture question

Both tools solve the same surface problem: "the cluster doesn't have enough capacity, add more nodes." The difference is *where in the provisioning stack* each tool operates.

**CAS** works at the Kubernetes layer. It watches for unschedulable pods, identifies the ASG whose nodes would allow the pod to schedule, and calls the AWS Auto Scaling API to increment `DesiredCapacity`. It then waits. AWS does everything else.

**Karpenter** works at the EC2 layer. It watches for unschedulable pods, runs its own scheduling simulation to determine what instance type satisfies the pod's requirements (CPU, memory, labels, zone, capacity type), and calls `ec2:RunInstances` (via EC2 Fleet) directly. No ASG in the loop.

This architectural difference is load-bearing — every other comparison follows from it.

---

## CAS — how it actually works

### The ASG-bound model

CAS discovers ASGs tagged with `k8s.io/cluster-autoscaler/<cluster-name>: owned` and `k8s.io/cluster-autoscaler/enabled: true`. It maintains an inventory of "node groups" backed by these ASGs.

Scale-up flow:
1. Pod lands in `Pending` because no existing node satisfies its requests.
2. CAS runs a simulation: which node group, if scaled up by 1, would allow this pod to schedule?
3. CAS calls `SetDesiredCapacity(current + 1)` on the chosen ASG.
4. AWS ASG lifecycle kicks in: launch an instance from the ASG's launch template, run user-data bootstrap, kubelet joins the cluster.
5. **Typical latency: 3-6 minutes** (EKS-optimized AMI) or longer (custom AMI with complex bootstrap).
6. Pod is finally scheduled.

Scale-down flow:
1. CAS checks every node every `--scale-down-delay-after-add` (default: 10 min) and `--scale-down-unneeded-time` (default: 10 min).
2. If a node's pods can all be moved to other nodes AND the node has been underutilized for the configured period, CAS cordons → drains → decrements ASG.
3. CAS will **not** drain a node if any pod has a PodDisruptionBudget that would be violated.

### CAS gotchas

**Multi-AZ balancing problem**: If you have one ASG per AZ (the CAS recommended setup for EKS), and zone A is full but zone B has capacity, CAS will scale zone A ASG up — even if the pod has no zone requirement. This causes unnecessary spend and uneven zone distribution.

**Scale-down floor**: CAS has a `--min-replica-count` per node group and a `--max-empty-bulk-delete` (default: 10) for how many nodes it can delete at once. In practice, scaling down a large fleet that got over-provisioned takes many cycles.

**Instance type rigidity**: A single ASG has a single instance type (or a MixedInstancesPolicy with a pre-defined list). You cannot retroactively change the ASG's type mix without replacing the ASG and draining the nodes.

**No active consolidation**: CAS scales down only when a node is *already underutilized for a period*. It does not proactively repack 3 half-loaded nodes into 1 full node. You end up paying for fragmented compute.

---

## Karpenter — how it actually works

### The NodePool + EC2NodeClass model

```yaml
# NodePool — the scheduling policy (what workloads, what constraints)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]   # compute, general, memory — excludes GPU, bare metal
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]             # 5th gen and newer only
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 1m
  limits:
    cpu: 1000
    memory: 1000Gi
```

```yaml
# EC2NodeClass — the AWS-specific provisioning config
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: "KarpenterNodeRole-my-cluster"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
```

### The provisioning algorithm

When a pod is unschedulable, Karpenter:
1. Simulates placing the pod on every possible instance type that satisfies the NodePool's requirements.
2. Scores candidates by **price** (cheapest that fits), then by **zone** (spread), then by **capacity type** (spot first if allowed).
3. Calls EC2 Fleet API with an ordered list of instance types. AWS launches the cheapest available.
4. Node is up in **60-90 seconds** — no ASG lifecycle, no launch template warm-up.

The result: a `t3.medium` for a 500m CPU / 512Mi memory batch job; an `r6i.4xlarge` for a memory-intensive ML inference pod. Each pod gets a node shaped to its requests, not to the ASG's fixed type.

### Consolidation — the active repacking loop

Karpenter runs a consolidation controller on a configurable interval. The algorithm:

1. Find all nodes that are "consolidatable" — i.e., their pods can be rescheduled elsewhere without violating:
   - PodDisruptionBudgets
   - `karpenter.sh/do-not-disrupt: "true"` annotation on the pod or node
   - The NodePool's `disruption.budgets` (see below)
2. Simulate moving those pods to other nodes. If all pods can land, the node is a consolidation candidate.
3. Cordon → drain → terminate the node. Pods reschedule.

Two consolidation modes:
- `WhenUnderutilized` — replaces underutilized nodes with smaller/cheaper ones that still fit the pods.
- `WhenEmpty` — only terminates completely empty nodes. More conservative; good for stateful workloads.

### Disruption budgets and TTL knobs

```yaml
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 5m    # how long a node must be underutilized before Karpenter acts
    budgets:
      - nodes: "20%"        # never disrupt more than 20% of nodes at once
      - schedule: "0 9 * * 1-5"  # no disruption during business hours
        duration: 8h
        nodes: "0"
```

The `budgets` field is the key operational knob. Without it, Karpenter might try to consolidate too aggressively during a traffic spike, causing unnecessary pod churn.

### Spot interruption handling

AWS gives spot instances a **2-minute termination notice** via the EC2 instance metadata endpoint. Karpenter:
1. Polls the metadata endpoint on each node OR subscribes to EventBridge spot interruption events.
2. On notice: cordons the node immediately, begins draining (evicts pods respecting PDB).
3. 2-minute window is usually sufficient for graceful shutdown.

With CAS, you'd need the `aws-node-termination-handler` DaemonSet separately. Karpenter handles this natively.

**Spot capacity rebalancing**: AWS also sends a "rebalance recommendation" event *before* the interruption notice (more lead time, no guarantee). Karpenter can act on that too, proactively moving pods to on-demand or a different spot pool — set `karpenter.k8s.aws/interruption-queue: "my-queue"` to enable EventBridge-based interruption handling.

---

## When to keep CAS

Karpenter is the right default for new EKS clusters. CAS is still appropriate when:

- **Windows nodes**: Karpenter's Windows support is newer; CAS has more battle testing here.
- **Bottlerocket with custom validation**: some organizations have compliance requirements around node bootstrapping that are easier to enforce via ASG user-data and custom AMI pipelines than Karpenter's `EC2NodeClass`.
- **Managed node groups with specific lifecycle requirements**: e.g., you need AWS to manage the AMI rotation and security patches automatically. Managed node groups + CAS is the "AWS manages everything" path.
- **Existing CAS investment**: if you have well-tuned CAS configs and no pain, the migration cost may not be worth it.

The migration from CAS to Karpenter is a one-time node drain + reapply. Most teams do it during a low-traffic window. Karpenter can coexist with CAS (CAS manages some ASGs, Karpenter provisions independently) during migration.

---

## The interview answer in 60 seconds

> "CAS is ASG-bound. When it scales up, it increments the ASG's DesiredCapacity and waits for the ASG lifecycle — 3-6 minutes of warm-up. The node type is also fixed to whatever the ASG's launch template specifies.
>
> Karpenter calls EC2 Fleet API directly — no ASG round-trip — so scale-up is 60-90 seconds. It also runs its own scheduling simulation to select the cheapest instance type that satisfies the pending pod's requests, so a small batch job gets a small node and a memory-heavy pod gets an `r6i`.
>
> The other big delta is consolidation. CAS only scales down when a node is already empty. Karpenter actively repacks — it simulates moving pods from underutilized nodes onto other nodes, and if PDBs allow, it drains and terminates the underutilized node. This cuts waste in fleets that get fragmented after scale-out.
>
> Karpenter also handles spot interruption natively — the 2-minute notice triggers a drain automatically. With CAS you'd need a separate node termination handler DaemonSet.
>
> The senior nuance: Karpenter gives you more power, but also more surface area. You need to configure disruption budgets carefully, or consolidation can churn pods during traffic peaks. The NodePool + EC2NodeClass split is powerful but has more YAML to reason about than a tagged ASG."

---

## Self-test drills

### 1. What problem does Karpenter solve that Cluster Autoscaler doesn't?

**Reference answer:**
- CAS is ASG-bound, creating 3-6 min scale-up latency; Karpenter calls EC2 Fleet directly → 60-90 sec.
- CAS fixes node shape to the ASG's launch template; Karpenter selects the cheapest fitting instance type per workload.
- CAS waits for nodes to be empty before scaling down; Karpenter actively consolidates (repacks) underutilized nodes.
- CAS needs a separate DaemonSet for spot interruption handling; Karpenter handles it natively via EventBridge or metadata polling.
- Bonus: CAS has the multi-AZ balancing problem (may scale the wrong zone's ASG); Karpenter picks the zone that fits the pod's topology constraints.

### 2. Your Karpenter cluster has 20 nodes running at 25% utilization. How does consolidation work, and what could block it?

**Reference answer:**
- Karpenter's consolidation loop simulates moving pods from each node onto other nodes. If all pods on node X can land elsewhere, it's a consolidation candidate.
- Karpenter cordons node X, evicts pods (respecting PDB grace periods), and terminates the node.
- What blocks it: (a) a PDB on any pod on node X where the disruption would violate `minAvailable` or `maxUnavailable`; (b) `karpenter.sh/do-not-disrupt: "true"` annotation on the pod or node; (c) the NodePool's `disruption.budgets` limiting concurrent disruptions to e.g. 20% of nodes; (d) the `consolidateAfter` timer not yet elapsed.
- Practical gotcha: a single pod with a misconfigured PDB (`minAvailable: 100%`) can block an entire node from ever being consolidated. Audit PDBs before expecting consolidation to work.

### 3. Walk through what happens when a spot node gets an interruption notice in a Karpenter cluster.

**Reference answer:**
- AWS sends a 2-minute termination warning to the EC2 instance metadata endpoint (IMDSv2 path: `latest/meta-data/spot/termination-time`).
- If EventBridge interruption queue is configured, Karpenter also receives the event proactively (may have more than 2 minutes via rebalance recommendation).
- Karpenter cordons the node immediately (no new pods can schedule there).
- Karpenter issues eviction API calls for all pods on the node, respecting PDB grace periods.
- Karpenter also proactively launches a replacement node (or finds available capacity on existing nodes) so pods have somewhere to land.
- 2-minute window is typically enough for most workloads. Stateful pods with long `terminationGracePeriodSeconds` may not finish — design workloads for preemptibility.

### 4. How does Karpenter pick which instance type to provision for a pending pod?

**Reference answer:**
- Karpenter simulates scheduling: for each instance type that satisfies the NodePool's `requirements` (category, generation, arch, capacity type), it checks whether the pod's resource requests + node overhead fit within the instance's allocatable capacity.
- It scores candidates by price (cheapest that fits) → zone spread (prefer zones with fewer existing Karpenter nodes) → capacity type preference (spot first if allowed in NodePool).
- EC2 Fleet API is called with an ordered list of instance types; AWS returns the cheapest available in the requested zones.
- Senior nuance: Karpenter bins-packs multiple pending pods together before calling Fleet — it solves a bin-packing approximation to minimize node count, then prices the solution. So a burst of 10 pending pods triggers one Fleet call, not 10.

---

## Further reading / watching

- **Karpenter docs**: [karpenter.sh/docs](https://karpenter.sh/docs/) — NodePool reference and disruption docs are the two essential reads.
- **Karpenter GitHub — disruption design**: search `karpenter-provider-aws` on GitHub, then `website/content/en/docs/concepts/disruption.md`. The design rationale is documented here.
- **AWS re:Invent — "Optimizing Kubernetes compute costs with Karpenter"**: search YouTube. The demo clarifies the EC2 Fleet vs ASG difference with live metrics.
- **Karpenter FAQs — "Karpenter vs Cluster Autoscaler"**: at [karpenter.sh/docs/faq](https://karpenter.sh/docs/faq/) — a direct comparison maintained by the Karpenter team.
- **aws-node-termination-handler GitHub**: [github.com/aws/aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler) — understanding what CAS needs that Karpenter builds in natively.

---

## The 4 dimensions (senior framing)

- **Tech**: CAS = ASG DesiredCapacity control loop; Karpenter = EC2 Fleet direct provisioning with its own scheduling simulation. Key numbers: 3-6 min vs 60-90 sec scale-up; active consolidation vs passive scale-down. NodePool limits (`cpu: 1000`) are the capacity guard.
- **People**: Karpenter's NodePool + EC2NodeClass YAML is more expressive but also more to learn than "tag the ASG." Platform team needs to own the NodePool definitions. Document the disruption budget policy — if a team's pods can't tolerate being evicted (missing PDB, long `terminationGracePeriodSeconds`), they need to know to set `do-not-disrupt`.
- **CI/CD**: NodePool and EC2NodeClass changes go through GitOps (ArgoCD / Flux). Test disruption budget changes in a staging cluster first — a misconfigured budget can halt all consolidation or conversely evict too aggressively. Karpenter itself is managed via Helm; pin the chart version and use Renovate for upgrades.
- **Operations**: Monitor `karpenter_nodes_created_total`, `karpenter_nodes_terminated_total`, and `karpenter_pods_scheduled_total` in Prometheus. Alert on `karpenter_nodes_termination_duration_seconds` p99 > 5 min (indicates drain is stuck on a PDB). Runbook for "pod stuck Pending despite Karpenter": check `kubectl describe nodeclaim` and `kubectl get nodeclaim` — Karpenter creates a `NodeClaim` before the EC2 node appears; look for errors in the NodeClaim status.
