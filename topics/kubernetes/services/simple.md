# Kubernetes Services — the simple version (the reception desk analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **A Service is a stable address in front of a set of pods. The pods come and go; the address doesn't.**

That's the whole concept. Everything else (ClusterIP, NodePort, LoadBalancer, kube-proxy, iptables) is just precision on top of that.

---

## The office building reception desk

Imagine a company in an office building. You, an outside visitor, want to reach "the payments team."

You don't call each engineer directly — you call **reception** (the Service). Reception routes you to whoever is at their desk (a healthy pod). If someone goes home sick, reception just stops routing to them. You still call the same number.

| In the office world | In the K8s world |
|---|---|
| Reception desk phone number | **ClusterIP** — a stable virtual IP inside the cluster |
| Visitor calls from inside the building | Another pod making a request |
| Visitor calls from outside the building | External traffic (needs NodePort or LoadBalancer) |
| Receptionist checks who is at their desk | kube-proxy programs iptables/IPVS to route to ready pods |
| Someone out sick → reception skips them | Pod with failing readiness probe → removed from Service endpoints |
| Building intercom extension number | NodePort — same idea but reachable from outside the building |
| Cloud doorman (AWS ALB) who connects callers outside | LoadBalancer — cloud-managed LB in front of NodePort |

---

## The 5 Service types in plain English

### ClusterIP (the default)

A virtual IP that only works **inside the cluster**. Other pods reach it by name (`my-service.my-namespace.svc.cluster.local`) or directly by IP. kube-proxy programs every node's iptables to DNAT the virtual IP to a real pod IP.

**Use it for**: internal APIs, databases, anything that only needs to be reachable by other services in the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  selector:
    app: payment-api
  ports:
  - port: 80
    targetPort: 8080
  # type: ClusterIP  ← this is the default; you can omit it
```

### NodePort

Opens a port (default range: 30000–32767) on **every node** in the cluster. Traffic to `<any-node-IP>:<nodePort>` gets routed to your pods. ClusterIP is automatically created too.

**Use it for**: exposing a service to internal infrastructure (load balancers, CI runners) without a cloud provider. Rarely used directly in prod.

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31080   # omit this to let K8s assign one
```

### LoadBalancer

Asks the cloud provider to provision a real load balancer (AWS NLB, GCP TCP LB) and point it at the NodePort. Your service is now reachable from the internet with a real external IP.

**Under the hood**: it's a NodePort with a cloud LB in front. The LB gets the external IP; the NodePort is the path behind it.

**Use it for**: exposing a single service to the internet. Gets expensive with many services — use Ingress instead if you have many HTTP services.

### ExternalName

No pods, no proxying. Just a DNS CNAME alias. When a pod resolves `my-service`, it gets the external DNS name you configured.

```yaml
spec:
  type: ExternalName
  externalName: rds.us-east-1.amazonaws.com
```

**Use it for**: giving a K8s-native DNS name to an external system (an RDS database, a third-party API). No actual traffic goes through K8s.

### Headless (clusterIP: None)

No virtual IP at all. DNS resolves directly to the **individual pod IPs**. You get back an A record per ready pod.

```yaml
spec:
  clusterIP: None
  selector:
    app: my-statefulset
```

**Use it for**: StatefulSets where the client needs to know which pod it's talking to (Kafka, Cassandra, Redis Cluster). The StatefulSet controller uses headless services to give pods stable DNS names (`pod-0.my-service`, `pod-1.my-service`).

---

## The gotcha: source IP and `externalTrafficPolicy`

When traffic comes in through a NodePort or LoadBalancer, it may be NATed — your pod sees the **node IP** as the source, not the client's real IP. This matters for logging, rate-limiting, and geo-blocking.

`externalTrafficPolicy: Local` tells kube-proxy to only route to pods **on the same node that received the traffic**. The real client IP is preserved. The trade-off: you lose load balancing across nodes (if the local node has no pods, requests fail).

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # preserve real client IP, lose cross-node LB
```

Most teams set `externalTrafficPolicy: Local` when source IPs matter and accept the pod-distribution trade-off.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's a ClusterIP? | A virtual IP that only works inside the cluster. Programmed by kube-proxy into iptables on every node. |
| What's a NodePort? | A port opened on every node. Traffic hits `<node-IP>:<port>` and gets routed to your pods. |
| What's a LoadBalancer? | NodePort + a real cloud load balancer. The LB is provisioned automatically. |
| What's ExternalName? | Just a DNS CNAME. No pods, no proxy, no real K8s traffic. |
| What's Headless? | `clusterIP: None`. DNS returns pod IPs directly. Used by StatefulSets. |
| Does a ClusterIP have an actual network interface? | No. It's a virtual IP that only exists in iptables rules. No pod or node actually "has" that IP. |
| Can kube-proxy drop requests? | Yes. kube-proxy is eventually consistent. A pod update (dying pod) takes hundreds of ms to seconds to propagate. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through how a request to a ClusterIP gets to a pod, and what changes when you switch to NodePort or LoadBalancer."**

**Reference answer (intuitive version):**

"A ClusterIP is a virtual IP — no real interface, no process listens on it. When pod A sends a request to the ClusterIP, kube-proxy's iptables rules on pod A's node DNAT it to one of the backing pod IPs before the packet even leaves the node. The pod receives the packet as if it was sent directly to it. So the journey is: pod A → kube-proxy iptables DNAT on the node → real pod IP.

When you switch to NodePort, K8s opens an extra port on every node. External traffic hits any node on that port; the same iptables DNAT mechanism then routes to a pod. The ClusterIP still exists behind it — NodePort is just an additional front door on every node.

LoadBalancer adds a cloud-managed LB in front of the NodePort. You get a single stable external IP; the LB distributes to node NodePorts; the node iptables routes to pods. Each layer adds one hop."

---

## Further reading

- [Official K8s docs — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Deep-dive: services.md](./deep-dive.md) — covers kube-proxy modes (iptables vs IPVS vs eBPF), the DNAT flow, externalTrafficPolicy hop trade-off, sessionAffinity, and Headless services for StatefulSets

## Next: the deep-dive

When the reception desk analogy feels obvious, jump to [`services.md`](./deep-dive.md). The deep-dive covers:

- How kube-proxy actually programs iptables (and why it's eventually consistent)
- IPVS mode vs iptables mode — the performance difference at scale
- `externalTrafficPolicy: Local` vs `Cluster` — the source IP / extra hop trade-off in detail
- `sessionAffinity: ClientIP` — when and why
- Headless services + StatefulSet DNS names (`pod-0.svc`, `pod-1.svc`)
- The "two default StorageClasses" equivalent for Services: the ExternalName trap
- 4-dimensions framing
- 4 self-test drills with reference answers
