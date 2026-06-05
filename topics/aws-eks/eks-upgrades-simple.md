# EKS upgrades — the simple version (the rolling highway resurfacing)

> Read this first. Once the analogy clicks, the [deep-dive doc](./eks-upgrades.md) becomes easy.

One idea:

> **An EKS upgrade is like resurfacing a highway while traffic is still moving. You can't close the whole road — you do it in sections, in order, and you check each section before moving on.**

The order matters: you always resurface the core infrastructure (control plane) before the vehicles (nodes), because a new road works with old cars, but old roads can break new cars.

---

## The highway analogy

| Highway resurfacing | EKS upgrade |
|---|---|
| Highway control center (traffic management) | EKS control plane (API server, scheduler, etcd) |
| Resurfacing control center first | Upgrade control plane first — always |
| Resurfacing lane by lane, not all at once | Rolling node upgrade — not all nodes at once |
| Checking each lane before opening it | Health checks and pod rescheduling after node drain |
| Some construction vehicles must be swapped out too | Addon upgrades: VPC CNI, CoreDNS, kube-proxy |
| Old cars still work on new road (backward compatible) | Kubelet N-2 behind control plane is allowed |
| New car model needs new road features | New kubelet needs at least N-1 control plane |
| Blocking road construction = detour (extra traffic elsewhere) | PDB-blocked drain = upgrade stalls until pod moves |

The absolute order: **control plane → addons → nodes.** Never the other way around.

---

## The 2 concepts that confuse people

### 1. Why do addons need to be upgraded too?

VPC CNI, CoreDNS, and kube-proxy are separate components with their own version numbers. They run in the cluster (as DaemonSets or Deployments) and talk to the K8s API. A K8s 1.30 API server may have deprecated or removed APIs that the old addon version still calls. If you upgrade the control plane and forget to upgrade the addons, you can end up with a broken cluster that looks healthy — until you hit the removed API path.

AWS publishes a compatibility matrix per EKS version. Before upgrading, check which addon versions are compatible with the target K8s version.

### 2. What's the difference between in-place rolling and blue-green node upgrade?

**In-place rolling (managed node groups)**: AWS drains nodes one by one and replaces them with new AMI nodes. Old nodes are drained; pods reschedule onto other nodes. You need `n+1` capacity to absorb the drain. Default; simpler.

**Blue-green**: you create a new node group with the new K8s version and AMI. Gradually cordon and drain the old node group. When all pods have migrated, delete the old group. More control; safer for stateful workloads; costs more temporarily (two node groups running at once).

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What order do you upgrade? | Control plane → addons → nodes (always in this order) |
| Who manages the control plane upgrade? | AWS — it's one API call or one button. You do the pre-flight, AWS does the upgrade. |
| How long does a control plane upgrade take? | ~10-15 minutes. Your workloads keep running (API may blip briefly). |
| What is the kubelet skew policy? | Kubelet can be at most N-2 behind the control plane version. |
| What breaks during an upgrade? | PDBs blocking node drains; deprecated API objects in Helm charts; addon version mismatch |
| How do you check for deprecated APIs? | `kubectl get --raw /metrics | grep apiserver_requested_deprecated_resources` or use tools like `pluto` |
| What's the fastest way to stall an upgrade? | A PDB with `minAvailable: 100%` on a single-replica deployment will block node drain forever |

---

## Self-test (the killer interview question)

Out loud:

> **"Walk me through a zero-downtime EKS minor version upgrade across the control plane and 100 nodes."**

**Reference answer (intuitive version):**

"I'd approach it in four phases.

**Phase 1 — Pre-flight**. Before touching anything, I'd check for deprecated API usage. The control plane upgrade will refuse if any objects in etcd use removed APIs. I'd run `pluto detect-all-in-cluster` or check `apiserver_requested_deprecated_resources` metrics. I'd also verify addon compatibility: which versions of VPC CNI, CoreDNS, and kube-proxy are compatible with the target K8s version? AWS publishes this matrix.

I'd also review all PodDisruptionBudgets in the cluster. Any PDB with `minAvailable: 100%` on a single-replica deployment will block node drains. Fix those first.

**Phase 2 — Control plane upgrade**. One API call (`aws eks update-cluster-version`) or the EKS console. AWS handles it — typically 10-15 minutes. During this time, the API server may blip (brief requests timeout); workloads keep running. Monitor API server latency but don't expect zero interruption at the API layer.

**Phase 3 — Addon upgrades**. Immediately after the control plane finishes, upgrade VPC CNI, CoreDNS, and kube-proxy to versions compatible with the new K8s version. This is critical — don't skip. These are managed via the EKS addon API (`aws eks update-addon`).

**Phase 4 — Node upgrades**. For managed node groups, rolling in-place: AWS replaces nodes 30% at a time by default (configurable `updateConfig.maxUnavailable`). Each node is cordoned → drained → terminated → replaced with a new node on the new AMI. The new node registers, becomes Ready, and the next batch starts.

For 100 nodes I'd increase `maxUnavailable` to maybe 20-30% to get through it faster, but I'd watch pod disruption closely. Any drain that stalls (PDB, long-running job) will pause the whole rolling update. I'd also make sure I have enough spare capacity (`maxSurge`) so drained pods have somewhere to go.

The senior nuance: the order is non-negotiable. Control plane first, then addons, then nodes. Skipping addons is the most common mistake — it looks fine until you hit the removed API."

---

## Further reading / watching

- **EKS upgrade guide**: [docs.aws.amazon.com/eks/latest/userguide/update-cluster.html](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) — the step-by-step official procedure.
- **EKS addon compatibility matrix**: search "EKS managed addons compatibility" on docs.aws.amazon.com. The table by K8s version is the reference you'll use every upgrade.
- **`pluto`** (Fairwinds): [github.com/FairwindsOps/pluto](https://github.com/FairwindsOps/pluto) — scans Helm releases and live cluster resources for deprecated or removed APIs. Run it in pre-flight.
- **EKS Best Practices — Cluster Upgrades**: [aws.github.io/aws-eks-best-practices/upgrades/](https://aws.github.io/aws-eks-best-practices/upgrades/) — the most complete checklist-style guide with gotchas.

---

## Next: the deep-dive

When the highway analogy and the four phases feel obvious, jump to [`eks-upgrades.md`](./eks-upgrades.md). The deep-dive covers:

- Pre-flight: deprecated API checks, addon matrix, PDB audit
- Control plane upgrade mechanics (what AWS does; what you monitor)
- Addon upgrade order and version pinning strategy
- In-place rolling vs blue-green node strategies with trade-offs
- The kubelet skew policy in detail
- PDB interaction with node drains — the exact failure mode
- 4-dimensions framing for interview
