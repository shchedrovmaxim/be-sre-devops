# AWS VPC CNI — the simple version (the parking lot)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

One idea:

> **VPC CNI gives every pod its own parking spot in the VPC. Each spot is a real IP address — not a tunnel, not a NAT, not an overlay. If you run out of spots, you can't park more pods.**

That's the whole concept. Everything else (ENI warm-pool, prefix delegation, IP exhaustion) is about how many spots you can have and how to get more.

---

## The parking lot analogy

| Parking lot | VPC CNI world |
|---|---|
| Parking lot = the node | EC2 instance with attached ENIs |
| Parking spots per lot | Secondary IPs per ENI × max ENIs per instance |
| Each car gets its own spot | Each pod gets its own VPC IP address |
| Cars can reach other cars directly (no ferry needed) | Pod-to-pod traffic is native VPC routing — no overlay, no encap |
| Lot attendant pre-warms spots before cars arrive | Warm pool: WARM_IP_TARGET IPs ready before pods are scheduled |
| Running out of spots = "no space available" | IP exhaustion: pending pods, `Insufficient free addresses in subnet` |
| "Compact parking" mode — bigger spot for multiple cars | Prefix delegation: one /28 prefix = 16 IPs per ENI slot instead of 1 |

The key difference from most other CNIs: **there is no overlay network**. On Calico BGP or Flannel, pod IPs are virtual — they're tunneled over the node's real IP. On VPC CNI, the pod's IP *is* a real AWS VPC IP, routable directly from anything in the VPC (other pods, ALBs, RDS, Lambda). No decap, no encap.

---

## The 2 concepts that confuse people

### 1. ENI/IP math — why do you run out of IPs so fast?

Each EC2 instance type has a fixed maximum number of ENIs, and each ENI has a fixed maximum number of secondary IP addresses. Multiply them, subtract 1 for the primary IP, and that's your max pod count per node.

Example for `m5.large`:
- Max ENIs: **3**
- Secondary IPs per ENI: **10**
- Max pods = (3 × 10) - 1 (primary node IP) = **29 pods**

AWS has a published table at [docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html). Search "instance ENI limit" to find it.

Some larger instance types look great on CPU/memory but terrible on pod density. `m5.xlarge` gives you 58 pods. `c5.large` gives you only 29. If you're running 100+ small microservices per node, instance type selection is about ENI limits as much as CPU.

### 2. Prefix delegation — the compact parking mode

Instead of allocating one IP per ENI slot, AWS can allocate a `/28` prefix — that's 16 IPs — per ENI slot. So `m5.large` goes from:
- Standard: 3 ENIs × 10 IPs = **29 pods**
- Prefix delegation: 3 ENIs × 10 prefixes × 16 IPs = **479 pods**

You enable it with one env var in the `aws-node` DaemonSet:
```bash
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
```

The catch: your subnet must have `/28` chunks available in its IP space. A fragmented subnet (from years of random allocations) may not have contiguous `/28` blocks. Plan subnet sizing with prefix delegation in mind from day one.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| How does VPC CNI assign pod IPs? | Each pod gets a secondary IP from an ENI attached to the node — a real VPC IP |
| Why is pod-to-pod routing fast? | No overlay. Pod IPs are native VPC IPs. Routing is just VPC routing. |
| What's the warm pool? | Pre-allocated IPs (and ENIs) sitting ready on each node, so pod startup doesn't wait for ENI attach |
| What's the bottleneck for pod density? | Number of ENIs × secondary IPs per ENI per instance type — hard AWS limit |
| What does prefix delegation do? | Allocates a /28 (16 IPs) per ENI slot instead of 1 IP — multiplies density by 16 |
| What are security groups for pods? | Give each pod its own SG (instead of sharing the node's SG) via ENI trunking |
| What breaks first under IP exhaustion? | Pods enter Pending state; scheduler can find nodes with capacity but no IPs to assign |

---

## Self-test (the killer interview question)

Out loud:

> **"You're hitting pod-density IP exhaustion on `m5.large` nodes. Walk through your options."**

**Reference answer (intuitive version):**

"First, confirm what's actually happening: `kubectl describe pod <pending-pod>` will show `0/N nodes are available: N Insufficient free addresses in subnet`. That's IP exhaustion, not CPU/memory.

Option 1 — **Enable prefix delegation**. One env var change on the `aws-node` DaemonSet. Goes from 29 IPs to up to 479 on `m5.large`. No node replacement needed immediately, but you need to recycle existing nodes to get them on the new mode. Also requires your subnets to have contiguous `/28` blocks available.

Option 2 — **Move to a larger instance type** with better ENI density. `m5.2xlarge` has 4 ENIs × 15 IPs = 58 pods. `m5.4xlarge` gets you 234. But if you have hundreds of tiny microservices, you hit the ceiling again.

Option 3 — **Custom networking**: route pod IPs through a secondary subnet with a larger CIDR, leaving your primary subnet for node IPs. Solves the subnet exhaustion part; doesn't fix the per-node ENI ceiling.

Option 4 — **Switch CNI to an overlay model** (Cilium in overlay mode, Flannel). Pods get virtual IPs; you're no longer limited by VPC IP space. Trade-off: you lose native VPC routing (pod IPs aren't reachable from outside the cluster without extra config), and you add the overhead of encapsulation.

The senior answer is: prefix delegation first (least disruption, biggest impact), combined with right-sizing instance types for your pod-density profile, and planning subnets with /28-friendly CIDR blocks from the start. If you genuinely need more than ~500 pods per node, switch CNI."

---

## Further reading / watching

- **AWS VPC CNI GitHub**: [github.com/aws/amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s) — the `README.md` and `docs/` folder have the warm-pool env vars and prefix delegation walkthrough.
- **EKS Best Practices Guide — Networking**: [aws.github.io/aws-eks-best-practices/networking/vpc-cni/](https://aws.github.io/aws-eks-best-practices/networking/vpc-cni/) — covers IP exhaustion, prefix delegation, and custom networking with diagrams.
- **EC2 ENI limits table**: docs.aws.amazon.com, search "EC2 instance ENI limits" — bookmark this; you'll reference it for capacity planning.
- **AWS blog — "Increase the amount of available IP addresses for your Amazon EKS nodes"**: search exact title. The prefix delegation announcement post explains the math clearly.

---

## Next: the deep-dive

When the parking lot analogy and the ENI math feel obvious, jump to [`vpc-cni.md`](./deep-dive.md). The deep-dive covers:

- The full ENI attach and IP warm-pool lifecycle
- All warm-pool tuning knobs (WARM_ENI_TARGET, WARM_IP_TARGET, MINIMUM_IP_TARGET)
- Prefix delegation in detail — the subnet fragmentation gotcha
- Security groups for pods (per-pod SGs via ENI trunking) and why it matters
- Custom networking for routing pods through different subnets
- The cost/density trade-off vs Cilium and other overlay CNIs
- 4-dimensions framing for interview
