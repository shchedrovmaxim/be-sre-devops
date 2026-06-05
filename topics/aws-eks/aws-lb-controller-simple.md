# AWS Load Balancer Controller — the simple version (the delivery route)

> Read this first. Once the analogy clicks, the [deep-dive doc](./aws-lb-controller.md) becomes easy.

One idea:

> **Target type `instance` is delivery to the building's front desk (NodePort), who then routes to the right apartment (pod). Target type `ip` is direct delivery to the apartment door (pod IP). Same package. Very different path.**

That distinction determines whether source IP is preserved, whether you pay an extra routing hop, and whether a pod restart breaks your connection mid-flight.

---

## The delivery analogy

| Delivery world | K8s / AWS LB world |
|---|---|
| Building address (NodePort) | Target type `instance` — LB sends traffic to node IP + port |
| Front desk re-routes to apartment | kube-proxy NATs to pod IP |
| Delivery truck doesn't know the apartment's real address | Source IP is SNAT'd by kube-proxy — the pod sees the node's IP |
| Direct-to-door delivery | Target type `ip` — LB sends traffic directly to pod IP |
| Delivery truck knows the real door | Source IP preserved — pod sees client's real IP |
| Needs GPS (VPC routing map) | Requires VPC CNI (pod IPs must be routable in the VPC) |
| Front desk may be on break (NodePort 30080 not responding) | kube-proxy hop can add latency and failure surface |

The key insight: **target type `ip` only works if pod IPs are real VPC IPs**. That means VPC CNI must be the CNI. An overlay CNI with virtual pod IPs breaks target type `ip` entirely.

---

## The 2 core concepts that confuse people

### 1. NLB vs ALB — which one and when?

**NLB (Network Load Balancer) = L4.** It sees TCP/UDP. It doesn't look at HTTP headers, cookies, or paths. Use it when:
- You need TCP passthrough (TLS terminated by the pod, not the LB)
- You need UDP
- You need the absolute lowest latency
- You need to preserve the client source IP (with target type `ip`)

**ALB (Application Load Balancer) = L7.** It sees HTTP. It can route by path (`/api → service A`, `/static → S3`), by header, by host. Use it when:
- You need path-based or host-based routing
- You want the LB to terminate TLS (ACM cert) and handle HTTP/2 or gRPC
- You want the LB to make intelligent routing decisions based on request content

The controller manages both. An `Ingress` resource creates an ALB. A `Service` of type `LoadBalancer` with the right annotations creates an NLB.

### 2. What does the pod readiness gate do?

When a deployment rolls out, pods become `Running` before they're registered in the target group and before they've passed their first health check. Without the readiness gate, traffic could be routed to a pod that isn't in the target group yet — causing errors.

The AWS LB Controller injects a readiness gate (`target-health.elbv2.k8s.aws/...`) into pods. The pod is only `Ready` once the LB target group reports it as healthy. This makes rolling deployments safe: the new pod must pass LB-level health checks before the old pod is terminated.

You enable it by adding the annotation to the namespace:
```bash
kubectl annotate namespace my-app \
  elbv2.k8s.aws/pod-readiness-gate-inject=enabled
```

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What does the AWS LB Controller do? | Watches K8s Ingress and Service resources; creates/updates real AWS ALBs and NLBs |
| Target type `instance` — how does traffic flow? | LB → node IP:NodePort → kube-proxy → pod. Source IP SNAT'd at kube-proxy. |
| Target type `ip` — how does traffic flow? | LB → pod IP directly (VPC routing). Source IP preserved. Requires VPC CNI. |
| When do you need target type `ip`? | When you need real client source IP in the pod; or to avoid the kube-proxy hop |
| What's TargetGroupBinding? | A CRD that lets you attach an existing AWS target group to K8s pods directly |
| What breaks PreserveSourceIP with target type `instance`? | kube-proxy does SNAT — the pod sees the node's IP, not the client's |
| What's the pod readiness gate for? | Ensures pods don't receive LB traffic until they're healthy in the target group |
| NLB vs ALB in one line? | NLB = TCP/UDP, fastest, no HTTP routing. ALB = HTTP/L7, path routing, ACM certs. |

---

## Self-test (the killer interview question)

Out loud:

> **"When would you use target-type IP vs target-type instance for an NLB, and why does it matter for K8s?"**

**Reference answer (intuitive version):**

"Target type `instance` routes traffic to each node's NodePort. From there, kube-proxy NATs the connection to the actual pod. Two consequences: first, there's an extra hop through kube-proxy on the node, which adds a small but measurable latency cost. Second, kube-proxy does SNAT — the source IP in the packet is replaced with the node's IP. The pod cannot see the real client IP. If you're doing IP-based rate limiting, geo-blocking, or logging, this is a problem.

Target type `ip` routes traffic directly to the pod's IP in the VPC. No kube-proxy hop. Source IP preserved end-to-end. But it only works if the pod IP is a real VPC IP — meaning you must be running VPC CNI. With an overlay CNI, the pod's IP is virtual and not routable from the NLB.

The choice in practice: use target type `ip` as the default for new workloads on EKS with VPC CNI — it's faster, simpler (no kube-proxy in the path), and preserves source IP. Use target type `instance` only if you have a specific reason: using an overlay CNI, using Windows nodes, or running an NLB in front of a nodegroup where VPC CNI isn't the CNI.

One more gotcha: during a rolling deployment, target type `instance` can route to any node — the kube-proxy on that node then forwards to any pod, even one that's being terminated. Target type `ip` registers individual pod IPs; when the pod is deleted, it's deregistered. With the pod readiness gate, the new pod isn't registered until healthy. This makes rolling deployments much cleaner with target type `ip`."

---

## Further reading / watching

- **AWS LB Controller docs**: [kubernetes-sigs.github.io/aws-load-balancer-controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) — the annotations reference is the most-referenced page; bookmark it.
- **AWS blog — "Exposing Kubernetes applications, Part 3: NLB"**: search "AWS NLB target type ip blog" — explains the target type difference with diagrams.
- **EKS Best Practices — Networking**: [aws.github.io/aws-eks-best-practices/networking/loadbalancing/](https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/) — NLB vs ALB decision tree.
- **TargetGroupBinding docs**: in the AWS LB Controller docs under "TargetGroupBinding" — the most underrated feature for attaching external target groups to K8s services.

---

## Next: the deep-dive

When the delivery analogy and the two target types feel obvious, jump to [`aws-lb-controller.md`](./aws-lb-controller.md). The deep-dive covers:

- The full annotation set for NLB and ALB
- TargetGroupBinding CRD and its use cases
- Ingress class + IngressClassParams for multi-controller setups
- ACM cert integration and TLS termination patterns
- Health check interaction with K8s readiness probes
- Surge and draining behavior on deployments and node termination
- 4-dimensions framing for interview
