# CoreDNS + NodeLocal DNS Cache — the simple version (the library reference desk analogy)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

One idea:

> **CoreDNS is the library's main reference desk. Every student in the school has to walk to the main desk to ask questions. NodeLocal DNS Cache is a mini reference desk on each floor — students ask their floor's desk first, and only bother the main desk for questions the floor desk hasn't heard before.**

The "questions the floor desk has heard" are cached DNS responses. Under heavy load, the main desk gets overwhelmed. NodeLocal shifts 80-90% of the queries to the floor desks, which never get overwhelmed because they serve one floor each (one node).

---

## The library reference desk analogy

| Library world | K8s world |
|---|---|
| Main reference desk (1 desk, centrally located) | CoreDNS Deployment (2-3 pods, running as a ClusterIP Service) |
| Every student walks to the main desk | Every pod sends DNS queries to CoreDNS via ClusterIP |
| Under heavy classes: long lines, desk staff exhausted | Under heavy load: CoreDNS pods overwhelmed, DNS timeouts |
| Mini desk on each floor | NodeLocal DaemonSet pod on each node |
| Student asks floor desk first | Pod asks the node-local IP (169.254.20.10) first |
| Floor desk has the answer → instant, no walk to main desk | Cache hit → response in microseconds, no network hop |
| New question → floor desk asks main desk once, caches the answer | Cache miss → NodeLocal asks CoreDNS, caches the result |
| Floor desk uses a private phone line to main desk (not the public queue) | NodeLocal uses TCP to CoreDNS — bypasses conntrack UDP race condition |

The "private phone line" detail is the senior gotcha. Without NodeLocal, all pods send UDP DNS queries through the kernel's conntrack table. Under high load, a race condition in conntrack causes ~5% of queries to fail. NodeLocal fixes this by using TCP (which doesn't hit the conntrack race condition).

---

## The whole idea in 2-3 sentences

CoreDNS is the cluster's central DNS server — a Deployment of 2-3 pods behind a ClusterIP. Every pod queries it for every DNS lookup. NodeLocal DNS Cache adds a DaemonSet that runs a DNS caching daemon on every node, listening on a link-local IP (`169.254.20.10`); pods on that node query this local cache first, which both reduces latency (no network hop, no ClusterIP) and eliminates the conntrack UDP race condition by having NodeLocal talk to CoreDNS over TCP.

---

## The 2 concepts that confuse people

### 1. The conntrack problem NodeLocal solves

When pods send UDP DNS queries, the packets go through the kernel's conntrack table (which tracks UDP "connections" by 5-tuple + timeout). Under high DNS load from many pods on the same node, two goroutines can simultaneously try to insert a new conntrack entry for the same 5-tuple — the second insert fails, and the DNS packet is dropped. The pod gets a timeout, retries, and the retry might succeed. This manifests as ~5% DNS failures under load.

NodeLocal eliminates this by:
- **Local cache hit**: the packet never leaves the node at all — no conntrack needed
- **Cache miss → CoreDNS**: NodeLocal sends this as **TCP** to CoreDNS, not UDP. TCP connections have different conntrack semantics that avoid the race condition

### 2. The link-local IP `169.254.20.10`

NodeLocal listens on `169.254.20.10` — this is a link-local address (like `169.254.x.x` in general) that is only reachable locally on the node. It's not a ClusterIP, not routable between nodes. The kubelet configures each pod's `/etc/resolv.conf` to point at this IP when NodeLocal is installed:

```
# /etc/resolv.conf in a pod with NodeLocal enabled:
nameserver 169.254.20.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Because the IP is node-local, queries to it never hit the ClusterIP load balancer and never touch the network fabric — they're handled directly by the DaemonSet pod on the same node.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What is CoreDNS? | Cluster DNS server — Deployment of 2-3 pods behind a ClusterIP |
| What is NodeLocal DNS Cache? | DaemonSet DNS cache on every node, listening on 169.254.20.10 |
| Why does NodeLocal help with reliability? | Bypasses conntrack for most queries; cache hits never leave the node |
| Why does NodeLocal help with latency? | Cache hits respond in microseconds; no network hop to CoreDNS ClusterIP |
| Why does NodeLocal use TCP to CoreDNS? | TCP avoids the conntrack UDP race condition for the upstream queries |
| What is `ndots:5`? | If a hostname has fewer than 5 dots, the resolver appends all search domains before trying the bare name — amplifies DNS query count dramatically |
| What does a Corefile look like? | A set of blocks like `cluster.local:53 { kubernetes ... cache ... }` — each block is a zone + plugin chain |
| Does NodeLocal replace CoreDNS? | No — it caches and forwards. CoreDNS still exists and handles cache misses |

---

## Self-test (the killer interview question)

Out loud:

> **"Why might adding NodeLocal DNS Cache dramatically improve DNS reliability and latency?"**

**Reference answer (intuitive version):**

"Two problems, two solutions.

The **reliability** problem is conntrack. Every UDP DNS query from a pod goes through the kernel's conntrack table. Under high load from many pods on the same node, there's a known race condition where two threads try to insert the same 5-tuple entry simultaneously — the second fails and the packet is dropped. This causes 5-10% intermittent DNS failures under load that manifest as timeouts or NXDOMAIN responses. NodeLocal fixes this by caching most queries locally (so they never hit conntrack) and sending cache misses to CoreDNS over TCP instead of UDP (TCP doesn't have the same race condition in conntrack).

The **latency** problem is distance and load. Without NodeLocal, every DNS query — even for `kubernetes.default.svc.cluster.local` that never changes — traverses the network to a CoreDNS pod, which might be on a different node. Under heavy load, CoreDNS pods become the bottleneck. With NodeLocal, cached responses are served from a daemon on the same node in sub-millisecond time, and CoreDNS only sees cache misses — a fraction of the original query volume.

The combined effect: DNS becomes fast, predictable, and stops being the cause of intermittent failures under load. This is one of the highest-value, lowest-risk improvements you can make to a production K8s cluster."

---

## Further reading / watching

- **Kubernetes docs — NodeLocal DNS Cache**: `kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/` — installation instructions and architecture diagram
- **CoreDNS docs — Corefile reference**: `coredns.io/docs/corefile/` — plugin reference; understand the `kubernetes`, `cache`, `forward`, and `errors` plugins
- **Kubernetes GitHub — NodeLocal DNS Cache design doc**: search "NodeLocal DNS Cache KEP kubernetes" — the original KEP explains the conntrack race condition in detail
- **Weave Works blog — "Racy conntrack and DNS lookup timeouts"**: the landmark post diagnosing the conntrack race condition; search "weave works racy conntrack dns" — this is the post that popularized the root cause

---

## Next: the deep-dive

When the library floor-desk analogy and the conntrack→TCP detail feel obvious, jump to [`coredns-nodelocal.md`](./deep-dive.md). The deep-dive covers:

- Corefile structure with a full annotated example
- The kubernetes plugin and how CoreDNS answers cluster DNS queries
- NodeLocal architecture in detail (DaemonSet, the `node-local-dns` binary, iptables rules it adds)
- The conntrack race condition with the exact kernel code path
- `ndots:5` query amplification and how to tune it
- Debugging DNS in production (the commands you actually run)
- The 4-dimensions framing (Tech / People / CI/CD / Operations)
