# Karpenter vs Cluster Autoscaler — the simple version (the taxi dispatcher)

> Read this first. Once the analogy clicks, the [deep-dive doc](./karpenter-vs-cas.md) becomes easy.

One idea:

> **Cluster Autoscaler resizes a fixed fleet of identical cars. Karpenter calls a new car of exactly the right size for each passenger.**

That's the whole thing. Everything else (NodePools, EC2NodeClass, consolidation, disruption budgets) is precision on top.

---

## The dispatcher analogy

Imagine a taxi company.

| Taxi dispatcher model | In Kubernetes |
|---|---|
| You have a fixed fleet of 10 identical sedans | Cluster Autoscaler, one ASG, all `m5.xlarge` |
| You add cars only by ordering another batch of sedans | CAS scales the ASG up — all new nodes are the same type |
| When 3 passengers show up, you dispatch 3 sedans even if 2 are going to the same destination | One ASG, no shape-matching |
| Call-center wait time before the new car arrives | ASG warm-up: 3-6 minutes (bake AMI → launch → bootstrap) |
| The company has a second dispatcher with a car app | Karpenter: calls Uber/Lyft directly |
| The app picks the cheapest car of the right size for each passenger | Karpenter picks the cheapest instance type fitting the workload |
| New car shows up in 60-90 seconds | Karpenter uses EC2 Fleet API directly — no ASG warm-up |
| At night, the app combines 3 half-empty sedans into 1 van | Karpenter consolidation: repacks underutilized nodes |

The key shift: **CAS manages a pool; Karpenter manages individual nodes on demand.**

---

## The 2 core concepts that confuse people

### 1. Why does CAS being ASG-bound matter?

When CAS wants to scale, it increments the ASG's `DesiredCapacity`. AWS then:
1. Launches a new EC2 instance from the ASG's launch template
2. Runs the bootstrap script (installs kubelet, joins the cluster)
3. The node registers with the API server
4. The scheduler can finally place the pod

That's a 3-6 minute pipeline. Every. Time. Karpenter skips the ASG entirely and calls EC2 Fleet API directly — the node is up in 60-90 seconds.

### 2. What does "consolidation" actually mean?

Imagine you have 5 nodes each running at 30% CPU. That's paying for 5 nodes to do 1.5 nodes of work. Karpenter can:
1. Pick the most "disruptable" node (lowest-priority pods, no PDB blocking)
2. Cordon it, drain it, terminate it
3. Reschedule its pods onto the remaining nodes

CAS scales *up* automatically; scaling *down* requires nodes to be empty for a configurable period (default: 10 min). Karpenter actively repacks — it doesn't wait for nodes to drain themselves.

---

## Intuition cheat sheet

| Question | CAS answer | Karpenter answer |
|---|---|---|
| What does it manage? | ASG DesiredCapacity | Individual EC2 nodes directly |
| Scale-up latency | 3-6 min (ASG warm-up + bootstrap) | 60-90 seconds (EC2 Fleet API) |
| Instance type flexibility | One type per ASG (or mixed-instance policy) | Any type fitting the workload, picked by price |
| How does it scale down? | Waits for empty nodes, then terminates | Active consolidation — repacks and drains proactively |
| Spot handling | Needs a Spot ASG wired separately | Built-in; handles 2-min interruption notice with node drain |
| Maturity | GA for years; battle-hardened | GA 2023; production-ready, AWS-native |
| Config model | `cluster-autoscaler` Deployment + ASG tags | `NodePool` + `EC2NodeClass` CRDs |
| When to keep CAS | Locked-in to specific AMIs; windows nodes; teams not ready to migrate | — |

---

## Self-test (the killer interview question)

Out loud:

> **"What problem does Karpenter solve that Cluster Autoscaler doesn't?"**

**Reference answer (intuitive version):**

"The core issue with CAS is that it's ASG-bound. To scale up, CAS increments the ASG's desired capacity and waits for the ASG lifecycle to spin up a new node — that's a 3-6 minute wait per scale event. The node shape is also fixed by the ASG's launch template, so you can't get a node that's perfectly sized to the pending pod.

Karpenter solves both. It calls EC2 Fleet API directly — no ASG round-trip — so scale-up is 60-90 seconds. It also selects the cheapest EC2 instance type that fits the pending pod's actual requests, rather than always getting the same type.

The other thing Karpenter adds is active consolidation. CAS waits for a node to be fully empty before it terminates it. Karpenter proactively repacks underutilized nodes — it looks at all running nodes, finds candidates it can drain without violating PodDisruptionBudgets or the node's disruption budget, drains them, and reschedules the pods more efficiently.

For spot workloads, Karpenter also has a built-in 2-minute interruption handler — when AWS sends the spot interruption notice, Karpenter cordons the node and drains it before the 2 minutes are up, gracefully moving pods. With CAS you'd need a separate node termination handler DaemonSet for that."

---

## Further reading / watching

- **Karpenter official docs**: [karpenter.sh/docs](https://karpenter.sh/docs/) — the Getting Started guide and the NodePool/EC2NodeClass reference are the two essential reads.
- **AWS blog — "Karpenter: Open-Source, High-Performance Kubernetes Cluster Autoscaler"**: search "AWS Karpenter blog 2021" on AWS. Good origin story.
- **Karpenter GitHub**: [github.com/aws/karpenter-provider-aws](https://github.com/aws/karpenter-provider-aws) — the disruption docs under `website/content` are authoritative on consolidation mechanics.
- **Re:Invent talk — "Optimizing Kubernetes compute costs with Karpenter"**: search on YouTube. The live demo clarifies the ASG-vs-direct-API difference visually.

---

## Next: the deep-dive

When the taxi analogy feels obvious, jump to [`karpenter-vs-cas.md`](./karpenter-vs-cas.md). The deep-dive covers:

- The full NodePool + EC2NodeClass config model
- How Karpenter selects instance types (the scheduling algorithm)
- Disruption budgets and TTL knobs
- Spot interruption handling end-to-end
- The consolidation algorithm and what blocks it
- CAS gotchas (the multi-AZ balancing problem, the scale-down floor)
- 4-dimensions framing for interview
