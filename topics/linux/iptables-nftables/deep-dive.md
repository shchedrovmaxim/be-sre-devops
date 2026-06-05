# iptables / nftables

> **Goal**: by the end you can answer the killer question — **"Walk me through how kube-proxy's iptables mode actually routes a Service IP to a pod IP."** — covering the 5 tables, 5 chains, packet traversal order, the DNAT chain structure, conntrack's role, nftables as replacement, and how Cilium/Calico change the picture.

> Start with the [simple version](./simple.md) if you haven't — the post office sorting analogy is the mental model.

---

## The senior framing

iptables is not just a firewall. In Kubernetes, it's the **Service load balancer**. Every packet to a ClusterIP, NodePort, or LoadBalancer Service IP is intercepted and DNAT'd by iptables rules that kube-proxy manages. Understanding this is what separates "I know Services exist" from "I can debug why Service discovery is broken."

The senior nuance: iptables's linear rule evaluation doesn't scale. At 10,000 Services (a realistic large cluster), each packet evaluation touches ~40,000 rules. This is why the ecosystem is actively moving to nftables (K8s 1.29+) and eBPF-based networking (Cilium, Calico eBPF mode).

---

## The 5 tables

Tables define the type of work being done on packets. Each table has its own set of chains.

### `filter` table — the firewall

Default table. Contains the core accept/drop decisions.

Default chains: `INPUT`, `FORWARD`, `OUTPUT`.

K8s usage:
- `KUBE-FIREWALL` chain: drops packets with invalid mark bits
- Network Policy enforcement: Calico, Weave write rules here to implement NetworkPolicy allow/deny
- `KUBE-PROXY-FIREWALL` chain (K8s 1.25+): blocks traffic that shouldn't reach Services

### `nat` table — address translation

Used whenever source or destination addresses need to be rewritten.

Default chains: `PREROUTING`, `OUTPUT`, `POSTROUTING`.

K8s usage:
- **PREROUTING**: DNAT for ClusterIP Services (incoming pod-to-Service traffic)
- **OUTPUT**: DNAT for host process-to-Service traffic (same node, bypasses PREROUTING)
- **POSTROUTING**: SNAT/masquerade (rewrite source address for traffic leaving the node, so return traffic routes back correctly)

### `mangle` table — packet modification

Used to mark packets or modify headers (TTL, DSCP/TOS bits).

Default chains: `PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`.

K8s usage:
- Calico marks packets with firewall marks before filter decisions
- kube-proxy uses marks to track which packets have already been processed

### `raw` table — pre-conntrack bypass

Processed before conntrack entries are created. Used to exclude specific flows from conntrack tracking (`NOTRACK` target).

Default chains: `PREROUTING`, `OUTPUT`.

K8s usage: High-throughput scenarios where conntrack overhead is unacceptable (e.g., certain data path optimizations).

### `security` table — SELinux marks

Applied after `filter`. Rarely used in standard K8s deployments. Relevant only when SELinux mandatory access control is integrated with networking.

---

## The 5 chains and packet traversal order

The order of table/chain evaluation for a packet depends on whether the packet is:
- **Arriving from outside** (e.g., external client → node → pod)
- **Leaving the node** (e.g., pod → internet)
- **Forwarded** (e.g., pod → Service → different pod)

### Incoming packet destined for a local process (INPUT path):

```
raw.PREROUTING → mangle.PREROUTING → nat.PREROUTING
    → routing decision (is this for us?) →
mangle.INPUT → filter.INPUT → local process
```

### Incoming packet being forwarded (FORWARD path):

```
raw.PREROUTING → mangle.PREROUTING → nat.PREROUTING
    → routing decision (forward this) →
mangle.FORWARD → filter.FORWARD →
mangle.POSTROUTING → nat.POSTROUTING → out
```

### Outgoing packet from local process (OUTPUT path):

```
local process →
raw.OUTPUT → mangle.OUTPUT → nat.OUTPUT → filter.OUTPUT
    → routing decision →
mangle.POSTROUTING → nat.POSTROUTING → out
```

**Key insight for K8s**: `nat.PREROUTING` runs before routing. So when a pod sends a packet to `10.96.0.1:80` (a Service ClusterIP), `nat.PREROUTING` rewrites the destination to `10.244.0.5:8080` (a pod IP). The routing decision then routes it to the pod — it never "sees" the Service IP. But `nat.OUTPUT` must also handle the case where the source is a local process on the node (host networking), because local-originated packets skip PREROUTING.

---

## kube-proxy iptables rules — the full structure

For a Service `myservice` with ClusterIP `10.96.0.50:80` and three endpoints at `10.244.x.x:8080`:

### nat table, PREROUTING chain

```
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

All traffic hits `KUBE-SERVICES` first.

### KUBE-SERVICES chain

```
-A KUBE-SERVICES -d 10.96.0.50/32 -p tcp -m tcp --dport 80 \
  -m comment --comment "default/myservice:http cluster IP" \
  -j KUBE-SVC-ABCDEF123456
```

Matches packets to the ClusterIP. Jumps to the Service's chain.

### KUBE-SVC-ABCDEF123456 chain (load balancing)

```
# 33% chance: go to pod 1
-A KUBE-SVC-ABCDEF123456 -m statistic --mode random --probability 0.33333333349 \
  -j KUBE-SEP-POD1HASH

# 50% of remaining (= 33% overall): go to pod 2
-A KUBE-SVC-ABCDEF123456 -m statistic --mode random --probability 0.50000000000 \
  -j KUBE-SEP-POD2HASH

# Fall through (33% overall): go to pod 3
-A KUBE-SVC-ABCDEF123456 -j KUBE-SEP-POD3HASH
```

The probability math is cumulative: each rule sees only the fraction of packets that fell through previous rules. The probabilities chain to 1/3 each total.

### KUBE-SEP-POD1HASH chain (endpoint DNAT)

```
# Mark the source if this pod is sending to itself (hairpin)
-A KUBE-SEP-POD1HASH -s 10.244.0.5/32 -j KUBE-MARK-MASQ

# DNAT the destination to the pod's actual address
-A KUBE-SEP-POD1HASH -p tcp -m tcp -j DNAT --to-destination 10.244.0.5:8080
```

The `KUBE-MARK-MASQ` handles the "pod contacting its own Service IP" hairpin case — the packet needs SNAT on the way out so the reply comes back correctly.

### POSTROUTING masquerade

```
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A KUBE-POSTROUTING -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

Packets marked with `0x4000` (the masquerade mark set by `KUBE-MARK-MASQ`) get their source IP rewritten to the node's IP as they leave. This ensures return traffic routes back to the node, where conntrack can restore the original addressing.

### Scale problem

Each Service creates 1 `KUBE-SVC-*` chain, N `KUBE-SEP-*` chains (one per endpoint), and ~3-5 rules in each. At 1,000 Services with 3 endpoints each, that's ~5,000 chains and ~20,000 rules. At 10,000 Services: ~50,000 rules. Rule evaluation is O(n) per packet. This is the scalability wall.

---

## conntrack — the connection tracking that makes DNAT work

conntrack maintains a table of all active connections (TCP) and pseudo-connections (UDP). For each DNAT'd flow, it records:

```
original:  src=10.244.0.3:45678  dst=10.96.0.50:80
reply:     src=10.244.0.5:8080   dst=10.244.0.3:45678
```

When the reply packet arrives with `src=10.244.0.5:8080`, conntrack matches it against the "reply" direction and rewrites the source to `10.96.0.50:80` automatically, so the original caller sees the reply from the Service IP.

### conntrack state machine

TCP connections go through states:

```
NEW → ESTABLISHED → (FIN_WAIT/CLOSE_WAIT) → CLOSE
```

`-m conntrack --ctstate NEW,ESTABLISHED,RELATED` is a common filter — allow new connections and established ones, but not untracked packets.

### The DNS/UDP conntrack gotcha

UDP has no connection semantics. conntrack tracks UDP by flow (src IP/port, dst IP/port, protocol). When a pod does DNS:
1. Pod sends UDP to `10.96.0.10:53` (kube-dns ClusterIP). conntrack creates a NEW entry.
2. DNAT rewrites to `10.244.0.2:53` (CoreDNS pod IP).
3. CoreDNS replies. conntrack matches the reply, rewrites source to `10.96.0.10:53`.

The problem: pod's source port is chosen randomly. If two simultaneous DNS queries from the same pod share the same source port (UDP port reuse under load), conntrack gets confused — the second query might be associated with the wrong DNAT entry. This is the famous DNS UDP race condition in K8s that shows up as intermittent `5s DNS timeout` errors in high-throughput services. The fix is often to run with more CoreDNS replicas, enable TCP-fallback in the application's DNS client, or use NodeLocal DNSCache.

```bash
# Check conntrack table on a node
conntrack -L --proto udp | grep ":53"
# conntrack -S for statistics (look for "insert_failed" — that's the race)
conntrack -S
```

---

## nftables — the modern replacement

nftables was merged in Linux 3.13 (2014) but became the practical replacement for iptables around Linux 5.x. The iptables command on modern systems often runs as `iptables-nft` — a compatibility shim translating iptables commands to nftables internally.

### Structural differences

**iptables**: all rules are in a flat linked list per chain. Evaluation is linear O(n). Adding or removing a rule requires locking the entire ruleset, modifying it, and flushing kernel state.

**nftables**: uses kernel data structures optimized for lookup — hash tables for exact IP matches, interval trees for ranges. Rule sets are committed as atomic transactions. No partial-application window.

```nft
# nftables equivalent of a DNAT rule:
table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        ip daddr 10.96.0.50 tcp dport 80 dnat to numgen random mod 3 map {
            0 : 10.244.0.5:8080,
            1 : 10.244.0.6:8080,
            2 : 10.244.0.7:8080
        }
    }
}
```

The `numgen random mod 3` is built-in load balancing — no probability chain needed.

### K8s migration to nftables

Kubernetes 1.29 introduced kube-proxy in **nftables mode** (beta). K8s 1.31+ it's generally available on supported distros. The nftables mode:
- Uses `nft` instead of `iptables`
- Rewrites the Service proxy as a single atomic nftables rule set
- Scalable: O(1) or O(log n) lookup for Service IP matching (sets/maps instead of chain walks)

To check: `kubectl -n kube-system get cm kube-proxy-config -o yaml | grep mode`

### Checking what's in use

```bash
# On a node:
iptables -V
# iptables v1.8.7 (nf_tables)  ← using nftables backend
# iptables v1.8.4 (legacy)     ← using traditional iptables

nft list ruleset | grep -c "chain"  # count nftables chains
iptables-save | wc -l              # count iptables rules
```

---

## How Cilium and Calico change the picture

### Cilium — bypass iptables entirely with eBPF

Cilium installs eBPF programs at the XDP (eXpress Data Path) and TC (Traffic Control) hook points — earlier in the packet path than iptables. When Cilium is the CNI:
- kube-proxy is disabled (`--skip-phases=proxy` at cluster creation)
- Cilium manages Service proxying via eBPF maps — O(1) lookup
- Network Policies are enforced at the eBPF layer — no iptables rules for policy
- Conntrack is handled by Cilium's own eBPF conntrack table (faster, per-CPU)

This is the architecture of most modern EKS clusters using Cilium: faster packet processing, less CPU for kube-proxy, no iptables scaling wall.

### Calico

Calico uses iptables by default for Network Policy enforcement (writes rules in the `filter` table) while leaving kube-proxy to handle Service proxying. Calico's eBPF mode (available since Calico 3.13) replaces both iptables and kube-proxy with eBPF programs, similar to Cilium.

```bash
# See Calico's iptables rules (iptables mode)
iptables -L | grep cali
# cali-INPUT, cali-FORWARD, cali-OUTPUT — Calico's chains
```

---

## The interview answer in 60 seconds

> "kube-proxy watches the Kubernetes API for Service and Endpoints changes. When a Service is created, kube-proxy writes iptables rules in the `nat` table's `PREROUTING` chain (and `OUTPUT` for host-originated traffic). The rules form a chain: `KUBE-SERVICES → KUBE-SVC-<hash>` (per Service) `→ KUBE-SEP-<hash>` (per endpoint). Each packet destined for the Service's ClusterIP gets DNAT'd in PREROUTING to one of the endpoint pod IPs, picked randomly using `--mode random --probability` math.
>
> conntrack records the original-to-rewritten flow. The reply packet from the pod has its source rewritten back to the Service IP by conntrack before the caller receives it — so DNAT is transparent end-to-end.
>
> The scale problem: each Service creates a chain, each endpoint adds rules, and evaluation is O(n) per packet. At 10,000 Services that's ~50,000 rules per packet — real latency. That's why K8s 1.29+ has a nftables mode (O(1) lookup via maps), and Cilium replaces kube-proxy entirely with eBPF for O(1) Service lookup without touching iptables at all."

---

## Self-test drills

### 1. Walk me through how kube-proxy's iptables mode routes a Service IP to a pod IP.

**Reference answer:**
- kube-proxy watches API server → writes nat.PREROUTING rules matching ClusterIP:port → jumps to KUBE-SVC chain → probabilistic jump to one of N KUBE-SEP chains → DNAT to pod IP:port
- conntrack records the rewrite; reply packets get source rewritten back automatically
- POSTROUTING MASQUERADE handles hairpin (pod-to-own-Service) and cross-node traffic

### 2. What is conntrack and why does K8s need it?

**Reference answer:**
- conntrack = kernel connection tracking table; records src/dst/proto/port for each active flow
- Without conntrack, DNAT would be one-way: request gets rewritten to pod IP, but reply comes from pod IP, confuses the caller
- conntrack's reply direction automatically reverses the DNAT rewrite — caller sees the Service IP in replies
- The DNS UDP race condition: conntrack can confuse simultaneous UDP DNS queries from the same pod (same source port) — manifests as 5s DNS timeouts at high connection rate

### 3. What are the key differences between iptables and nftables, and why is K8s migrating?

**Reference answer:**
- iptables: linear list evaluation O(n), non-atomic updates (rules applied one-by-one, inconsistent window), separate tools per family
- nftables: hash/tree lookups O(1) or O(log n), atomic transactions (whole ruleset committed at once), single `nft` tool
- K8s 1.29+ beta kube-proxy nftables mode uses sets/maps for O(1) Service IP matching instead of chain walks
- Cilium goes further: eBPF at XDP/TC layer, no iptables at all, O(1) via eBPF maps, replaces kube-proxy entirely

### 4. How does Cilium change the iptables story for K8s?

**Reference answer:**
- Cilium is a CNI plugin that installs eBPF programs at XDP (very early packet path) and TC hooks
- kube-proxy is disabled; Cilium handles Service proxying via eBPF maps — O(1) lookup
- Network Policies enforced at eBPF layer — no iptables filter rules for policy
- Cilium's own eBPF conntrack table replaces kernel conntrack for pod traffic — faster, per-CPU
- On EKS with Cilium: no `iptables -L | grep KUBE` rules — completely different data path
- Tradeoff: eBPF programs are kernel-version-dependent; requires Linux 4.9+ (ideally 5.10+)

---

## Further reading / watching

- **iptables man page**: `man iptables` and `man iptables-extensions`
- **nftables wiki**: [wiki.nftables.org](https://wiki.nftables.org) — the primary reference
- **K8s kube-proxy nftables mode**: [kubernetes.io/docs/reference/networking/virtual-ips/](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- **Cilium — eBPF and XDP overview**: [docs.cilium.io/en/stable/concepts/ebpf/](https://docs.cilium.io/en/stable/concepts/ebpf/)
- **Deep dive: kube-proxy mode comparison** — search "kube-proxy iptables vs nftables vs ipvs benchmark"

---

## The 4 dimensions (senior framing)

- **Tech**: 5 tables × 5 chains; PREROUTING nat for ClusterIP DNAT; conntrack for bidirectional address rewriting; KUBE-SERVICES → KUBE-SVC → KUBE-SEP chain structure; iptables O(n) at scale → nftables O(log n) atomic → Cilium eBPF O(1).
- **People**: when a developer reports "I can reach Service X from some pods but not others," the debugging path goes through iptables. Train your team to use `iptables -L -n -v | grep KUBE-SVC` to find which chains exist and `conntrack -L` to see active flows. Demystify it: it's not magic, it's DNAT rules.
- **CI/CD**: if you're running network policy tests in CI, verify they work against the same CNI + kube-proxy mode as production. A NetworkPolicy that blocks in Calico iptables mode may behave differently in Cilium eBPF mode — the semantics are slightly different for edge cases (egress from host network namespace, LoadBalancer IP ranges).
- **Operations**: `iptables -L -n --line-numbers | wc -l` tells you how many rules are in the filter table. `conntrack -S` shows conntrack statistics — watch `insert_failed` (hash collisions under high load, often correlates with DNS timeouts). On large clusters, consider Cilium or kube-proxy ipvs mode to escape the iptables O(n) wall before it becomes a latency problem.
