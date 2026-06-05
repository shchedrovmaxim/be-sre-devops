# CoreDNS + NodeLocal DNS Cache — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why might adding NodeLocal DNS Cache dramatically improve DNS reliability and latency?"** — naming the conntrack UDP race condition, the TCP upstream path, the cache hit latency improvement, and the ndots:5 query amplification problem.

> Start with the [simple version](./coredns-nodelocal-simple.md) if you haven't. The library floor-desk analogy is the spine.

---

## The senior framing — DNS is the invisible load-bearing layer

Every service call starts with a DNS lookup. `redis.payments.svc.cluster.local`, `api.stripe.com`, `s3.amazonaws.com` — all DNS. If DNS is slow or flaky, **everything is slow or flaky**, and it looks like an app bug.

Senior SRE insight: the most common cause of intermittent errors in a K8s cluster that "looks like the app is flaky" is DNS. The conntrack race condition has been causing production incidents since 2018 (the Weave Works "racy conntrack" post). NodeLocal DNS Cache is the standard mitigation, and it's been available since K8s 1.18.

If your cluster doesn't have NodeLocal deployed, and you're running more than ~20 pods per node under any load, **you are likely hitting this issue intermittently.**

---

## CoreDNS — architecture and Corefile

### What CoreDNS does

CoreDNS is a DNS server running as a `Deployment` (typically 2-3 replicas) in the `kube-system` namespace, behind a ClusterIP Service at (usually) `10.96.0.10`. Every pod's `/etc/resolv.conf` points here.

When a pod queries `payments-api.payments.svc.cluster.local`:
1. Pod sends UDP DNS query to CoreDNS ClusterIP (`10.96.0.10:53`)
2. CoreDNS receives it, runs the query through its plugin chain for the matching zone
3. The `kubernetes` plugin queries the in-memory Service/Endpoint cache (synced from the API server)
4. CoreDNS returns the ClusterIP or pod IP

When a pod queries `api.stripe.com`:
1. Doesn't match any cluster zone → falls through to the `forward` plugin
2. CoreDNS forwards to the upstream resolver (usually the node's `/etc/resolv.conf` → VPC DNS → 8.8.8.8)
3. Response cached by the `cache` plugin, returned to pod

### The Corefile — annotated example

```
# Default CoreDNS Corefile for EKS (simplified)

# Zone 1: cluster-local DNS (Services, pods)
cluster.local:53 {
    errors                    # log errors to stderr
    health {                  # /health endpoint on port 8080
       lameduck 5s            # drain in-flight queries before stopping (graceful shutdown)
    }
    ready                     # /ready endpoint for readinessProbe
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure           # allow pod A records (10-0-0-1.default.pod.cluster.local)
       fallthrough in-addr.arpa ip6.arpa  # pass PTR queries that don't match to next plugin
    }
    prometheus :9153          # expose metrics on :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000    # cap upstream concurrent queries
    }
    cache 30                  # cache positive responses for 30s
    loop                      # detect forwarding loops
    reload                    # hot-reload Corefile on ConfigMap change
    loadbalance               # round-robin A/AAAA record responses
}

# Zone 2: everything else (external DNS)
.:53 {
    errors
    health
    forward . 8.8.8.8 8.8.4.4 {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

**Plugin execution order matters**: plugins run top-to-bottom for each query. The `kubernetes` plugin either answers the query (returns) or falls through to the next plugin. Order determines priority.

### Key plugins to know

| Plugin | What it does |
|---|---|
| `kubernetes` | Answers DNS for Services, pods, and namespaces from the API server cache |
| `forward` | Forwards unmatched queries to upstream resolvers |
| `cache` | Caches positive and negative responses; dramatically reduces upstream queries |
| `errors` | Logs SERVFAIL and upstream errors |
| `health` | Liveness endpoint at `:<port>/health` |
| `ready` | Readiness endpoint; returns 200 only when all plugins are ready |
| `prometheus` | Exposes metrics at `:<port>/metrics` |
| `loop` | Detects and breaks forwarding loops |
| `reload` | Watches the ConfigMap; hot-reloads without pod restart |
| `loadbalance` | Round-robins A record answers (simple client-side LB) |

### The `ndots:5` query amplification problem

Every pod's `/etc/resolv.conf` has:
```
options ndots:5
```

`ndots:5` means: if the queried name has fewer than 5 dots, try appending all the search domains before trying the bare name. For a query to `redis`:

```
1. redis.default.svc.cluster.local  → NXDOMAIN
2. redis.svc.cluster.local          → NXDOMAIN
3. redis.cluster.local              → NXDOMAIN
4. redis.your-company.internal      → NXDOMAIN
5. redis.                           → found!  (or still NXDOMAIN)
```

A query to `api.stripe.com` (3 dots, < 5):
```
1. api.stripe.com.default.svc.cluster.local  → NXDOMAIN
2. api.stripe.com.svc.cluster.local          → NXDOMAIN
3. api.stripe.com.cluster.local              → NXDOMAIN
4. api.stripe.com.                           → answer
```

3 NXDOMAIN queries for every 1 real query. Under load, this multiplies CoreDNS's workload by 3-4×. NodeLocal's cache absorbs the repeated NXDOMAINs.

Mitigation for known FQDNs: use trailing dots in application config: `api.stripe.com.` — a trailing dot signals "this is already fully qualified, don't append search domains."

---

## NodeLocal DNS Cache — architecture

### The problem it solves

Standard CoreDNS setup:
```
Pod → UDP → ClusterIP (kube-proxy iptables) → CoreDNS pod (random node)
```

Problems:
1. **Network hop**: even if CoreDNS is on the same node, kube-proxy routes via ClusterIP — the packet might go off-node
2. **CoreDNS load**: all DNS traffic concentrates on 2-3 pods
3. **Conntrack race condition**: UDP DNS through the conntrack table; the race condition drops ~5% of packets under load (see the conntrack doc)

NodeLocal DNS Cache:
```
Pod → (169.254.20.10) → NodeLocal DaemonSet pod (same node) → cache hit → return
                                                               → cache miss → TCP → CoreDNS pod
```

### The DaemonSet and the link-local IP

NodeLocal deploys a DaemonSet running `node-local-dns` (a CoreDNS fork). It listens on:
- `169.254.20.10:53` — the link-local IP, for pod queries
- `<node-IP>:53` — for some legacy compatibility

`169.254.20.10` is assigned to a dummy interface on the node (not to a pod). It's a link-local address — not routable between nodes, only reachable locally. The DaemonSet pod uses `hostNetwork: true` to own this IP.

When NodeLocal is installed, kubelet configures new pods' `/etc/resolv.conf` to point at `169.254.20.10` instead of the CoreDNS ClusterIP:

```
nameserver 169.254.20.10  ← was 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### How NodeLocal avoids the conntrack race

Incoming pod queries to `169.254.20.10`:
- Link-local address → handled locally on the node → **no iptables DNAT, no conntrack entry needed**
- Pod's UDP packet is received directly by the NodeLocal DaemonSet socket

Outbound cache-miss queries to CoreDNS:
- NodeLocal connects to CoreDNS via **TCP** (not UDP)
- TCP connections in conntrack go through the `SYN_SENT → ESTABLISHED` state machine
- The race condition only exists for the UDP path (the "insert_failed" problem for new UDP 5-tuples)
- TCP avoids it because connection establishment is serialized

Result: the conntrack insert_failed race condition is completely bypassed for the vast majority of DNS traffic.

### NodeLocal Corefile (inside the DaemonSet)

```
# NodeLocal DNS Cache Corefile
cluster.local:53 {
    errors
    cache {
       success 9984 30    # cache up to 9984 positive responses for 30s
       denial 9984 5      # cache NXDOMAIN for 5s (don't cache misses too long)
    }
    reload
    loop
    bind 169.254.20.10   # only listen on the link-local IP
    forward . __PILLAR__DNS__SERVER__ {  # CoreDNS ClusterIP filled in at deploy time
       force_tcp           # USE TCP TO COROUTINES — this is the race-condition fix
    }
    prometheus :9253      # metrics
    health 169.254.20.10:8080
}
```

The critical line: `force_tcp` — all upstream queries from NodeLocal to CoreDNS are TCP.

### Installing NodeLocal

```bash
# The canonical installation:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

# Or via Helm (kube-system-extras chart in many distributions)
```

Before installing: update the `__PILLAR__DNS__SERVER__` placeholder to your CoreDNS ClusterIP and `__PILLAR__LOCAL__DNS__` to `169.254.20.10`.

After installing: verify pods are running on all nodes, then verify existing pods have updated `/etc/resolv.conf` (requires pod restart — NodeLocal changes `resolv.conf` at pod creation, not live).

---

## Metrics and debugging

### Key CoreDNS metrics (Prometheus)

```promql
# Query rate per second
rate(coredns_dns_requests_total[1m])

# Error rate
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[1m])
rate(coredns_dns_responses_total{rcode="NXDOMAIN"}[1m])

# Latency percentiles
histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))

# Cache hit rate
rate(coredns_cache_hits_total[1m]) / rate(coredns_dns_requests_total[1m])

# Forward (upstream) query rate — should drop dramatically with NodeLocal
rate(coredns_forward_requests_total[1m])
```

### Debugging a DNS issue

```bash
# 1. Is DNS working at all?
kubectl run dns-test --image=busybox --restart=Never -it --rm -- nslookup kubernetes.default

# 2. Which nameserver is the pod using?
kubectl exec <pod> -- cat /etc/resolv.conf

# 3. Test NodeLocal is responding
kubectl exec <pod> -- nslookup kubernetes.default 169.254.20.10

# 4. Check NodeLocal pod health on the node
kubectl get pods -n kube-system -l k8s-app=node-local-dns -o wide

# 5. Check NodeLocal cache stats
kubectl exec -n kube-system <node-local-dns-pod> -- dig @169.254.20.10 kubernetes.default.svc.cluster.local

# 6. CoreDNS logs (verbose mode)
kubectl logs -n kube-system -l k8s-app=kube-dns --since=5m

# 7. ndots:5 amplification — check NXDOMAIN rate
# If coredns_dns_responses_total{rcode="NXDOMAIN"} is high, ndots:5 is amplifying
```

### The ndots:5 amplification check

```bash
# Count NXDOMAIN responses — high means ndots amplification
kubectl exec -n kube-system <coredns-pod> -- \
  cat /proc/net/udp | wc -l  # rough proxy; use Prometheus for production
```

In production, if your NXDOMAIN rate is more than 2× your expected external query rate, ndots:5 is amplifying. Mitigation: have applications use FQDNs with trailing dots, or reduce ndots in the CoreDNS config.

---

## The interview answer in 60 seconds

> "NodeLocal DNS Cache improves DNS reliability and latency for two distinct reasons.
>
> Latency: without NodeLocal, every DNS query goes through kube-proxy to a CoreDNS pod, which might be on a different node. With NodeLocal, a DaemonSet on every node listens on `169.254.20.10`. Cache hits are served from the same node in microseconds — no network hop, no ClusterIP. CoreDNS only sees cache misses, so it handles a fraction of the original query volume.
>
> Reliability: there's a well-known kernel race condition in conntrack for UDP DNS. When many pods on the same node all do DNS lookups simultaneously, two goroutines can try to insert the same 5-tuple into the conntrack table simultaneously — the second insert fails, and the packet is dropped. The pod retries, the retry might succeed, and you see intermittent 5-10% DNS failures under load. NodeLocal fixes this in two ways: most queries hit the local cache (no conntrack at all), and cache misses are sent to CoreDNS over TCP — TCP avoids the UDP conntrack race condition.
>
> The combined effect: DNS latency drops from 1-5ms to sub-millisecond for cached responses, and the intermittent NXDOMAIN failures under load disappear entirely."

---

## Self-test drills

### 1. Why might adding NodeLocal DNS Cache dramatically improve DNS reliability and latency?

**Reference answer:** Two reasons. Latency: local node cache serves responses without network hop; CoreDNS load drops dramatically. Reliability: fixes the conntrack UDP race condition — cache hits bypass conntrack entirely, cache misses use TCP which avoids the UDP insert_failed race. See the 60-second interview answer above for the full formulation.

### 2. What is ndots:5 and why does it amplify DNS query volume?

**Reference answer:** `ndots:5` is a resolver setting that causes names with fewer than 5 dots to have all search domain suffixes appended before trying the bare name. For external FQDNs like `api.stripe.com` (3 dots), the resolver first tries `api.stripe.com.default.svc.cluster.local`, `api.stripe.com.svc.cluster.local`, `api.stripe.com.cluster.local` — generating 3 NXDOMAIN queries before the successful lookup. Under load this multiplies CoreDNS workload by 3-4×. Mitigation: use FQDNs with trailing dots in app config, or tune ndots down in cluster DNS config for specific namespaces.

### 3. Why does NodeLocal use TCP to talk to CoreDNS?

**Reference answer:** TCP avoids the conntrack UDP race condition. UDP DNS queries that are cache misses would still trigger the conntrack insert_failed race if sent over UDP. TCP connections go through `SYN → SYN-ACK → ACK` handshake, which serializes the conntrack entry creation — no simultaneous-insert race. The `force_tcp` directive in NodeLocal's Corefile enforces this for all upstream queries.

### 4. What happens if a NodeLocal DaemonSet pod crashes on a node?

**Reference answer:** Pods on that node have `169.254.20.10` as their nameserver. If the NodeLocal pod is down, DNS queries to `169.254.20.10` time out. The resolver falls back to the next nameserver in `/etc/resolv.conf` — but only if there's one. In default NodeLocal installations, there's only one nameserver (the NodeLocal IP). To mitigate: configure NodeLocal with CoreDNS ClusterIP as a fallback, or monitor NodeLocal pod health aggressively with a PodDisruptionBudget and a liveness probe. This is why NodeLocal pods should not be disrupted during node maintenance without verifying DNS continuity.

---

## Further reading / watching

- **Kubernetes docs — NodeLocal DNS Cache**: `kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/` — official installation guide and architecture
- **CoreDNS docs**: `coredns.io` — Corefile reference, plugin documentation, troubleshooting guide
- **Weave Works blog — "Racy conntrack and DNS lookup timeouts"**: the 2018 post that documented the conntrack race condition; search "weave works racy conntrack dns" — this is the origin post for this entire topic area
- **Kubernetes GitHub KEP #32 — NodeLocal DNS Cache**: the original design doc; explains the race condition and solution in kernel terms
- **Netto Farah — "Debugging DNS in Kubernetes"** on the Datadog blog: search "datadog dns kubernetes debugging" — excellent practical debugging guide with real dig/kubectl commands

---

## The 4 dimensions (senior framing)

- **Tech**: Corefile plugin chain; `kubernetes` plugin for cluster DNS; `cache` plugin for NXDOMAIN caching; NodeLocal DaemonSet on `169.254.20.10`; `force_tcp` to CoreDNS to bypass conntrack UDP race; `ndots:5` amplification and trailing-dot mitigation.
- **People**: DNS failures look like app failures — the first 30 minutes of any incident will be spent blaming the app before someone checks DNS. Make DNS the first item in your debugging runbook. Train everyone on the `kubectl exec -- nslookup` smoke test. NodeLocal installation is invisible to developers once done, which is the ideal kind of infrastructure improvement.
- **CI/CD**: include a DNS smoke test in cluster provisioning CI: `kubectl run dns-check --image=busybox --restart=Never --rm -it -- nslookup kubernetes.default`. After NodeLocal is deployed, verify `/etc/resolv.conf` in newly created pods. Alert on NodeLocal DaemonSet pod count < node count — a missing pod means that node's pods have unreliable DNS.
- **Operations**: alert on `coredns_dns_responses_total{rcode="SERVFAIL"}` rate > 0 for more than 30 seconds (indicates upstream resolution failure). Alert on NodeLocal pod crash-loop or pending state. Monitor the cache hit rate — if it drops below 60%, check for ndots amplification or application patterns causing cache churn. CoreDNS horizontal scaling (more replicas) helps throughput but doesn't fix the conntrack race — NodeLocal is the right fix for the reliability problem.
