# AWS Load Balancer Controller — the deep-dive

> **Goal**: by the end you can answer **"When would you use target-type IP vs target-type instance for an NLB, and why does it matter for K8s?"** — naming the kube-proxy hop, source IP preservation, VPC CNI dependency, pod readiness gates, and draining behavior. You can also design an Ingress for an ALB with ACM certs and path routing without looking anything up.

> Start with the [simple version](./simple.md) if you haven't — the direct-delivery analogy is the mental model.

---

## The senior framing — the controller bridges K8s intent and AWS primitives

The AWS Load Balancer Controller is a Kubernetes controller that watches `Ingress` and `Service` resources and translates them into AWS ALB and NLB configurations. It replaced the in-tree cloud provider load balancer integration (which still exists but is being deprecated).

The key conceptual shift: **AWS LB Controller is declarative**. You declare the desired LB state in K8s YAML; the controller reconciles AWS to match. You don't call the AWS console or CLI to create a target group. The controller does it, tracks it, and garbage-collects it when the K8s resource is deleted.

---

## NLB vs ALB — the full picture

### NLB (Network Load Balancer) — L4

Routes by TCP/UDP. Does not inspect HTTP headers, paths, or cookies. Passes through the packet (or opens a new TCP connection, depending on target type).

Use cases:
- TLS passthrough (NLB passes the encrypted bytes; pod terminates TLS with its own cert)
- gRPC and WebSocket traffic that needs raw TCP
- UDP (e.g., a DNS server, a game server, QUIC/HTTP/3)
- Lowest possible latency (no HTTP header parsing)
- Source IP preservation (with target type `ip`)

Create via `Service` of type `LoadBalancer`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - port: 443
      targetPort: 8080
```

### ALB (Application Load Balancer) — L7

Routes by HTTP. Reads headers, paths, host names, query params. Terminates TLS. Supports HTTP/2 and gRPC at the LB level.

Use cases:
- Path-based routing (`/api → service A`, `/web → service B`)
- Host-based routing (`api.example.com → service A`, `app.example.com → service B`)
- ACM TLS termination (the LB holds the cert; pods see plain HTTP)
- WAF attachment (AWS WAF rules at the LB layer)
- OIDC authentication at the ALB (redirect to cognito/okta before forwarding to pod)

Create via `Ingress`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/success-codes: "200,201"
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

## Target types — the full technical story

### Target type `instance`

Traffic flow:
```
Client → NLB → Node IP:NodePort → kube-proxy iptables rule → Pod IP
```

What happens at the kube-proxy hop:
1. The packet arrives at the node on NodePort (e.g., TCP 32080).
2. kube-proxy's iptables rule intercepts it, does DNAT (destination NAT) to the pod IP.
3. kube-proxy also does SNAT (source NAT) — replaces the source IP with the node's IP.
4. The pod receives the packet; the `remoteAddr` is the **node's IP**, not the client's.

Consequence: if your app logs source IPs, rate-limits by IP, or uses IP for geo-detection, you'll see node IPs instead of client IPs.

Exception: `externalTrafficPolicy: Local` on the Service tells kube-proxy not to SNAT and not to forward to pods on other nodes. The pod sees the real client IP. But now only nodes that have a pod running will receive traffic — you need the LB to health-check individual pods and route only to nodes that have healthy pods.

### Target type `ip`

Traffic flow:
```
Client → NLB → Pod IP (direct VPC routing)
```

Requirements:
- VPC CNI must be the CNI (pod IPs must be real VPC secondary IPs).
- The ENI that holds the pod's secondary IP must be in a subnet that the NLB can route to.

What you get:
- No kube-proxy in the path — one fewer hop, lower latency.
- Source IP preserved end to end.
- Individual pod IPs registered in the target group — when a pod is deleted, its IP is deregistered, not the node.

The operational implication during rollovers: target type `ip` + pod readiness gate means:
1. New pod starts.
2. Controller registers pod IP in target group.
3. LB starts health-checking the pod.
4. Once healthy, readiness gate passes → pod is `Ready` → old pod is terminated.
5. Old pod IP is deregistered with a connection draining period.

This is the cleanest zero-downtime deployment path.

---

## Pod readiness gate — deep dive

Without the readiness gate, a pod becomes `Ready` in Kubernetes (all containers running + readiness probe passing) before the LB has registered it and before the first health check passes. In target type `ip` mode, this means the pod could start receiving Deployment rollout traffic before the LB is actually sending traffic to it — but the edge case is the reverse: pod is `Ready`, old pod is terminated, but new pod isn't yet healthy in the LB.

The readiness gate adds an extra condition to the pod's Ready status: `target-health.elbv2.k8s.aws/<target-group-arn>: Healthy`. The pod is not `Ready` until the LB reports it healthy.

Enable per namespace:
```bash
kubectl annotate namespace production \
  elbv2.k8s.aws/pod-readiness-gate-inject=enabled
```

Gotcha: the readiness gate is only injected at pod creation time (via the mutating webhook). Existing pods don't get it retroactively. You need to roll all pods in the namespace after enabling the annotation.

---

## TargetGroupBinding CRD

TargetGroupBinding is the underrated feature. It lets you attach an existing AWS target group to K8s service endpoints — without the LB Controller managing the LB itself.

Use case: you have an NLB provisioned by Terraform (or AWS console), and you want the controller to manage just the target group membership as pods come and go.

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: my-api-tgb
spec:
  serviceRef:
    name: my-api
    port: 8080
  targetGroupARN: arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123
  targetType: ip
```

The controller then syncs pod IPs to the target group as pods scale up/down/replace. You keep full control of the LB configuration in Terraform.

---

## IngressClass and IngressClassParams

If you run multiple Ingress controllers (e.g., AWS LB Controller for external traffic and nginx-ingress for internal traffic), you need to tell each Ingress which controller should handle it.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb-public
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: alb-public-params
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: IngressClassParams
metadata:
  name: alb-public-params
spec:
  scheme: internet-facing
  ipAddressType: ipv4
  group:
    name: shared-alb-group   # multiple Ingresses share one ALB
```

The `group.name` annotation is important for cost: by default, each `Ingress` resource creates a separate ALB. Sharing an ALB across Ingresses (same `group.name`) means one LB, multiple listener rules — fewer ALBs, lower cost, but shared fate (one Ingress change can affect all).

---

## ACM cert integration

The ALB terminates TLS. Certs are in ACM (AWS Certificate Manager):

```yaml
annotations:
  alb.ingress.kubernetes.io/certificate-arn: >-
    arn:aws:acm:us-east-1:123456789012:certificate/abc123,
    arn:aws:acm:us-east-1:123456789012:certificate/def456
```

Multiple ARNs comma-separated = multiple certs on one listener. ALB uses SNI to pick the right cert per hostname.

The controller **does not** provision or rotate ACM certs — that's cert-manager's job (or manual ACM management). The controller just associates existing ACM cert ARNs with the listener.

---

## Health checks and K8s readiness probes — the interaction

The LB health check and K8s readiness probe are independent:

| Aspect | K8s readiness probe | LB health check |
|---|---|---|
| Configured where | In pod spec | In Ingress/Service annotations |
| Checked by | kubelet | Load balancer |
| Affects | `kubectl get pods` READY column; whether pod receives `Service` traffic | Whether pod is `InService` in the target group |
| Default | Same path/port as livenessProbe | `HTTP:8080/` unless annotated otherwise |

They should align but aren't automatically synced. A pod can be `Ready` in K8s but `Unhealthy` in the LB target group (e.g., the LB uses a different port or path). Always set:

```yaml
annotations:
  alb.ingress.kubernetes.io/healthcheck-path: /healthz
  alb.ingress.kubernetes.io/healthcheck-port: "8080"
  alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
  alb.ingress.kubernetes.io/healthy-threshold-count: "2"
  alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
```

---

## Surge and draining on rollovers

When a pod is being terminated (rolling update, node drain), the LB Controller:
1. Deregisters the pod IP from the target group.
2. The LB begins draining the pod — it stops sending new connections; existing connections are allowed to complete for `deregistration_delay.timeout_seconds` (default: 300 seconds for ALB, 0 for NLB).
3. After the drain timeout, the LB closes remaining connections and the pod is fully deregistered.

The pod's `terminationGracePeriodSeconds` and the LB drain timeout must be aligned. If the pod's grace period is 30 seconds but the LB drain timeout is 300 seconds, the pod dies before connections drain → 503 errors. Set `terminationGracePeriodSeconds >= deregistration_delay_timeout`.

```yaml
annotations:
  alb.ingress.kubernetes.io/target-group-attributes: >-
    deregistration_delay.timeout_seconds=30,
    slow_start.duration_seconds=30
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # >= deregistration delay
```

The `slow_start.duration_seconds` is the reverse: a new pod gets a ramped traffic share over N seconds instead of immediately getting full traffic. Useful for JVM services that take 20-30 seconds to warm up caches.

---

## The interview answer in 60 seconds

> "Target type `instance` routes traffic to each node's NodePort. kube-proxy on the node then NATss to the pod. There are two consequences: an extra hop (adds latency, adds failure surface), and SNAT at the kube-proxy — the pod sees the node's IP, not the client's. Source IP preservation requires `externalTrafficPolicy: Local`, which has its own trade-off: only nodes with a pod accept traffic.
>
> Target type `ip` routes directly to the pod IP in the VPC. No kube-proxy hop. Real client IP preserved end-to-end. But it requires VPC CNI — pod IPs must be real VPC secondary IPs, not virtual IPs from an overlay.
>
> For K8s deployments, target type `ip` + pod readiness gate is the cleaner story: the pod isn't marked Ready until the LB reports it healthy, so rolling updates don't shift traffic before the new pod is ready to serve.
>
> On draining: set `terminationGracePeriodSeconds` to at least the LB's `deregistration_delay.timeout_seconds`. Otherwise the pod dies before the LB finishes draining connections, and you get 503s."

---

## Self-test drills

### 1. When would you use target-type IP vs target-type instance for an NLB, and why?

**Reference answer:**
- `ip`: direct to pod, no kube-proxy hop, source IP preserved, requires VPC CNI, cleaner deployment story with readiness gates.
- `instance`: works with any CNI (including overlays), more battle-tested with Windows nodes, necessary if you can't use VPC CNI.
- Default recommendation: `ip` for new EKS clusters on VPC CNI. `instance` only when explicitly needed.

### 2. You deployed a new version and see 503s during the rollout. What do you check?

**Reference answer:**
1. Check if pod readiness gate is enabled — if not, new pods might be receiving traffic before LB registration.
2. Check `terminationGracePeriodSeconds` vs `deregistration_delay.timeout_seconds` — if grace period < drain timeout, pods die mid-drain.
3. Check if the new pods' readiness probes are passing — if they're not, the pod is `Running` but not receiving traffic from the Service, but it may still be in the LB's target group if readiness gate isn't set up.
4. Check LB access logs for the source of 503s — is it the LB reporting backend unhealthy, or the backend itself returning 503?

### 3. You want to share one ALB across 5 microservices to reduce costs. How?

**Reference answer:**
- Use Ingress groups: set `alb.ingress.kubernetes.io/group.name: shared-prod` on all 5 Ingress resources.
- The controller creates one ALB with listener rules for each Ingress's routing rules.
- One ALB = one monthly cost (~$16/month) instead of five.
- Trade-off: shared fate. An Ingress misconfiguration on one service can affect others. Also, group-level annotations (like `scheme: internet-facing`) must be consistent across all Ingresses in the group.
- Alternative: IngressClassParams with `group.name` set at the class level — all Ingresses using that class share the ALB automatically.

### 4. What does TargetGroupBinding enable and when is it useful?

**Reference answer:**
- TargetGroupBinding lets you attach an externally-managed target group (created by Terraform, not the controller) to a K8s Service.
- Useful when: LB provisioning is owned by a platform team in Terraform (audit trail, change management), but pod lifecycle management is in K8s.
- The controller handles only the dynamic part: registering/deregistering pod IPs as pods scale or replace.
- Also useful for NLBs created outside K8s (e.g., shared NLB fronting multiple services where you don't want K8s to own the LB config).

---

## Further reading / watching

- **AWS LB Controller annotations reference**: [kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/ingress/annotations/) — bookmark this; it's the most-used reference page.
- **EKS Best Practices — Load Balancing**: [aws.github.io/aws-eks-best-practices/networking/loadbalancing/](https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/) — NLB vs ALB decision tree, target type guidance.
- **AWS blog — "Exposing Kubernetes applications, Part 3: NLB IP mode"**: search exact title. The target type `ip` walkthrough with diagrams.
- **TargetGroupBinding guide**: in the AWS LB Controller docs — underrated feature, worth a 10-minute read.

---

## The 4 dimensions (senior framing)

- **Tech**: NLB (L4, TCP/UDP) vs ALB (L7, HTTP). Target type `instance` (node → kube-proxy → pod, SNAT) vs `ip` (pod direct, VPC CNI required). Pod readiness gate ensures no traffic before LB health check passes. Drain timeout must be >= pod grace period. TargetGroupBinding for externally-managed LBs.
- **People**: developers need to understand why `remoteAddr` shows a node IP in `instance` mode — it trips them up in logging and rate-limiting code. Document the target type choice in your platform runbook. The readiness gate annotation must be applied to the namespace — remind developers it's not automatic, especially when they create new namespaces.
- **CI/CD**: LB creation happens on Ingress/Service apply — fast (seconds for target group updates, 1-2 minutes for full ALB creation). Ingress group changes affect all services in the group — treat shared ALB changes as a wider blast radius than single-service changes. Terraform owns ACM certs + IngressClassParams; GitOps owns Ingress YAML. Don't let them conflict.
- **Operations**: monitor target group unhealthy host count (`TargetResponseCodeCount` 5xx from the ALB). Alert on NLB/ALB unhealthy host count > 0 for > 5 minutes. Runbook for 503s: check LB access logs (S3 or CloudWatch) → check target group health in console → check pod readiness gate in `kubectl describe pod` → check drain timeout alignment.
