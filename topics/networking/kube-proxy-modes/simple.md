# kube-proxy modes — the simple version (the telephone switchboard analogy)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

One idea:

> **kube-proxy is a telephone switchboard operator. When a call comes in for "the payment service", the operator looks up who's answering that number today and reroutes the call. iptables mode has the operator reading through a printed directory from top to bottom for every call. IPVS mode has a hash table — instant lookup. Cilium eBPF mode removes the operator entirely and bakes the lookup into the phone hardware.**

The printed directory is fine when you have 10 Services. At 5,000 Services it's thousands of pages long, and you read the whole thing for every single call.

---

## The telephone switchboard analogy

| Switchboard world | K8s world |
|---|---|
| A call comes in for "payment service" | A packet arrives destined for a ClusterIP |
| Operator reads a printed directory (top to bottom) | iptables chain walk — linear rule evaluation |
| At 5,000 services: thousands of pages to read per call | Measurable per-packet overhead |
| Operator with a hash-indexed rolodex | IPVS mode — O(1) hash lookup |
| Remove the operator; lookup is baked into the phone's firmware | Cilium eBPF mode — kube-proxy gone entirely |
| Two operators contending over the same printed directory | iptables global lock under pod churn |

---

## The whole idea in 2-3 sentences

kube-proxy is a DaemonSet that watches the API server for Service/Endpoint changes and programs the node's kernel to redirect ClusterIP traffic to real pod IPs. In iptables mode, it writes a chain of NAT rules; every packet walks the chain linearly — O(N) where N is the number of Services. IPVS mode uses kernel hash tables (O(1)) and is the right choice for clusters with hundreds of Services; Cilium eBPF replaces kube-proxy entirely, handling Service routing in eBPF programs without any iptables.

---

## The 2 concepts that confuse people

### 1. Why O(N) is a problem — the math

At 500 Services with 5 endpoints each, iptables has roughly 2,500–5,000 rules in the NAT table. At 5,000 Services: 25,000–50,000 rules. Every packet to a ClusterIP evaluates these rules **linearly until a match**. At scale:

- Rule updates during pod churn acquire the global iptables lock (`ip_tables_mutex`)
- Lock contention causes packet drops under high pod churn
- Observed latency tail grows as the rule table grows

The inflection point is typically around 1,000–5,000 Services. Below that, iptables is invisible. Above that, you start measuring it.

### 2. iptables mode vs IPVS mode — what actually changes

iptables mode: kube-proxy writes `KUBE-SVC-*` and `KUBE-SEP-*` chains in the NAT table. Each ClusterIP lookup involves jumping through 2-3 chains before hitting DNAT.

IPVS mode: kube-proxy uses the kernel's Virtual Server module. Service IPs are programmed as virtual servers, endpoints as real servers. Lookup is a hash table lookup — O(1) regardless of Service count. Load balancing algorithms (round-robin, least-connections, weighted) are built in at the kernel level.

The migration path: `kube-proxy --proxy-mode=ipvs` and install the IPVS kernel modules (`ip_vs`, `ip_vs_rr`, `ip_vs_sh`, `ip_vs_wrr`). Most managed K8s offerings make this a config flag.

---

## Intuition cheat sheet

| Question | iptables mode | IPVS mode | Cilium eBPF |
|---|---|---|---|
| Lookup algorithm | O(N) linear chain walk | O(1) hash table | O(1) hash map |
| Scale inflection point | ~1,000–5,000 Services | Handles 10,000+ Services | Handles 10,000+ Services |
| Kernel module required | netfilter (always present) | `ip_vs_*` modules | eBPF (kernel 4.9+) |
| Load balancing algorithms | Random (probability rules) | rr, lc, wrr, sh and more | rr, maglev and more |
| kube-proxy still needed? | Yes | Yes | No — kube-proxy removed |
| Conntrack table impact | High — every Service → conntrack entry | Lower | Lower (bypasses conntrack for some paths) |
| Migration risk | Baseline | Low — just flip mode | Medium — remove kube-proxy DaemonSet |

---

## Self-test (the killer interview question)

Out loud:

> **"Why might iptables mode become a problem at 5,000 Services?"**

**Reference answer (intuitive version):**

"Three reasons. First, the **O(N) lookup**: iptables evaluates rules linearly — at 5,000 Services with roughly 5 endpoints each, that's ~25,000–50,000 rules walked per packet to a ClusterIP. The kernel has to match each rule against the packet's 5-tuple until it finds a hit.

Second, the **global lock**: when kube-proxy updates rules — adding or removing endpoints during pod churn — it takes the global iptables lock (`ip_tables_mutex`). Under high pod churn (CI clusters, spot instances cycling, batch jobs) this lock contention causes packet drops and kube-proxy update latency that compounds into Service flapping.

Third, **memory growth**: each Service adds iptables rules that consume kernel memory. There's no hard limit but at 100,000+ rules clusters have seen iptables becoming a measurable memory consumer.

The fixes: IPVS mode (O(1) hash table, same kube-proxy DaemonSet, just a different kernel module) or Cilium eBPF (removes kube-proxy entirely). For a new cluster above 500 Services I'd use IPVS at minimum; for a new cluster where I control the CNI, Cilium eBPF is the right call."

---

## Further reading / watching

- **Kubernetes docs — kube-proxy modes**: `kubernetes.io/docs/reference/networking/virtual-ips/` — covers iptables, IPVS, nftables
- **Cilium — kube-proxy replacement docs**: `docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/` — the full kube-proxy replacement setup guide
- **Kubernetes blog — "IPVS-Based In-Cluster Load Balancing Deep Dive"** (2018): search "IPVS kube-proxy Kubernetes blog" — the original deep-dive when IPVS mode was introduced; still the clearest technical explanation
- **Anurag Gupta (Google) — "Life of a Packet" KubeCon talk** on YouTube: traces a packet end-to-end through iptables — solidifies the mental model

---

## Next: the deep-dive

When the switchboard analogy and O(N) vs O(1) distinction feel solid, jump to [`kube-proxy-modes.md`](./deep-dive.md). The deep-dive covers:

- The exact iptables chain structure (KUBE-SVC, KUBE-SEP) with a concrete example
- IPVS module setup and the load balancing algorithms
- Cilium's eBPF Service map and how it replaces kube-proxy
- Real benchmark numbers (not benchmarked by me — from published sources, with caveats)
- The migration path from iptables → IPVS → Cilium and what breaks
- The 4-dimensions framing (Tech / People / CI/CD / Operations)
