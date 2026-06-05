# iptables / nftables — the simple version (the post office sorting analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **iptables is a set of rules the kernel uses to inspect and redirect network packets as they travel through the networking stack. The kube-proxy Service IP → pod IP magic is just DNAT rules in the `nat` table's `PREROUTING` chain. Every packet arriving at a Service IP gets its destination rewritten to a real pod IP before it even reaches the application.**

That's the whole thing. Once you see it as "packet arrives, rules run, destination gets rewritten, packet continues," the rest is detail.

---

## The post office sorting analogy

Imagine a post office where every package (network packet) goes through a sorting room before delivery. The sorting room has:

- **Tables** — different types of work (one table handles address changes, another handles blocking packages, another handles marking packages for special treatment)
- **Chains** — ordered lists of sorting instructions at each table
- **Rules** — individual instructions in each chain ("if the package is addressed to PO Box 10 (Service IP), change it to 42 Main Street (pod IP)")

A package arrives. The sorting room runs it through the relevant chains in order, applying rules. If a rule matches, it takes an action (rewrite the address, block the package, let it through). At the end, the package is delivered with potentially a different destination.

That's iptables. The packet arrives → rules in chains in tables run → packet goes where the rules say.

---

## The 5 tables (types of work)

| Table | What it does | When you'd touch it |
|---|---|---|
| `filter` | Allow or block packets | Firewall rules, Network Policies |
| `nat` | Rewrite source or destination addresses | Service IP → pod IP (kube-proxy), masquerading |
| `mangle` | Modify packet headers (TTL, marks) | Traffic marking for QoS |
| `raw` | Bypass conntrack for specific traffic | High-performance flows, avoid conntrack overhead |
| `security` | Set SELinux security marks | Rare; SELinux integration |

For K8s purposes: you care about **`filter`** (allows/blocks) and **`nat`** (address rewriting). The others are rarely touched in normal operations.

---

## The 5 chains (sorting stages)

Packets travel through the kernel along a path, and iptables has "hook points" along that path:

```
packet in
    ↓
PREROUTING    ← first stop for incoming packets (nat table's DNAT happens here)
    ↓
  (routing decision: is this for us? or should it be forwarded?)
    ↓         ↓
 INPUT    FORWARD          INPUT: packet destined for this machine
    ↓         ↓            FORWARD: packet just passing through
(local)  POSTROUTING
    ↓         ↓
OUTPUT   POSTROUTING ← last stop before packets leave (SNAT/masquerade happens here)
    ↓
packet out
```

**The K8s key insight**: Service IP → pod IP rewriting happens at **PREROUTING** in the **nat** table. By the time the routing decision runs, the destination is already the pod IP — so routing sends it to the right pod.

---

## How kube-proxy's Service IP magic actually works

A Service has IP `10.96.0.1` (ClusterIP) with 3 pods at `10.244.0.5`, `10.244.0.6`, `10.244.0.7`.

kube-proxy creates iptables rules that look roughly like:

```
# PREROUTING (nat table): match packets going to the Service IP
-A PREROUTING -d 10.96.0.1/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

# KUBE-SVC-XXXX: randomly pick one of 3 endpoints (1/3 probability each)
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.333 -j KUBE-SEP-POD1
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.500 -j KUBE-SEP-POD2
-A KUBE-SVC-XXXX -j KUBE-SEP-POD3

# KUBE-SEP-POD1: rewrite destination to pod 1's IP
-A KUBE-SEP-POD1 -j DNAT --to-destination 10.244.0.5:8080
```

The packet comes in addressed to `10.96.0.1:80`. PREROUTING rewrites it to `10.244.0.5:8080`. The routing table sends it to pod 1's network. conntrack remembers the rewrite so the reply packet gets its source rewritten back to `10.96.0.1` automatically.

The user's code that made the request to `10.96.0.1:80` gets a reply that looks like it came from `10.96.0.1:80`. The DNAT/conntrack machinery is invisible.

---

## conntrack — the memory that makes DNAT work

conntrack is the kernel's connection tracking table. When DNAT rewrites a destination, conntrack records the original flow → rewritten flow mapping. When the reply packet comes back from the pod, conntrack automatically rewrites the source address from the pod IP back to the Service IP.

Without conntrack, DNAT would be one-way — you could send a packet to the pod, but the reply would come from the pod IP, confuse the client, and the connection would fail.

```bash
# See conntrack table entries on a node
conntrack -L | grep 10.96.0.1
# tcp  6 300 ESTABLISHED src=10.244.0.3 dst=10.96.0.1 sport=45678 dport=80
#   → src=10.244.0.5 dst=10.244.0.3 sport=8080 dport=45678 [ASSURED]
```

---

## nftables — why it's replacing iptables

nftables is the modern Linux packet filtering framework. Same concept, different implementation:

| | iptables | nftables |
|---|---|---|
| Data structure | Linked list of rules (linear scan) | Hash tables and interval trees (faster lookups) |
| Atomic updates | Rules applied one by one (can get inconsistent mid-update) | Atomic transactions (full ruleset applied atomically) |
| Table model | Separate tools per table (iptables, ip6tables, arptables, ebtables) | Single tool for all (`nft`) |
| K8s support | kube-proxy's default until recently | kube-proxy has nftables mode (K8s 1.29+ beta) |

For K8s, the important bit: kube-proxy in **iptables mode** (the default through K8s 1.28) creates thousands of iptables rules for large clusters, and rule evaluation is O(n) per packet. At 10,000 Services, that's noticeable latency. nftables mode and Cilium (which replaces kube-proxy entirely with eBPF) solve this.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What's a chain? | An ordered list of rules at a specific hook point in the packet path. |
| What's a table? | A type of work: filter=allow/block, nat=rewrite addresses, mangle=modify headers. |
| Where does Service DNAT happen? | PREROUTING chain, nat table. Before routing. |
| What's conntrack? | The connection tracking table that makes DNAT two-way (reply packets get source-rewritten automatically). |
| What does kube-proxy do? | Watches the K8s API for Services/Endpoints, writes iptables rules to implement the DNAT load balancing. |
| What's nftables? | The modern replacement for iptables — atomic updates, faster lookups, single tool. |
| What's Cilium? | A CNI plugin that replaces kube-proxy entirely using eBPF — no iptables rules at all. |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through how kube-proxy's iptables mode actually routes a Service IP to a pod IP."**

**Reference answer (intuitive version):**

"When a Service is created, kube-proxy watches the API server and writes iptables rules in the `nat` table's `PREROUTING` chain. The rules match packets destined for the Service's ClusterIP and port, then randomly DNAT them to one of the Service's ready endpoint pod IPs — using `--mode random --probability` to distribute load.

The packet arrives at the node addressed to the Service IP. PREROUTING rules run, match the Service IP, pick a pod endpoint with probability math, and rewrite the packet's destination to that pod's IP and port. The kernel's routing then sends the packet to the pod.

conntrack records this rewrite. When the pod sends the reply, conntrack automatically rewrites the source back to the Service IP — so the original caller sees a clean response from the Service IP, not the pod IP.

For kube-proxy, this is all implemented as chains: `KUBE-SERVICES` → `KUBE-SVC-<hash>` (for the Service) → `KUBE-SEP-<hash>` (for each endpoint). Large clusters can have thousands of these chains, which is why nftables mode and Cilium eBPF exist as more scalable alternatives."

---

## Further reading / watching

- **iptables man page**: `man iptables` on any Linux host
- **Kubernetes networking deep dive by Ahmet Alp Balkan**: search "k8s networking iptables" on his GitHub/blog
- **Cilium docs on eBPF vs iptables**: [docs.cilium.io/en/stable/concepts/ebpf/](https://docs.cilium.io/en/stable/concepts/ebpf/)

---

## Next: the deep-dive

When the post office analogy feels obvious, jump to [`iptables-nftables.md`](./deep-dive.md). The deep-dive covers:

- All 5 tables and 5 chains with the full packet traversal order
- kube-proxy iptables rules in detail (KUBE-SERVICES, KUBE-SVC-*, KUBE-SEP-*)
- conntrack in depth — state machine, table, DNS UDP problem
- nftables as the modern replacement — atomic updates, tree lookups, the K8s migration story
- How Cilium and Calico use or bypass iptables
- 4 self-test drills
- The 4 dimensions framing
