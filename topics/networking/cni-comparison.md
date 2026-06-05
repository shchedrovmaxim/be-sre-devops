# CNI comparison: Cilium vs Calico vs AWS VPC CNI — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why might you pick Cilium over Calico for a new EKS cluster in 2026?"** — naming the datapath mechanism (eBPF vs iptables), the observability difference (Hubble vs nothing), the L7 policy gap, the kube-proxy replacement story, and the one case (SGs for pods) where VPC CNI wins.

> Start with the [simple version](./cni-comparison-simple.md) first if you haven't. The highway analogy is the spine.

---

## The senior framing — CNI is a policy and performance decision, not just plumbing

Mid-level engineers treat CNI as "the thing that gives pods IPs." Senior engineers know CNI choice is a **load-bearing architectural decision** that determines:

- Your NetworkPolicy expressiveness (L3/L4 vs L7 HTTP/gRPC)
- Your performance ceiling at 1,000+ Services
- Your observability story (can you see per-flow traffic without a service mesh?)
- Your IP address budget (VPC CNI) vs encapsulation overhead (overlays)
- Your blast radius when the CNI has a bug at 3 AM

Changing CNI on a running cluster is painful. Pick once, pick deliberately.

---

## Mental model: the CNI plugin binary flow

When kubelet creates a pod, it calls into the CNI plugin (a binary on the node) via a well-defined interface:

```
kubelet → CRI (containerd) → creates network namespace
                           → calls CNI plugin binary with ADD command
                           → plugin configures veth pair, IP, routes
                           → plugin returns assigned IP to kubelet
```

Every CNI does this. What differs is *how* they implement the rest:

| Step | VPC CNI | Calico | Cilium |
|---|---|---|---|
| IP source | ENI secondary IP pre-warmed by `aws-node` DaemonSet | Pod CIDR (IPAM varies) | Pod CIDR (IPAM: cluster-pool, ENI, etc.) |
| Routing | VPC route table (native) | BGP (on-prem/bare metal) or VXLAN overlay | eBPF redirect map or VXLAN overlay |
| Policy enforcement | None (delegate to Calico add-on for policy) | iptables chains via `calico-node` | eBPF programs via `cilium-agent` |
| kube-proxy | Standard iptables kube-proxy | Standard, or eBPF replacement | Full eBPF kube-proxy replacement |

---

## AWS VPC CNI — the native path

### How it works

The `aws-node` DaemonSet runs on every node. It calls EC2 APIs to attach additional ENIs to the node and pre-warm a pool of secondary private IPs. When a pod starts, `aws-node` assigns one of those pre-warmed IPs to the pod's network namespace.

```
Node: m5.large
  ENI 1 (primary): 10.0.1.5 (node IP)
    Secondary IPs: 10.0.1.10, 10.0.1.11, 10.0.1.12 ...
  ENI 2 (secondary): 10.0.1.50
    Secondary IPs: 10.0.1.51 ... 10.0.1.60

Pod gets 10.0.1.10 → it's a real VPC IP → routable anywhere in the VPC
```

No overlay. No encapsulation. A packet from pod A to pod B takes the same path as a packet between two EC2 instances — through the VPC routing fabric.

### The IP exhaustion math

Each EC2 instance type has a hard limit: `max_ENIs × IPs_per_ENI`.

| Instance | Max ENIs | IPs/ENI | Max pods (without prefix delegation) |
|---|---|---|---|
| t3.medium | 3 | 6 | 17 (15 secondary + 2 ENIs used) |
| m5.large | 3 | 10 | 29 |
| m5.2xlarge | 4 | 15 | 58 |
| m5.8xlarge | 8 | 30 | 234 |

A cluster with 50× `m5.large` nodes can run ~1,450 pods max — before VPC CIDR exhaustion. With **prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`), each slot holds a `/28` (16 IPs) instead of a single IP, multiplying capacity by 16. But you still need to plan subnet sizing carefully: a `/19` gives you 8,192 IPs, and a busy cluster can eat through that.

### When VPC CNI is the right choice

- You need **AWS Security Groups for pods** — VPC CNI is the only CNI that supports `SecurityGroupPolicy` CRDs (assigns an EC2 SG directly to a pod's ENI). This is a hard requirement if your compliance team says "pods must be in SGs, not just behind NetworkPolicy."
- Your workload is **latency-sensitive and cross-VPC** — real VPC IPs mean no decapsulation overhead, transit gateway routing works naturally, VPC flow logs capture pod-level traffic.
- Your team is **fully AWS-native** and you want one fewer moving part.

### VPC CNI gotchas

- Enabling prefix delegation on existing clusters requires draining nodes — it's a node-level change.
- `aws-node` runs in `hostNetwork: true` mode; a bug in it takes down pod networking on the node.
- No native NetworkPolicy support — you must add Calico or Cilium as a policy engine add-on (without their CNI parts).

---

## Calico — the iptables workhorse (and its eBPF mode)

### How it works — iptables mode

Calico creates a `veth` pair for each pod (one end in the pod namespace, one on the host). Routing is either:

- **BGP** (bare metal, on-prem): Calico peers with your routers and advertises pod CIDRs. No overlay. Packets route natively. Used heavily on-prem and with MetalLB.
- **VXLAN/IPIP overlay** (cloud, default EKS): pod packets are encapsulated in UDP/VXLAN before leaving the node. Works everywhere but adds ~50–100 µs overhead and ~10% throughput reduction in some workloads.

NetworkPolicy is enforced by `calico-node` writing iptables rules. For each pod, Calico creates a chain in the `FORWARD` table. For each NetworkPolicy that selects that pod, Calico creates match rules in that chain.

```
# Simplified iptables view:
-A cali-to-wl-dispatch -i calid+ -g cali-tw-<pod-iface>
-A cali-tw-<pod-iface> -m mark --mark 0x... -j RETURN  # allowed
-A cali-tw-<pod-iface> -j DROP                          # default deny
```

At 1,000 pods with 5 policies each → ~5,000–10,000 iptables rules. Every packet walks this chain. The kernel processes it linearly — O(N) where N = rules.

### Calico eBPF mode

Calico has had an eBPF mode since v3.13. It:
- Replaces kube-proxy with eBPF
- Implements NetworkPolicy via eBPF programs instead of iptables
- Reduces conntrack table usage

It's production-ready but less mature than Cilium's eBPF implementation. Feature parity with Cilium's L7 policy is not there. If you want eBPF on Calico, you should benchmark your specific workload — the gains are real but the operational experience is less documented than Cilium.

### When Calico is the right choice

- **On-prem with BGP infrastructure** — Calico's BGP mode is the gold standard for bare-metal K8s. You get real routable pod IPs without a cloud vendor's fabric.
- **Existing clusters already running Calico** — migration to Cilium is disruptive. Calico at L3/L4 is extremely well-understood.
- **Team has deep iptables expertise** — the debugging tools are well-known (`calicoctl`, `iptables-save`, `cali-*` chains).

### Calico gotchas

- **iptables at scale**: at 5,000+ Services (each adding rules), iptables rule processing becomes measurable in latency. The typical inflection point is 5,000–10,000 rules, which you can hit easily in large multi-tenant clusters.
- **iptables locking**: the kernel iptables lock (`ip_tables_mutex`) is a global lock. Under high concurrency, rule updates (during pod churn) and packet processing contend on it. This is the often-cited failure mode in clusters with high pod churn (CI clusters, batch clusters).
- **IPIP/VXLAN overhead**: the default cloud mode adds encapsulation. Measure with your actual workload before assuming it's negligible.

---

## Cilium — the eBPF-native CNI

### How it works

Cilium runs `cilium-agent` on every node. Instead of iptables chains, it compiles eBPF programs and loads them into the kernel at `tc` (traffic control) hooks and socket-level hooks (`sockmap`). The packet path looks like:

```
Pod A socket → sockmap redirect → Pod B socket   (same node, no kernel stack traversal)
Pod A → tc eBPF hook → decision → veth → physical → tc eBPF hook on Pod B's node
```

The critical difference: for same-node pod-to-pod traffic, Cilium can use `sockmap` to redirect at the socket layer — packets never go through the full IP stack. This is why you see latency numbers 20-40% lower than iptables mode in same-node benchmarks.

For Service routing (ClusterIP → Endpoint), Cilium uses eBPF maps (hash tables) instead of iptables `DNAT` rules:

```
lookup(service_ip:port) → {endpoint_ip:port}   # O(1) hash table lookup
```

Compare to kube-proxy's iptables: `PREROUTING → KUBE-SVC-XXXX → KUBE-SEP-XXXX → DNAT` — a linear chain walk for every packet.

### eBPF performance numbers

Real benchmark numbers from Cilium's published benchmarks (2023, their own cluster — treat as directional, not gospel):

| Metric | kube-proxy iptables | Cilium eBPF |
|---|---|---|
| 1-hop HTTP throughput (same node) | baseline | +30-40% |
| Service lookup latency | O(N) rules | O(1) hash |
| CPU overhead at 10k Services | measurable iptables lock contention | flat |
| Pod-to-pod latency (same node) | ~50 µs | ~15 µs (sockmap path) |

The numbers vary by workload — connection-heavy workloads see more gain than large-transfer workloads. But the O(N) → O(1) principle is real and not benchmarkable away.

### kube-proxy replacement

With `kubeProxyReplacement: strict` in the Cilium config, `cilium-agent` handles all Service routing. kube-proxy is not deployed. This removes the entire iptables KUBE-* chain family.

The DaemonSet scheduling impact: one fewer DaemonSet (kube-proxy) running on every node. For cost-sensitive large clusters this can matter (freed CPU/memory on small nodes).

### CiliumNetworkPolicy — L7 policy

Standard K8s `NetworkPolicy` is L3/L4: IP blocks, namespaceSelectors, ports. You can say "allow port 8080 from namespace A." You cannot say "allow GET /health from namespace A but not POST /admin."

`CiliumNetworkPolicy` adds L7:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-payments-only
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: checkout
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/payment/status"
        - method: "POST"
          path: "/payment/charge"
```

This is enforced in the eBPF datapath — not in a sidecar, not via an Envoy proxy per pod. The performance overhead is minimal because the enforcement happens at the socket level.

### Hubble — per-flow observability

Hubble is Cilium's observability layer. Because Cilium sees every packet in eBPF, it can export per-flow metadata:

- Source/destination pod (with K8s labels, not just IPs)
- HTTP method, path, status code
- DNS queries and responses
- TCP connection state
- Policy verdict (allowed / dropped / redirected)

You get this **without a sidecar and without sampling**. The equivalent in a Calico/kube-proxy world requires either a service mesh (Istio + Envoy sidecars, with the overhead and complexity that brings) or packet capture.

```bash
# Hubble CLI — live flow inspection
hubble observe --namespace payments --protocol http
# TIMESTAMP         SOURCE                DESTINATION           TYPE          VERDICT
# 2026-06-05T10:00  payments/checkout-7   payments/payment-api  HTTP GET /... FORWARDED
# 2026-06-05T10:00  payments/rogue-svc    payments/payment-api  HTTP POST/..  DROPPED (policy)
```

---

## The comparison matrix (full)

| Property | VPC CNI | Calico (iptables) | Calico (eBPF) | Cilium (eBPF) |
|---|---|---|---|---|
| Routing mechanism | Real ENI/VPC routes | BGP or VXLAN/IPIP | BGP or VXLAN/IPIP | eBPF or VXLAN |
| NetworkPolicy | None (add-on needed) | Standard (L3/L4) | Standard (L3/L4) | Standard + L7 |
| kube-proxy replacement | No | No | Yes (limited) | Yes (full) |
| Service lookup complexity | iptables O(N) | iptables O(N) | eBPF O(1) | eBPF O(1) |
| Same-node pod latency | VPC route | iptables | eBPF | eBPF sockmap |
| Observability | CloudWatch flow logs | Limited | Limited | Hubble (L7, labels) |
| SGs for pods | Yes (native) | No | No | No |
| IP exhaustion risk | High (VPC IPs) | Low | Low | Low |
| Operational maturity | Highest (AWS-managed) | High (10y+ battle-tested) | Medium | High (growing rapidly) |
| Good for | AWS-native, SG-per-pod | On-prem BGP, existing installs | On-prem wanting eBPF | New clusters, L7 policy, observability |

---

## The interview answer in 60 seconds

> "I'd pick Cilium for a new EKS cluster in 2026 for three reasons.
>
> First, **performance**: Calico's default mode uses iptables — one rule chain per Service, O(N) lookup per packet. At scale, that becomes measurable. Cilium uses eBPF hash tables for Service routing — O(1) regardless of cluster size. Same-node pod-to-pod traffic goes through sockmap, skipping the IP stack entirely.
>
> Second, **kube-proxy replacement**: Cilium can replace kube-proxy entirely with its eBPF implementation, removing the iptables KUBE-* chain sprawl completely. That's one fewer DaemonSet and one fewer iptables footprint to debug.
>
> Third, **L7 policy and Hubble**: CiliumNetworkPolicy lets you write HTTP-level rules (allow GET /health, deny POST /admin) enforced in the kernel, not in a sidecar. And Hubble gives you per-flow observability with K8s labels without a service mesh. That's operationally huge — you can see exactly which service is hitting which endpoint, which policies are dropping traffic, all from the CLI.
>
> The one exception: if I need AWS Security Groups assigned per pod — only VPC CNI supports that. Otherwise, Cilium wins on a new cluster."

---

## Self-test drills

### 1. Why might you pick Cilium over Calico for a new EKS cluster in 2026?

**Reference answer:** Three reasons — eBPF vs iptables O(1) vs O(N) Service lookup; full kube-proxy replacement removing iptables sprawl; L7 CiliumNetworkPolicy + Hubble per-flow observability without a sidecar. Exception: VPC CNI wins when SGs-per-pod are required.

### 2. What is the IP exhaustion problem with VPC CNI and how do you mitigate it?

**Reference answer:** Each pod gets a real VPC secondary IP from the node's ENI. Each EC2 instance type has a hard limit (`max_ENIs × IPs_per_ENI`). On `m5.large` that's 29 max pods. At scale with a `/19` subnet you can exhaust IPs. Mitigation: **prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`) allocates `/28` prefixes instead of single IPs, multiplying pod capacity ~16×. Still need proper subnet sizing and planning.

### 3. How does Calico enforce NetworkPolicy at the kernel level?

**Reference answer:** Calico writes iptables rules into the `FORWARD` table. For each pod, a dispatch chain routes to a pod-specific chain. For each policy selecting that pod, rules in that chain allow or deny based on IP/port. At scale, the linear rule walk becomes measurable. Calico's eBPF mode replaces this with BPF maps (hash tables), same O(1) property as Cilium, but is less mature for L7 and production-tested less widely.

### 4. What's the difference between a standard NetworkPolicy and a CiliumNetworkPolicy?

**Reference answer:** Standard NetworkPolicy is L3/L4 — you match on namespaceSelectors, podSelectors, and port numbers. CiliumNetworkPolicy extends this to L7 — you can match HTTP methods, paths, gRPC service/method names, and even Kafka topics. Enforcement is in the eBPF datapath (not a sidecar), so the performance overhead is minimal. This enables "allow only GET /health from namespace X, deny everything else" without any proxy.

---

## Further reading / watching

- **Cilium docs — Architecture**: `docs.cilium.io/en/stable/overview/component-overview/` — the canonical architecture overview
- **Liz Rice — "What is eBPF?"** (O'Reilly free ebook): the clearest eBPF introduction; covers TC hooks, XDP, sockmap — all the hooks Cilium uses
- **Thomas Graf (Cilium co-creator) — "eBPF, Past, Present, and Future"** on YouTube — motivation and architecture from the author
- **Cilium blog — CNI performance benchmarks**: `cilium.io/blog` — search for "benchmark" or "eBPF performance"; take numbers directionally
- **AWS VPC CNI docs — prefix delegation**: AWS docs, search "VPC CNI prefix delegation EKS"
- **Calico docs — eBPF dataplane**: `docs.tigera.io` — the eBPF mode docs explain when/how to enable it and known limitations

---

## The 4 dimensions (senior framing)

- **Tech**: iptables O(N) vs eBPF O(1) for Service routing; VPC CNI native IPs vs overlay encapsulation; CiliumNetworkPolicy L7 in the datapath; Hubble as a sidecar-free observability layer; SGs-for-pods as the VPC CNI moat.
- **People**: Calico is more widely known in the ops community — more blog posts, more StackOverflow answers. Cilium is the direction the community is heading but requires some eBPF literacy for debugging. Budget for training if migrating an existing team. Hubble's CLI is genuinely operator-friendly — show it to skeptical teammates, it's compelling.
- **CI/CD**: CNI choice is a cluster-level decision, not a workload-level one. Validate CNI behavior in CI with network policy tests (e.g., `netpol-tester` or simple `kubectl exec` connectivity checks). Don't assume policy works — test it as part of cluster provisioning automation. Cilium has `cilium connectivity test` built-in.
- **Operations**: changing CNI on a live cluster means node drain and replacement — plan it like a migration, not a patch. Monitor eBPF map saturation (`cilium status`, `bpftool map show`) — maps have size limits. Monitor `kube-proxy` iptables rule count if still using it (`iptables -L | wc -l`). For Hubble, expose the Hubble relay as a Service and use `hubble observe` in your standard debugging workflow.
