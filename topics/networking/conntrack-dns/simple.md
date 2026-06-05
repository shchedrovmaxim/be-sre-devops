# conntrack + UDP DNS — the simple version (the coat check analogy)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

One idea:

> **The conntrack table is a coat-check service. Every time your app sends a UDP packet, the kernel writes a ticket: "this person sent a coat, here's the claim number." The problem: the coat check room has a maximum capacity. When it fills up, new coats are refused — the packet is dropped. For UDP DNS, there's also a race condition where two attendants write the same claim number at the same time and one ticket gets torn up.**

Those two failure modes — table full and race condition — are the root cause of every "intermittent DNS NXDOMAIN under load" incident.

---

## The coat check analogy

| Coat check world | K8s / conntrack world |
|---|---|
| You hand over your coat, get a claim ticket | Pod sends a UDP packet, kernel creates a conntrack entry (5-tuple) |
| Ticket says: "client A gave coat at 3:02pm, from table 7" | Entry: src IP, src port, dst IP, dst port, protocol, state |
| Ticket expires after 30 seconds if you don't claim your coat | UDP conntrack timeout: 30s default (no "FIN" for UDP, so timeout-based) |
| Coat check room is full: "sorry, no more tickets" | `nf_conntrack_max` reached: new packets DROP (no conntrack entry = no route) |
| Two attendants write ticket #4521 simultaneously | Race condition: two goroutines inserting same 5-tuple → `insert_failed` |
| One ticket gets torn up, customer gets a NXDOMAIN from the waiter | Second insert fails, UDP packet dropped → DNS timeout |
| Bigger room | Raise `nf_conntrack_max` |
| One attendant per section (NodeLocal DNS) | NodeLocal Cache: queries bypass conntrack or use TCP |

---

## The whole idea in 2-3 sentences

The Linux kernel tracks every connection (including UDP "connections") in the conntrack table — a fixed-size hash table bounded by `nf_conntrack_max`. Under high DNS load in a Kubernetes cluster, two pods on the same node can simultaneously send DNS queries that hash to the same conntrack slot, causing a race condition where one entry's insert fails and the packet is silently dropped. The pod then waits for its DNS timeout (default 5 seconds), retries, and the retry might succeed — manifesting as intermittent 5% NXDOMAIN failures under load.

---

## The 2 concepts that confuse people

### 1. Why does UDP need conntrack at all?

UDP is stateless — there's no handshake, no FIN. But the kernel still tracks UDP "connections" so that DNAT (used by kube-proxy for Service routing) can work: when a response comes back to the pod, the kernel needs to know which pod sent the original packet and un-NAT the response.

Without a conntrack entry for the UDP "connection," the response packet arrives at the node, the kernel has no state, and it can't route it back to the right pod. So even though UDP doesn't have connections, conntrack creates one with a timeout (30s default).

### 2. The `insert_failed` race condition

DNS queries under load are frequent and bursty. On a busy node:
- Pod A and Pod B simultaneously send DNS queries
- Both queries hash to the same conntrack bucket
- The kernel does a bucket lookup → empty → both threads decide to insert
- Thread 1 inserts, Thread 2 tries to insert → sees bucket already occupied → `insert_failed`
- Thread 2's packet is dropped at the network stack before it even reaches kube-proxy

This is not a bug the app can handle — the packet never leaves the node's network stack. It's invisible to application logs. The only evidence is the kernel counter `nf_conntrack_stat: insert_failed`.

```bash
# Check on any node:
cat /proc/net/stat/nf_conntrack | head -2
# The 8th column is insert_failed — non-zero means you're hitting this
```

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What is conntrack? | Kernel table tracking "connections" (including UDP) for NAT and stateful firewalling |
| Why does DNS use conntrack? | DNAT (kube-proxy) needs to track which pod sent which UDP query to un-NAT the response |
| What is `nf_conntrack_max`? | The maximum size of the conntrack table; default often 65536 or 131072; too low → drops |
| What is the race condition? | Two simultaneous UDP DNS packets hashing to the same slot; one insert fails, packet dropped |
| What's the kernel counter? | `insert_failed` in `/proc/net/stat/nf_conntrack` |
| What's the standard fix? | NodeLocal DNS Cache — most queries bypass conntrack; cache misses use TCP |
| What's the emergency fix? | Raise `nf_conntrack_max` (buys time); tune UDP timeout down (`nf_conntrack_udp_timeout`) |
| Can you monitor this in Prometheus? | Yes — `node_nf_conntrack_entries` vs `node_nf_conntrack_entries_limit`; also `insert_failed` needs a custom metric or node exporter textfile |

---

## Self-test (the killer interview question)

Out loud:

> **"5% of pods get intermittent DNS NXDOMAIN under load. Walk through how you'd diagnose this."**

**Reference answer (intuitive version):**

"I'd start by confirming the symptom: is it truly DNS NXDOMAIN or is it connection refused? Those are different things. Run `nslookup` from an affected pod with verbose output — confirm the response is `NXDOMAIN` and not a timeout.

Next, look for correlation: does it happen under load and not at rest? Is it only on certain nodes? DNS failures that are load-correlated and node-correlated point immediately to conntrack.

Then I'd check two counters on the affected nodes. First: `cat /proc/net/stat/nf_conntrack` — the `insert_failed` column. If it's non-zero and growing, that's the race condition smoking gun. Second: `cat /proc/sys/net/netfilter/nf_conntrack_count` vs `nf_conntrack_max` — if count is near max, you have table exhaustion.

The fix is NodeLocal DNS Cache. It puts a caching DNS daemon on every node — most queries hit the local cache (no conntrack), and cache misses go to CoreDNS over TCP (TCP avoids the UDP race condition). This is the permanent fix. As an emergency measure while deploying NodeLocal, raise `nf_conntrack_max` to buy headroom.

The senior nuance: don't just raise `nf_conntrack_max` and call it done. A higher limit delays the problem but doesn't fix it. NodeLocal is the actual fix."

---

## Further reading / watching

- **Weave Works blog — "Racy conntrack and DNS lookup timeouts"** (2018): the landmark post; search "weave works racy conntrack dns" — this explained the race condition for the first time to the K8s community and is still the definitive reference
- **Cilium blog — "Kernel packet drop tracing"**: `cilium.io/blog` — explains how to trace packet drops with eBPF tools (`bpftrace`, Hubble)
- **Linux kernel conntrack docs**: search "Linux netfilter conntrack" — the netfilter project docs explain the table structure and tunables
- **Kubernetes docs — NodeLocal DNS Cache**: `kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/` — the fix, with installation instructions

---

## Next: the deep-dive

When the coat-check analogy and the `insert_failed` counter feel obvious, jump to [`conntrack-dns.md`](./deep-dive.md). The deep-dive covers:

- How the conntrack table is structured (hash table, buckets, chaining)
- The full UDP state machine (NEW → UNREPLIED → ASSURED → timeout)
- The exact DNAT + UDP race condition at the kernel code level
- Tuning `nf_conntrack_max` and `nf_conntrack_buckets` — the math
- The `single-request` and `single-request-reopen` resolver options
- NodeLocal as the canonical fix (and why it works)
- Host-network workaround for critical workloads
- The 4-dimensions framing (Tech / People / CI/CD / Operations)
