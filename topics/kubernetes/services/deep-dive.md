# Kubernetes Services — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through how a request to a ClusterIP gets to a pod, and what changes when you switch to NodePort or LoadBalancer."** — naming the virtual IP mechanism, the kube-proxy DNAT chain, the eventual consistency window, and the externalTrafficPolicy trade-off.

> Start with the [simple version](./simple.md) if you haven't read it. The reception desk analogy is the spine of everything here.

---

## The senior framing — Services are about stable addressing, not load balancing

Mid-level engineers think Services are "how you expose a deployment." Senior engineers know Services solve two distinct problems:

1. **Stable addressing**: pods are ephemeral. Their IPs change on restart, reschedule, or rollout. A Service gives you one address that survives all of that.
2. **Endpoint tracking**: the Service + Endpoint controller continuously watches pods and updates the routing table. kube-proxy on every node uses that table to DNAT packets to real pod IPs.

The load balancing is almost a side effect — it falls out of "pick a random healthy endpoint from the routing table." The real engineering is in the consistency model of that table and how fast changes propagate.

---

## Mental model: what a ClusterIP actually is

A ClusterIP is **not** an IP address with a listening process. No node has that IP on a network interface. There is no socket bound to it. It exists only in iptables (or IPVS) rules on every node.

The flow for an in-cluster request:

```
pod-A (10.0.1.5) sends SYN to ClusterIP (10.96.0.10:80)
        │
        ▼
Node's kernel — packet hits iptables PREROUTING chain
        │
        ▼
kube-proxy-generated rule matches 10.96.0.10:80
        │
        ▼
DNAT: destination rewritten to a real pod IP (e.g., 10.0.2.8:8080)
chosen from the endpoint list (probabilistic DNAT)
        │
        ▼
Packet delivered to pod-B (10.0.2.8) on its real IP
pod-B sees source IP = pod-A's IP, destination = its own IP
(the ClusterIP never appears in the packet once it leaves pod-A's node)
```

The DNAT happens in the kernel before the packet leaves the node. It's invisible to the application.

---

## kube-proxy modes

kube-proxy watches the API server for Service and Endpoint/EndpointSlice changes and translates them into kernel-level forwarding rules.

### iptables mode (default)

Generates chains of iptables `DNAT` rules. Packet matching is O(N) in the number of rules — each rule is tried in sequence.

**Scale ceiling**: at ~10,000 Services / 100,000 endpoints, iptables rule churn becomes a performance problem. Each change rewrites and reloads the entire chain for that Service.

### IPVS mode

Uses Linux's IPVS (IP Virtual Server) kernel module. True hash-table lookup — O(1) rule matching regardless of rule count.

```
kube-proxy --proxy-mode=ipvs
```

Better throughput and lower CPU at scale (thousands of Services). Also supports more load-balancing algorithms (round-robin, least-conn, source-hash).

**Gotcha**: requires the `ip_vs*` kernel modules and `ipset`. Not always available in locked-down cloud images.

### eBPF mode (Cilium, eBPF kube-proxy replacement)

Cilium can replace kube-proxy entirely with eBPF programs attached to network interfaces. Even lower overhead, per-connection tracking, L7 visibility. This is the direction the ecosystem is heading. See the CNI/eBPF topic for details.

---

## Service types — the full picture

### ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-api
  namespace: payments
spec:
  selector:
    app: payment-api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  sessionAffinity: None   # default; or ClientIP (see below)
```

DNS name (cluster-internal): `payment-api.payments.svc.cluster.local`

The `port` field is what clients use. The `targetPort` is the container's actual port. They don't have to match.

### NodePort

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31080     # optional; K8s assigns from --service-node-port-range (default 30000-32767)
```

K8s opens port 31080 on **every node** — even nodes that don't have a matching pod. Traffic to `<any-node-ip>:31080` gets DNAT'd to a pod. The ClusterIP still exists and works for in-cluster traffic.

**Gotcha**: the NodePort range is cluster-wide and conflicts if two Services request the same port. The API server validates this at creation time.

### LoadBalancer

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8443
  loadBalancerSourceRanges:        # optional: restrict who can reach the LB
  - 10.0.0.0/8
```

The cloud controller manager watches Services of type LoadBalancer and provisions the cloud LB. On AWS this means an NLB (or legacy CLB). The LB's external IP appears in `.status.loadBalancer.ingress[].ip` (or `hostname` on AWS).

**The stack under the hood**: cloud LB → NodePort on a node → iptables DNAT → pod. Three hops minimum.

### ExternalName

```yaml
spec:
  type: ExternalName
  externalName: rds.us-east-1.amazonaws.com
```

No selector, no endpoints, no proxying. CoreDNS returns a CNAME to `rds.us-east-1.amazonaws.com`. The client resolves it and connects directly.

**Gotcha**: TLS certificates are issued for the actual hostname. If your app validates the TLS cert against the ExternalName (which it won't see — it connects to the resolved CNAME), you'll get cert mismatch errors. Use this only for external services your app doesn't TLS-verify by hostname.

### Headless (clusterIP: None)

```yaml
spec:
  clusterIP: None
  selector:
    app: my-kafka
```

CoreDNS returns an A record **per ready pod** rather than a single virtual IP. The client receives all pod IPs and decides how to use them.

StatefulSet pods automatically get stable DNS names via their headless Service:

```
pod-0.my-kafka.my-namespace.svc.cluster.local → 10.0.1.5
pod-1.my-kafka.my-namespace.svc.cluster.local → 10.0.1.6
pod-2.my-kafka.my-namespace.svc.cluster.local → 10.0.1.7
```

This is how Kafka clients, Cassandra seeds, and Redis Cluster members address each other by identity — not just "any pod."

---

## externalTrafficPolicy: Local vs Cluster

This field controls what kube-proxy does with traffic arriving at a NodePort or LoadBalancer.

### Cluster (default)

```yaml
spec:
  externalTrafficPolicy: Cluster
```

kube-proxy can route to **any pod in the cluster**, not just the ones on the receiving node. To do this, it uses SNAT to replace the client's source IP with the node's IP (so the response knows how to come back).

- **Pro**: perfect load distribution, works even with uneven pod placement.
- **Con**: client's real source IP is lost. Logs, rate limiters, geo-blocks see the node IP instead.

### Local

```yaml
spec:
  externalTrafficPolicy: Local
```

kube-proxy only routes to pods **on the same node** that received the traffic. No SNAT — the real client IP is preserved in the packet.

- **Pro**: source IP is preserved. Useful for logging, rate limiting, Cloudflare IP passthrough.
- **Con**: if the node has no pods, traffic is dropped. Load is distributed across **nodes**, not pods. If pod distribution across nodes is uneven, load is uneven.

**Senior tip**: when using `externalTrafficPolicy: Local`, ensure your pods are spread evenly across nodes with `topologySpreadConstraints`. Otherwise a node with 2 pods gets 1× traffic and a node with 1 pod gets 1× traffic — the 2-pod node's pods get half the traffic of the 1-pod node's pod.

---

## sessionAffinity: ClientIP

By default, each connection is independently routed to a random pod. With sessionAffinity:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # default: 10800 (3 hours)
```

kube-proxy records which pod a client IP was routed to and sticks them there for the timeout window. Implemented in iptables with a `recent` module match.

**When to use**: stateful protocols (WebSockets, long-polling) where reconnecting to a different pod would lose state. If your pods are truly stateless, don't bother.

**Gotcha**: IPVS mode has better session affinity implementation than iptables mode. In iptables mode, the "clientIP" is the source IP — if your clients share an IP (CGNAT, corporate proxy), all of them are pinned to the same pod.

---

## Endpoint propagation and the eventual consistency window

This is the most important production gotcha on Services.

When a pod is deleted or becomes unready:
1. The endpoint controller removes the pod IP from the EndpointSlice.
2. kube-proxy on each node watches the API server and receives the update.
3. kube-proxy rewrites iptables rules on that node.

Step 2 and 3 are asynchronous. In a large cluster with many nodes, this can take **hundreds of milliseconds to seconds**. During that window, requests from other pods can still be routed to the dying pod's IP.

This is exactly why `preStop: sleep 10` is the standard pattern — see the pod-lifecycle deep-dive. The sleep gives kube-proxy time to propagate before the app starts shutting down.

Concretely:

```
T+0ms:   Pod termination begins. Endpoint removed from EndpointSlice.
T+100ms: kube-proxy on node-1 receives update. Stops routing there.
T+300ms: kube-proxy on node-2 receives update. Stops routing there.
T+800ms: kube-proxy on node-3 receives update. (Cluster was busy.)
T+10000ms: preStop sleep ends. SIGTERM sent to app.
```

With the 10-second sleep, by the time the app starts shutting down, all kube-proxy instances should have already stopped routing to it.

---

## The complete iptables chain (what kube-proxy actually generates)

For the curious — this is what you see on a node with `iptables-save | grep payment-api`:

```
# Main dispatch chain
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

# Load balance across 3 pods (probabilistic DNAT)
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.33333 -j KUBE-SEP-AAA
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.50000 -j KUBE-SEP-BBB
-A KUBE-SVC-XXXX -j KUBE-SEP-CCC

# Each SEP rule DNATs to a specific pod IP
-A KUBE-SEP-AAA -p tcp -j DNAT --to-destination 10.0.1.5:8080
-A KUBE-SEP-BBB -p tcp -j DNAT --to-destination 10.0.2.8:8080
-A KUBE-SEP-CCC -p tcp -j DNAT --to-destination 10.0.3.2:8080
```

The probabilities are chosen so each pod gets an equal share of new connections. The first rule fires with p=1/3, the second with p=1/2 of the remaining 2/3 (= 1/3), and the third catches the rest (1/3).

---

## The interview answer in 60 seconds

> "A ClusterIP is a virtual IP that exists only in kube-proxy's iptables rules on each node — no actual interface or process listens on it. When pod A sends a packet to the ClusterIP, the kernel intercepts it at the PREROUTING chain, finds kube-proxy's DNAT rule, rewrites the destination to a real pod IP chosen probabilistically from the endpoint list, and delivers the packet. The application never sees the ClusterIP.
>
> NodePort adds a port on every node. Traffic hits `<node-ip>:<nodePort>`, the same DNAT chain fires, and it ends up at a pod. The ClusterIP still works in parallel.
>
> LoadBalancer adds a cloud LB in front of the NodePorts. You get an external IP; the LB distributes to node NodePorts; the node routes to pods. Three hops.
>
> The key production gotcha: kube-proxy is eventually consistent. When a pod is removed from the endpoint list, it takes hundreds of milliseconds to seconds for every node to reprogram its iptables. During that window, dying pods still receive traffic. The fix is `preStop: sleep 10` — wait for the propagation before starting your shutdown.
>
> For source IP: `externalTrafficPolicy: Local` preserves the real client IP but requires pods to be spread across nodes. `Cluster` (the default) gives better distribution but SNATs source IPs to the node IP."

---

## Self-test drills

### 1. Walk me through how a request to a ClusterIP gets to a pod, and what changes when you switch to NodePort or LoadBalancer.

**Reference answer**: ClusterIP = virtual IP in iptables DNAT rules. Packet hits PREROUTING, gets DNAT'd to a real pod IP. NodePort opens an extra port on every node — same DNAT chain behind it. LoadBalancer = cloud LB → NodePort → DNAT → pod. ClusterIP still works for in-cluster traffic in all cases.

### 2. I set `externalTrafficPolicy: Local`. My service is sometimes returning 502 errors. What might be happening?

**Reference answer**: The node receiving the traffic has no ready pods. With `Local`, kube-proxy drops the traffic rather than routing to another node. Fixes: use `topologySpreadConstraints` to spread pods across nodes, or use a cloud LB that supports health-checking per node and draining nodes with no healthy pods (AWS NLB does this with target group health checks). If you can tolerate SNAT, switch back to `Cluster`.

### 3. Why do I need a headless Service for a StatefulSet? Why can't I use a normal ClusterIP Service?

**Reference answer**: StatefulSet pods need stable, individual identity. A ClusterIP routes to any pod — you can't address pod-0 specifically. Headless (clusterIP: None) makes CoreDNS return per-pod A records and gives each pod a stable DNS name (`pod-0.svc.namespace.svc.cluster.local`). Kafka consumers need to connect to specific partition leaders; Cassandra seeds need to talk to specific peers. A random-routing ClusterIP breaks that.

### 4. I have 100 Services in my cluster and kube-proxy is using a lot of CPU. What are my options?

**Reference answer**: Switch to IPVS mode (`--proxy-mode=ipvs`). iptables mode scales O(N) — each packet traverses up to N rules. IPVS is a hash table: O(1) lookups. Better option at scale: replace kube-proxy with Cilium's eBPF dataplane entirely. It handles endpoint selection in the kernel with even lower overhead and gives you L7 visibility as a bonus.

---

## Further reading

- [K8s docs — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [K8s docs — Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)
- [Thockin's networking diagrams](https://github.com/mermaid-js/mermaid) — Tim Hockin's networking deep-dives are the canonical source for K8s network internals
- [Cilium docs — kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- See also: `endpoints-endpointslices.md` (how kube-proxy gets its endpoint data), `ingress-vs-gateway-api.md` (HTTP routing above Services)

---

## The 4 dimensions (senior framing)

- **Tech**: ClusterIP = virtual IP in iptables/IPVS/eBPF; kube-proxy eventual consistency window; `externalTrafficPolicy: Local` for source IP preservation; headless for StatefulSet identity; IPVS or Cilium eBPF at scale; `preStop: sleep` to cover propagation lag.
- **People**: developers default to `type: LoadBalancer` for everything — that's one cloud LB per Service, which gets expensive fast. Pair with platform team to establish: stateless HTTP → Ingress; internal APIs → ClusterIP; stateful protocols → Headless. Document this as a decision guide, not a rule they have to memorize.
- **CI/CD**: enforce Service type policy with OPA/Kyverno (no `type: LoadBalancer` without a justification annotation). Add a linting step that validates `sessionAffinity` is only set on Services whose pods are known to be stateful. Alert when a new LB Service appears without a corresponding Ingress annotation.
- **Operations**: monitor endpoint churn rate — high churn (rapid pod cycling) causes kube-proxy CPU spikes from iptables rewrites. Track LB provisioning latency in cloud controllers (can take 30–90 seconds on AWS). Runbook for "Service not routing": check endpoints (`kubectl get ep`), check readiness probes, check kube-proxy logs on the node.
