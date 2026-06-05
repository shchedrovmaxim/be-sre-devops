# CNI comparison: Cilium vs Calico vs AWS VPC CNI — the simple version (the highway analogy)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

One idea:

> **A CNI plugin is the contractor that wires every new pod into the network. The question is which contractor to hire — and the answer depends on whether you want a private road, a shared highway, or AWS's own expressway.**

That's it. Each CNI has a different answer to "how do packets actually move?" and that answer determines what you can observe, what you can enforce, and what breaks at scale.

---

## The highway analogy

Imagine your pods are cars. They need roads to talk to each other.

| What you see | What it maps to |
|---|---|
| AWS VPC: pods drive on the actual public highway (VPC subnet lanes) | **VPC CNI** — each pod gets a real VPC IP, routes through the actual subnet fabric |
| Calico: pods drive on private local roads that have a highway on-ramp via iptables | **Calico iptables** — pods get an overlay or BGP-routed IP; iptables is the toll booth checking every car |
| Cilium: pods get their own fast lane enforced by a kernel traffic cop that sits in-line | **Cilium eBPF** — packets are intercepted at the kernel level, no toll booth, no overhead |

The "toll booth" (iptables) metaphor is the key one. At 1,000 Services, the toll booth has 10,000 rules it checks *for every packet*. That gets slow. Cilium's traffic cop makes one hash lookup instead.

---

## The whole idea in 2-3 sentences

Every CNI plugin does the same job (give pods IPs, wire up routing) but via completely different kernel mechanisms. AWS VPC CNI attaches real ENIs to nodes and gives pods VPC-native IPs — zero overlay, zero encapsulation, but you'll run out of IPs fast. Calico uses iptables or eBPF to implement NetworkPolicy and routing; Cilium goes all-in on eBPF, replaces kube-proxy, adds L7 policy, and surfaces per-flow observability via Hubble.

---

## The 2 concepts that confuse people

### 1. "eBPF" sounds like magic — it's just fast kernel code

eBPF lets you run small verified programs *inside the kernel* without writing a kernel module. Cilium uses this to intercept packets at the socket/XDP level — before they ever go through iptables. The upshot: no linear rule scan, no conntrack overhead for pod-to-pod traffic, and rich per-flow telemetry with basically no overhead.

Think of it as: instead of a rulebook (iptables) that the kernel reads cover-to-cover for each packet, eBPF is a lookup table. O(1) instead of O(N).

### 2. VPC CNI's IP exhaustion problem

In EKS, VPC CNI gives each pod a real ENI secondary IP. Each EC2 instance type has a hard limit on how many secondary IPs it can hold (`m5.large` = 3 ENIs × 10 IPs = 30 max pods). On a large cluster, you burn through your `/19` subnet shockingly fast. The AWS VPC CNI with prefix delegation (`--enable-prefix-delegation`) helps (allocates `/28` prefixes instead of single IPs), but you still need to plan for this.

Cilium and Calico use overlay networks (VXLAN or GENEVE) or BGP peering, so pod IPs come from a separate CIDR — no VPC IP exhaustion.

---

## Intuition cheat sheet

| Question | VPC CNI | Calico | Cilium |
|---|---|---|---|
| How does routing work? | Real VPC subnet routes via ENI | BGP or VXLAN overlay | eBPF socket-level interception |
| NetworkPolicy support? | None (delegated to Calico/Cilium add-on) | Standard K8s NetworkPolicy | Standard + CiliumNetworkPolicy (L7 HTTP/gRPC) |
| kube-proxy replacement? | No | Partial (with eBPF mode) | Yes — full kube-proxy replacement |
| IP source: VPC real IPs? | Yes — pods use VPC IPs | No — pod CIDR separate | No — pod CIDR separate |
| Scale ceiling pain? | IP exhaustion at large pod counts | iptables O(N) rule scan | Hash table O(1) — handles large clusters well |
| Deep AWS integration (SGs for pods)? | Yes — native | No | No |
| Observability (flows, L7 metrics)? | CloudWatch | Limited | Hubble — per-flow, L7 labels |
| When to pick it? | Need SGs for pods, tight AWS integration | Mature on-prem or BGP requirements | New cluster, need L7 policy or eBPF perf |

---

## Self-test (the killer interview question)

Out loud:

> **"Why might you pick Cilium over Calico for a new EKS cluster in 2026?"**

**Reference answer (intuitive version):**

"Three reasons. First, **eBPF vs iptables**: Calico in its default mode processes NetworkPolicy through iptables — one rule chain per Service/policy. At 5,000 Services that's thousands of rules evaluated *per packet*. Cilium uses eBPF hash tables — one lookup regardless of cluster size. For a new cluster that might grow, you want the O(1) path.

Second, **kube-proxy replacement**: Cilium can fully replace kube-proxy with eBPF, removing another iptables sprawl point. Calico's eBPF mode can do this too, but Cilium's implementation is more mature and more widely production-tested in 2026.

Third, **L7 policy and observability**: Calico NetworkPolicy is L3/L4 (IPs and ports). CiliumNetworkPolicy adds L7 — you can say 'only allow GET /health from this namespace', which is a qualitatively different security posture. And Hubble gives you per-flow observability without a service mesh — that's operationally huge.

The one case where I'd stick with VPC CNI is if I need native AWS Security Groups for pods — VPC CNI is the only CNI that integrates there. For everything else on a new EKS cluster in 2026, Cilium is the default choice."

---

## Further reading / watching

- **Cilium docs — Architecture overview**: `docs.cilium.io/en/stable/overview/intro/` — explains the eBPF datapath clearly
- **Liz Rice — "What is eBPF?"** (O'Reilly free ebook): search "Liz Rice eBPF O'Reilly report" — 60 pages, the clearest intro to eBPF internals
- **Cilium — "CNI Benchmark 2023"** on `cilium.io/blog` — real numbers on iptables vs eBPF at scale (look for the bandwidth and latency comparisons)
- **AWS VPC CNI GitHub**: `github.com/aws/amazon-vpc-cni-k8s` — the README explains ENI secondary IP allocation and prefix delegation
- **Thomas Graf (Cilium co-creator) — "eBPF, Past, Present, and Future"** on YouTube — the authoritative talk on why eBPF matters for networking

---

## Next: the deep-dive

When the highway analogy and the eBPF vs iptables distinction feel solid, jump to [`cni-comparison.md`](./deep-dive.md). The deep-dive covers:

- Exactly how each CNI assigns IPs at pod creation (the CNI plugin binary flow)
- Calico BGP peering mode and when it matters
- Cilium's eBPF datapath in detail: XDP, tc hooks, sockmap
- The ENI secondary IP math for VPC CNI (with instance type tables)
- CiliumNetworkPolicy L7 examples
- Hubble architecture and what you can see that you can't see elsewhere
- The 4-dimensions framing (Tech / People / CI/CD / Operations)
