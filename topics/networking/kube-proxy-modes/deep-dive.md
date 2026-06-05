# kube-proxy modes — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why might iptables mode become a problem at 5,000 Services?"** — naming the O(N) rule walk, the global iptables lock contention under pod churn, the IPVS O(1) alternative, and Cilium eBPF as the kube-proxy-free path.

> Start with the [simple version](./simple.md) if you haven't. The switchboard operator analogy is the spine.

---

## The senior framing — kube-proxy is load-bearing kernel glue

kube-proxy is one of those "it just works until it doesn't" components. Most engineers have never thought about it because at 50 Services it's invisible. At 5,000 Services, in a high-churn cluster (spot instances, CI runners, batch jobs), it becomes one of the top sources of "mysterious Service connectivity problems."

The senior insight: **kube-proxy's mode is a cluster-level setting that's hard to change after the fact.** Pick the right mode at cluster creation, or be prepared for a disruptive migration.

---

## What kube-proxy actually does

kube-proxy runs as a DaemonSet on every node. It watches the API server for:
- `Service` objects (ClusterIP, port, targetPort, selector)
- `Endpoints` / `EndpointSlices` (the actual pod IPs behind each Service)

When these change, kube-proxy programs the node's kernel to redirect traffic from `ClusterIP:port` to one of the backing pod IPs.

The packet path:
```
App calls ClusterIP:8080
→ kernel intercepts at netfilter (PREROUTING hook)
→ kube-proxy's rules translate ClusterIP → PodIP (DNAT)
→ packet leaves node to target pod (or stays local)
→ response comes back, SNAT back to ClusterIP
→ app sees ClusterIP:8080 as the source (transparent)
```

This is what "virtual IP" means in K8s — the ClusterIP never has a real network interface. It's a phantom IP that kernel rules translate.

---

## iptables mode — the default, and where it breaks

### The chain structure

For every Service, kube-proxy creates an iptables chain in the NAT table:

```
PREROUTING → KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash-1> (endpoint 1)
                                               → KUBE-SEP-<hash-2> (endpoint 2)
                                               → KUBE-SEP-<hash-3> (endpoint 3)
```

For a Service with 3 endpoints, the chain looks like:
```
-A KUBE-SVC-XXXXXXXXXXX -m statistic --mode random --probability 0.33 -j KUBE-SEP-AAAA
-A KUBE-SVC-XXXXXXXXXXX -m statistic --mode random --probability 0.50 -j KUBE-SEP-BBBB
-A KUBE-SVC-XXXXXXXXXXX -j KUBE-SEP-CCCC

-A KUBE-SEP-AAAA -j DNAT --to-destination 10.0.1.10:8080
-A KUBE-SEP-BBBB -j DNAT --to-destination 10.0.1.11:8080
-A KUBE-SEP-CCCC -j DNAT --to-destination 10.0.1.12:8080
```

The probability-based random selection is how iptables achieves "random" load balancing — each rule has a `--probability` that makes the math work out to even distribution.

### Rule count math

| Services | Endpoints/svc | Approximate total rules |
|---|---|---|
| 100 | 5 | ~750 |
| 500 | 5 | ~3,750 |
| 1,000 | 5 | ~7,500 |
| 5,000 | 5 | ~37,500 |
| 10,000 | 5 | ~75,000 |

Every packet to any ClusterIP starts at `KUBE-SERVICES` and walks down the chain until it finds a match. Worst case: the Service is last in the list → the kernel evaluated all preceding rules.

### The global iptables lock problem

When kube-proxy adds or removes rules (e.g., a pod dies and its endpoint is removed), it:
1. Dumps the current iptables ruleset
2. Modifies it
3. Atomically replaces the full ruleset with `iptables-restore`

Step 3 acquires `ip_tables_mutex`, a kernel-wide lock. During the lock:
- **No packet can match any iptables rule** (PREROUTING chains are blocked)
- On large rulesets, the `iptables-restore` atomic swap takes longer (proportional to ruleset size)
- Under high pod churn (hundreds of endpoint changes/sec), lock acquisitions can queue up

The result: brief packet drops during high-churn periods. Services become intermittently unreachable. This is the bug that looks like "our app is flaky" but is actually "the iptables lock is contended."

Real numbers from the community (treat as directional — exact numbers vary by kernel version and workload):
- At 10,000 rules: `iptables-restore` can take 100–200ms
- Under 10 endpoint changes/sec: barely noticeable
- Under 100 endpoint changes/sec (a rolling deployment on a large cluster): lock contention becomes observable in connection drop rates

### nftables mode (Kubernetes 1.31+)

In Kubernetes 1.31, kube-proxy gained an `nftables` mode. nftables is the modern replacement for iptables in the kernel — same netfilter framework, but with sets/maps that give O(1) lookup (similar to IPVS) and atomic rule updates without a global lock.

As of 2026, nftables mode is stable but IPVS is still more commonly deployed in production. Worth knowing it exists.

---

## IPVS mode — the O(1) path

### How it works

IPVS (IP Virtual Server) is a kernel module (`ip_vs`) originally designed for Linux load balancers. It maintains its own hash table of Virtual Services (ClusterIPs) mapped to Real Servers (pod endpoints).

```
Virtual Service: 10.96.0.1:8080 (ClusterIP)
  Real Server: 10.0.1.10:8080 (pod 1)  weight=1
  Real Server: 10.0.1.11:8080 (pod 2)  weight=1
  Real Server: 10.0.1.12:8080 (pod 3)  weight=1
```

When a packet arrives destined for `10.96.0.1:8080`, the kernel does a hash lookup → finds the virtual service → picks a real server per the load balancing algorithm → DNATs the packet. One lookup, O(1), regardless of the number of Services.

### IPVS kernel modules required

```bash
# Must be loaded on every node
modprobe ip_vs
modprobe ip_vs_rr    # round-robin
modprobe ip_vs_wrr   # weighted round-robin
modprobe ip_vs_sh    # source hash (session affinity)
modprobe nf_conntrack  # still needed for connection tracking
```

Managed K8s (EKS, GKE, AKS) handles this. Self-managed clusters need it in node configuration.

### Load balancing algorithms in IPVS

| Algorithm | Flag | When to use |
|---|---|---|
| Round Robin | `rr` | Default, stateless services |
| Weighted RR | `wrr` | Different pod sizes/capacities |
| Least Connections | `lc` | Long-lived connections (databases, WebSocket) |
| Source Hash | `sh` | Session stickiness by client IP |
| Random | `random` | Simple distribution |

Compare to iptables: iptables only does probabilistic random selection. IPVS gives you richer, kernel-native algorithms.

### Enabling IPVS mode

```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      algorithm: rr
    iptables:
      masqueradeAll: false
```

IPVS mode still uses a small number of iptables rules (for masquerading and NodePort handling) — it doesn't eliminate iptables entirely, but reduces the NAT table from 75,000 rules to ~50 rules.

---

## Cilium eBPF — the kube-proxy-free path

### How it differs

With `kubeProxyReplacement: strict`, Cilium completely removes kube-proxy from the cluster. Service routing is handled by the `cilium-agent` eBPF programs.

The eBPF Service map:
```
# Cilium stores services as BPF maps (hash tables in kernel memory)
Service map:   ClusterIP:port → ServiceID
Backend map:   ServiceID → [PodIP:port, PodIP:port, ...]
```

At packet arrival:
```
Socket-level hook: before packet even enters the IP stack
  → lookup ClusterIP in eBPF map → get backend → DNAT at socket layer
  → packet goes directly to pod IP (no conntrack NAT needed for same-node)
```

For cross-node traffic, Cilium still uses DNAT but via eBPF instead of iptables. The conntrack table is only used for the cross-node path; same-node pod-to-pod goes through sockmap without conntrack.

### The upgrade path — what breaks

Migrating from kube-proxy iptables → Cilium eBPF:

1. Deploy Cilium with `kubeProxyReplacement: partial` first — Cilium handles some routing, kube-proxy handles the rest
2. Validate Service connectivity thoroughly
3. Switch to `kubeProxyReplacement: strict` and remove kube-proxy DaemonSet
4. Flush residual iptables rules (`iptables-legacy --flush`)

**What can break:**
- Any workload that inspects raw iptables rules (some monitoring agents do this) may break or report false positives
- NodePort handling changes behavior slightly (Cilium uses BPF NodePort vs iptables DNAT)
- Services with `externalTrafficPolicy: Local` need verification — Cilium handles this differently

The migration is a node-level rollout: drain nodes, update, validate, uncordon. Not a rolling restart — full drain.

---

## Performance numbers (directional, from published sources)

From Cilium's published benchmarks (2023, their test environment — treat as directional):

| Cluster size | kube-proxy iptables CPU | IPVS CPU | Cilium eBPF CPU |
|---|---|---|---|
| 100 services | negligible | negligible | negligible |
| 1,000 services | 0.5–1% per node | 0.1% | 0.1% |
| 10,000 services | 2–5% per node sustained | 0.2–0.5% | 0.2% |

From the Kubernetes IPVS introduction blog post (2018, Google):
- iptables rule sync time at 20,000 rules: ~11 seconds per update
- IPVS sync time at 20,000 services: ~0.1 seconds

The rule-sync latency is the critical number: slow rule updates = endpoint flapping = brief Service unreachability during deployments.

---

## The interview answer in 60 seconds

> "kube-proxy in iptables mode writes a chain of NAT rules in the kernel. For every Service, there's a chain; for every endpoint in that Service, there's a rule in the chain. When a packet arrives at a ClusterIP, the kernel walks the chain linearly until it finds a match — O(N) where N is the total number of rules. At 5,000 Services with 5 endpoints each, that's ~37,500 rules to evaluate per packet.
>
> The second problem is the global iptables lock. When kube-proxy updates rules — any pod restart removes and re-adds endpoints — it takes the kernel-wide `ip_tables_mutex`. At high pod churn (rolling deployments, spot instance cycling), lock contention under a large ruleset causes packet drops and rule-sync delays of seconds.
>
> The fix has two levels. IPVS mode is the immediate one — switch kube-proxy's mode to `ipvs`, install the `ip_vs_*` kernel modules, and you get O(1) hash-table lookup with proper load balancing algorithms (round-robin, least-connections, weighted). Rule updates don't block packet processing. For a new cluster where I choose the CNI, Cilium eBPF removes kube-proxy entirely — Service routing happens in eBPF maps loaded into the kernel by `cilium-agent`, same O(1) lookups, with the added benefit that same-node traffic bypasses conntrack."

---

## Self-test drills

### 1. Why might iptables mode become a problem at 5,000 Services?

**Reference answer:** O(N) linear rule walk per packet (~37,500 rules at 5k Services × 5 endpoints). Global `ip_tables_mutex` lock contention during endpoint updates causing packet drops under high pod churn. Rule sync time grows proportionally to ruleset size — at 20k rules, can take 10+ seconds per update. IPVS mode (O(1) hash) or Cilium eBPF are the solutions.

### 2. What kernel modules are required for IPVS mode and why?

**Reference answer:** `ip_vs` (core virtual server), `ip_vs_rr` (round-robin), `ip_vs_wrr` (weighted RR), `ip_vs_sh` (source hash for session affinity), and `nf_conntrack` (still needed for connection tracking on cross-node traffic). Without these loaded on every node, kube-proxy will fail to start in IPVS mode or silently fall back to iptables.

### 3. What breaks when you migrate from kube-proxy to Cilium eBPF?

**Reference answer:** (1) Any tool that reads raw iptables rules (some monitoring agents, custom scripts) may report errors or false positives — the KUBE-SVC/KUBE-SEP chains are gone. (2) NodePort behavior changes subtly — Cilium's BPF NodePort implementation differs from iptables DNAT in edge cases (especially with `externalTrafficPolicy: Local`). (3) The migration requires node draining (full drain, not rolling restart), so it's a planned maintenance window. Mitigate by using `kubeProxyReplacement: partial` first, validating, then switching to `strict`.

### 4. How does IPVS mode still use iptables?

**Reference answer:** IPVS mode replaces the Service-routing NAT rules with kernel hash tables, but it still uses a small number of iptables rules for: (1) masquerading outbound traffic (SNAT) for NodePort and external traffic, (2) handling some ICMP/health-check cases, (3) interoperability with existing netfilter rules. The total iptables rule count drops from tens of thousands to ~50 rules. The lock contention and O(N) lookup problems disappear because there are almost no packet-matching rules left.

---

## Further reading / watching

- **Kubernetes blog — "IPVS-Based In-Cluster Load Balancing Deep Dive"** (2018): search on `kubernetes.io/blog` — the authoritative technical intro to IPVS mode from when it launched
- **Kubernetes docs — Virtual IPs and Service Proxies**: `kubernetes.io/docs/reference/networking/virtual-ips/` — covers all modes including the new nftables mode
- **Cilium docs — kube-proxy replacement**: `docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/` — step-by-step migration guide
- **Anurag Gupta / KubeCon — "Life of a Packet"**: search on YouTube — traces a packet through iptables chains; locks in the mental model viscerally
- **Calico blog — "Why is kube-proxy such a pain in the neck?"**: search `projectcalico.org kube-proxy pain` — the Calico team's explanation of why they built their own Service routing

---

## The 4 dimensions (senior framing)

- **Tech**: iptables O(N) → IPVS O(1) → Cilium eBPF O(1) + kube-proxy removal; the global lock is the acute failure mode under high churn; nftables is the modern iptables replacement arriving in K8s 1.31+; IPVS still needs conntrack while Cilium bypasses it for local traffic.
- **People**: kube-proxy mode is often set at cluster creation and forgotten. Document which mode your cluster uses and why in a cluster ADR. Operators debugging "Service unreachable during deployment" rarely think of iptables lock contention — add it to your network troubleshooting runbook.
- **CI/CD**: test Service connectivity after any kube-proxy config change (including kube-proxy version upgrades, which sometimes change default mode behavior). Include `kubectl get svc` + connectivity smoke tests in cluster provisioning CI. For a Cilium migration, have a rollback plan (re-deploy kube-proxy DaemonSet, re-flush iptables) before starting.
- **Operations**: monitor kube-proxy's rule sync latency (`kubeproxy_sync_proxy_rules_duration_seconds` Prometheus metric). Alert if sync > 5 seconds — that means the lock is being held long enough to cause packet loss. For IPVS, monitor `ipvsadm -l --stats` for connection counters. For Cilium, monitor `cilium status` and BPF map saturation.
