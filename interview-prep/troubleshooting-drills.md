# Troubleshooting Drills — 5 Scenarios End-to-End

> Practice each out loud. Structure of your answer matters as much as content — interviewers listen for systematic thinking.

For each drill:
- **Scenario** — how they'll frame it
- **Right shape of the answer** — the systematic approach
- **Step-by-step walkthrough** — what to actually say
- **Senior signals** — moves that earn points
- **Trap to avoid** — what kills the answer
- **60-second TL;DR** — rehearse this verbatim

---

## Drill 1 — Latency spike at the API edge

### Scenario
> "At 14:00 UTC, p99 latency on the Wallet API jumped from 80ms to 2.5 seconds. Error rate is normal — no 5xx spike. Mobile users are complaining. Walk me through how you'd debug this."

### Right shape
Latency without errors = **slow path, not a broken path**. Localize the slow hop by walking the stack from edge to backend, then look at infrastructure (DB, dependencies, network), then look at change events around 14:00.

### Step-by-step walkthrough

**Step 1: Confirm scope and pattern.** Look at the latency metric broken down — is it all endpoints, or just one? Is it all consumers/plans, or one partner? In Gravitee analytics group by path and by plan. Path-specific = downstream of routing. Plan-specific = might be a policy issue on that plan.

**Step 2: Walk the request path from edge to backend, layer by layer.**
- **CloudFront/WAF metrics** — any latency contribution? CloudFront has its own latency distribution.
- **NLB metrics** — `TargetResponseTime` in CloudWatch. If NLB sees fast, slowness is downstream.
- **Gravitee gateway** — `gateway_response_time` minus `api_response_time` = time spent in policies. Recently changed policies, especially custom Groovy/JS or `transform-body`.
- **Istio ingress gateway and mesh** — `istio_request_duration_milliseconds`. Compare source side and destination side metrics — if source Envoy sees more latency than destination Envoy reports, latency is between them.
- **Application metrics** — RED metrics from the wallet service.
- **Backend dependencies** — DB query latency, Redis latency, downstream blockchain RPC calls.

**Step 3: Look at the change log.** What was deployed or changed around 14:00? ArgoCD sync history, Terraform plans/applies, Gravitee API publishes, AWS console event history. **Most latency spikes correlate with a recent change.**

**Step 4: Upstream dependencies.** For a wallet API, likely culprits:
- Blockchain RPC provider degraded (Infura, Alchemy)
- DB (RDS, MongoDB) under unexpected load
- Cache miss rate spike (Redis cold or evicted)
- Cross-region call where a region is degraded

**Step 5: Distributed tracing.** Find a slow trace, look at the span breakdown. Slow span tells you the slow component.

**Step 6: Slice by infrastructure dimensions.** Node-specific? AZ-specific? Pod-specific? `topk(10, histogram_quantile(0.99, ...) by (pod))` — if one pod is slow, noisy-neighbor or throttled EBS.

### Senior signals
- Not jumping to a fix. Walking the stack systematically.
- "Compare source-side and destination-side metrics" — finds network/sidecar vs app.
- Calling out the change log — most outages correlate with a change.
- AZ/pod-level slicing for noisy-neighbor.
- Naming likely upstream culprits for a crypto company (blockchain RPCs, Redis cache).

### Trap to avoid
- "I'd restart the pods." That's not debugging, it's hoping.
- Going straight to "it's the app" without checking the path.

### 60-second TL;DR
> "Latency without errors means a slow path, not a broken one. I'd walk the request layer by layer — CloudFront, NLB, Gravitee gateway, Istio sidecar, app, dependencies — looking at each layer's latency contribution. Most useful technique is comparing source-side and destination-side Envoy metrics — if the source sees more latency than the destination reports, it's mesh or network. Then I'd check the change log around 14:00 — most spikes correlate with a recent deploy or config change. For a wallet API specifically, I'd expect the usual culprits: a degraded blockchain RPC provider, Redis cache miss spike, or DB query getting slow due to data volume growth. Distributed tracing closes it out by showing me the slow span."

---

## Drill 2 — ArgoCD application stuck OutOfSync

### Scenario
> "An ArgoCD application has been stuck in OutOfSync state for 30 minutes. Git matches what we expect to be deployed. Manual sync doesn't fix it. What's going on?"

### Right shape
OutOfSync means ArgoCD thinks cluster state differs from Git. Either: (a) something on the cluster IS different and you missed it, (b) ArgoCD is comparing wrong things (ignored fields, generated annotations), (c) something is constantly reverting the change.

### Step-by-step walkthrough

**Step 1: See what's actually different.** ArgoCD UI or `argocd app diff <app>`. The diff tells you exactly which resource and field differs.

**Step 2: Categorize the diff.**
- **Field added by something else** — a webhook (Istio sidecar injection adds annotations, Kyverno adds labels, AWS LB Controller adds finalizers). Normal; configure `ignoreDifferences`.
- **Field reset by a controller** — HPA setting replicas on a Deployment. Don't manage replicas in Git for HPA-managed Deployments; mark replicas as ignored.
- **Field added in cluster missing from Git** — actual drift. Someone kubectl-edited it (find via audit log) or another controller created it.

**Step 3: Why doesn't sync fix it?**
- **Mutating admission webhook changes the resource after ArgoCD applies.** ArgoCD applies → webhook mutates → ArgoCD sees mutation as drift → loop.
- **Server-side apply field ownership conflicts.** Another controller "owns" the field.
- **Finalizer blocking changes.** `kubectl describe` shows finalizers.
- **Auto-sync disabled.**
- **Sync policy has `selfHeal: false` or wave dependency unresolved.**

**Step 4: Common-cause checklist for Istio + ArgoCD.**
- Istio injects sidecar containers into Pod spec → ArgoCD sees pods as drifted.
- Istio mutates Service annotations.
- Cluster Autoscaler / Karpenter changes node selectors.

**Step 5: Fix patterns.**
```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers: [/spec/replicas]
  - group: ""
    kind: Service
    jqPathExpressions:
    - '.metadata.annotations["service.beta.kubernetes.io/aws-load-balancer-controller-version"]'
```
- Set `syncPolicy.syncOptions: [ServerSideApply=true]` for field ownership conflicts.
- Use `Replace=true` carefully — hammer, can lose data on stateful resources.

### Senior signals
- Knowing **mutating webhooks** are the most common cause.
- Specific examples: HPA replicas, Istio sidecar injection, AWS LB Controller annotations.
- Distinguishing **noise** (webhook adds) from **real drift** (kubectl-edited).
- Mentioning **field ownership** with Server-Side Apply.
- Pointing at the **audit log** to find who kubectl-edited.

### Trap to avoid
- Recommending `--force` or `Replace=true` as the first move. "Fixes" the symptom but masks the cause and can lose data on stateful resources.

### 60-second TL;DR
> "OutOfSync with manual sync not fixing it usually means something is mutating the resource after ArgoCD applies. Three common causes: a webhook (Istio sidecar, AWS LB Controller, Kyverno) adding fields ArgoCD then sees as drift; a controller like HPA reverting a field ArgoCD manages (replicas); or field ownership conflicts with Server-Side Apply. First move is `argocd app diff` to see exactly what's different. If it's a webhook mutation, add `ignoreDifferences` for those specific fields. If it's an HPA owning replicas, ignore /spec/replicas. If it's field ownership, enable Server-Side Apply in syncOptions. Never reach for `--force` or `Replace=true` as the first move — they mask the cause and can lose data."

---

## Drill 3 — Mass UH 503s across multiple services

### Scenario
> "We're seeing 503 with response flag UH across five different services simultaneously. Started 10 minutes ago. Walk me through your response."

### Right shape
UH = no healthy upstream. Across many services at once = either (a) shared dependency is down and outlier detection is cascading, (b) control plane is degraded so endpoint discovery is stale, or (c) network partition. Triage in that order.

### Step-by-step walkthrough

**Step 1: Confirm the pattern.** `istio_requests_total{response_flags="UH"}` grouped by destination_workload. Really many services, or five services that all depend on one common downstream?

**Step 2: Most likely cause — cascading outlier-detection.** If those five services share a dependency — a DB, Redis, an external RPC provider — and that dependency is sick:
- Upstream pods start returning 5xx because their downstream is broken
- Outlier detection ejects them from the LB pool
- Remaining pods get amplified load, also fail
- All endpoints ejected → callers see UH

First move: `istioctl proxy-config endpoints` on a representative caller — are endpoints actually missing/unhealthy? Then `curl localhost:15000/stats | grep outlier_detection.ejections_active` on the caller sidecar. High number → cascading ejection.

**Step 3: Find the root.** Which downstream do these five services all call? Check that downstream's own metrics and logs. Common roots: RDS connection exhaustion, Redis OOM, third-party API (Infura, Stripe) degraded, internal authn service (Gravitee AM) saturated.

**Step 4: Mitigate the cascade.** While debugging:
- Relax outlier detection temporarily — bump `minHealthPercent` so we never eject everyone.
- Reduce upstream retry attempts to slow the storm.
- Scale up affected services horizontally if root cause is load.

**Step 5: Rule out control plane.** If endpoints look fine in `istioctl proxy-config endpoints`, issue isn't data plane. `istioctl proxy-status` — are sidecars in sync? Many STALE = istiod overloaded or partitioned. Sidecars have stale endpoint lists, traffic going to terminated pods.

**Step 6: Bypass test.** Debug pod without sidecar injection, curl destinations directly. Works without mesh → mesh-layer issue. Fails both ways → infrastructure layer (network, destination itself).

### Senior signals
- Naming the cascading ejection pattern by name.
- Mentioning `minHealthPercent` as the emergency mitigation lever.
- Separating control-plane from data-plane suspicion with `proxy-status`.
- Bypass test (non-injected debug pod) as sanity check.
- Naming likely root causes specific to the company (Infura, Redis, RDS, Gravitee AM).

### Trap to avoid
- Looking at affected services in isolation, missing the shared dependency.
- Restarting pods of the affected services — they're not broken, their dependency is.

### 60-second TL;DR
> "UH across many services at once almost always means cascading outlier detection. The pattern: a shared dependency degrades, upstream pods start returning 5xx, outlier detection ejects them from the LB pool, surviving pods get crushed, everyone ejected, callers see UH. First move: `istioctl proxy-config endpoints` and the outlier-detection stats on a caller sidecar. High ejection count confirms the cascade. Then find the shared root — usually a DB, Redis, or third-party API the affected services all call. While debugging, relax `minHealthPercent` to break the cascade. Also rule out control plane via `istioctl proxy-status` — if sidecars show STALE, it's stale endpoint discovery, not actual upstream failure. Final sanity check: bypass test with a non-injected pod."

---

## Drill 4 — Intermittent DNS resolution failures in cluster

### Scenario
> "Engineers are reporting intermittent DNS resolution failures — sometimes `nslookup wallet-svc` works, sometimes it times out. Affects different pods at different times. Walk through."

### Right shape
DNS issues are usually: (a) CoreDNS underscaled/saturated, (b) conntrack table full on nodes (the classic), (c) ndots/search-domains causing extra queries, (d) NodeLocal DNS Cache misbehavior. Intermittent + pod-random nature points strongly to conntrack or CoreDNS capacity.

### Step-by-step walkthrough

**Step 1: CoreDNS health first.**
- `kubectl -n kube-system top pods -l k8s-app=kube-dns` — CPU/memory under limit?
- CoreDNS metrics: `coredns_dns_responses_total` rate, `coredns_dns_request_duration_seconds`. High latency or error rate?
- `kubectl -n kube-system logs deploy/coredns` — SERVFAIL, "i/o timeout" upstream errors?
- How many replicas? Default is 2 — at scale you need more (`cluster-proportional-autoscaler`).

**Step 2: The classic conntrack culprit.** DNS uses UDP. Linux UDP `conntrack` entries default to 30-second timeout. Under high QPS, conntrack table fills, new DNS queries can't be tracked, kernel drops them. Intermittent timeouts.
- `cat /proc/sys/net/netfilter/nf_conntrack_count` vs `nf_conntrack_max`.
- `dmesg | grep nf_conntrack` for 'table full'.
- Fix: bump `nf_conntrack_max` on nodes, or deploy **NodeLocal DNS Cache** which uses TCP and bypasses conntrack.

**Step 3: ndots and search domains.** Kubernetes injects default `ndots: 5` and several search domains. Every short-name lookup tries `wallet-svc.payments.svc.cluster.local`, then `wallet-svc.svc.cluster.local`, then `wallet-svc.cluster.local`, then external `wallet-svc`. Most are NXDOMAIN. 4-5× the queries needed.
- Fix: FQDNs with trailing dot, or set `dnsConfig.options: [{name: ndots, value: '2'}]` on Pod spec.

**Step 4: NodeLocal DNS Cache.** If deployed, check its health — if misbehaving, queries fail. `kubectl -n kube-system get pods -l k8s-app=node-local-dns`. Listens on link-local IP (`169.254.20.10`); broken iptables = timeouts.

**Step 5: External resolution.** If failures are external names, issue is CoreDNS forwarding upstream — VPC DNS, resolver IP CoreDNS forwards to. Check Corefile `forward` plugin config.

**Step 6: Per-pod investigation.** Debug pod, repeatedly `dig wallet-svc.payments.svc.cluster.local`, measure latency. `tcpdump -i any port 53 -w /tmp/dns.pcap` to see what's sent/received.

### Senior signals
- Naming **conntrack** as prime suspect for intermittent UDP DNS — very common, easy to miss.
- Knowing **NodeLocal DNS Cache** exists and what it does (TCP path, bypass conntrack).
- Mentioning **ndots** as a query-rate multiplier.
- Distinguishing in-cluster vs external resolution paths.

### Trap to avoid
- "I'd just scale CoreDNS" — might not be the issue.
- Not knowing about conntrack at all — UDP networking gap is a senior-level red flag.

### 60-second TL;DR
> "Intermittent DNS failures, especially affecting random pods, is almost always one of three things: CoreDNS underscaled, conntrack table full on nodes, or ndots causing excessive query amplification. Order of investigation: CoreDNS health and metrics first, then conntrack — `nf_conntrack_count` vs `nf_conntrack_max` on affected nodes, and `dmesg` for 'table full'. UDP DNS through conntrack with a 30-second timeout fills the table fast at scale. Fix: NodeLocal DNS Cache, which uses TCP and bypasses conntrack for the hot path. Also worth checking ndots — Kubernetes' default of 5 means every short-name lookup generates 4-5 NXDOMAIN queries before the right one. Setting `ndots: 2` or using FQDNs cuts the volume dramatically."

---

## Drill 5 — Unexpected EKS cost spike

### Scenario
> "AWS bill went up 40% last month. Most of the spike is on EKS. How do you find what's driving it?"

### Right shape
Cloud cost is multi-dimensional: compute (EC2), networking (NAT, cross-AZ, LB), storage (EBS, S3), data services (RDS, ElastiCache, OpenSearch). On EKS: EC2 + cross-AZ data transfer + EBS + load balancers. Approach: Cost Explorer to identify which dimension grew, then drill into K8s-level attribution.

### Step-by-step walkthrough

**Step 1: AWS-level attribution first.** Cost Explorer, group by **service** for which AWS service drove the spike. Then by **usage type**. EKS-related costs: EC2 (`BoxUsage:*`), EBS (`EBS:VolumeUsage*`), NAT (`NatGateway-Bytes`), LBs (`LoadBalancerUsage`), inter-AZ data (`DataTransfer-Regional-Bytes`).

**Step 2: Map to K8s-level cause.**
- **EC2 BoxUsage up** → more nodes. Why? HPA scaling, or Karpenter/CA provisioning for pending pods. Check Karpenter metrics + node count history.
- **EBS up** → PVCs growing, many small PVCs from StatefulSets, or gp3 IOPS provisioned higher than default.
- **NAT bytes up** → pods making outbound traffic to internet, or hitting public S3/ECR instead of VPC endpoints. **VPC endpoints for S3/ECR/DynamoDB save real money** at scale.
- **Cross-AZ data** → mesh traffic crossing AZs that should be locality-aware. Istio locality LB enabled?
- **Load balancers** → someone created LoadBalancer Services in a loop, or multiple ALBs where one shared would do.

**Step 3: K8s-level attribution.** Use **OpenCost** or **Kubecost** to attribute cluster cost back to namespaces, deployments, labels. Look for:
- Namespace whose cost jumped (new service, broken HPA).
- Pod with disproportionate resource requests (overcommitting reserves nodes).
- Infinite loop in a CronJob.

**Step 4: Common specific causes.**
- **Karpenter or Cluster Autoscaler running away** — pods can't schedule due to `nodeSelector` typo, autoscaler provisions idle nodes.
- **HPA misconfigured** — scaling on metric that always trends up (queue depth never drains because consumer is broken).
- **StatefulSet creating unbounded PVCs** — Elasticsearch scaling up without retention policy.
- **Cross-AZ chatter from a new microservice** — pods not locality-aware, traffic ping-ponging.
- **Log volume to CloudWatch / S3** — service started logging DEBUG in prod.
- **Datadog/Observability spend** — 100% sampling = ingest bills explode.

**Step 5: Stop the bleeding.** While investigating:
- Right-size requests on suspect workloads.
- Enable Spot on non-prod node groups.
- Enable cluster autoscaler scale-down (sometimes disabled inadvertently).
- VPC endpoints for S3/ECR if NAT bytes are the issue.

**Step 6: Long-term controls.**
- Budget alerts at 50%, 80%, 100%.
- Resource quotas per namespace.
- LimitRanges with sensible defaults.
- OpenCost as a permanent dashboard.

### Senior signals
- Going **AWS-level first**, K8s-level second.
- Naming **OpenCost / Kubecost**.
- Mentioning **VPC endpoints** for NAT savings — easy money, often missed.
- Mentioning **cross-AZ chatter** as a mesh-locality issue.
- Closing with **proactive controls** (budgets, quotas, LimitRanges).

### Trap to avoid
- Jumping to "we should use Spot" without knowing what's driving cost. Spot won't help if cost is networking, not compute.
- Not knowing about cost-attribution tools at K8s layer.

### 60-second TL;DR
> "Cost work is two-layered: AWS attribution first to find which service drove the spike, then K8s attribution to map it to a workload. In Cost Explorer, group by service then usage type — EKS spend lands in EC2 BoxUsage, EBS, NAT, LBs, and cross-AZ data transfer. Each AWS line maps to a K8s cause: EC2 up means more nodes (HPA or Karpenter running), NAT bytes up means pods hitting the internet through NAT instead of VPC endpoints, cross-AZ up means non-locality-aware traffic. For K8s-level attribution I use OpenCost or Kubecost to break cost down by namespace and deployment. Common specific causes: a misconfigured HPA scaling forever, a CronJob in an infinite loop, a StatefulSet creating unbounded PVCs, or observability ingest exploding from DEBUG-level logging. Mitigations are immediate (VPC endpoints for S3/ECR, right-sizing requests, Spot on non-prod). Controls are long-term (budgets, ResourceQuotas, LimitRanges, OpenCost dashboard)."

---

## How to use these drills

1. **Read each scenario once.**
2. **Cover the answer.** Walk through it out loud yourself.
3. **Compare to the walkthrough.** Where did your thinking diverge? What did you forget?
4. **Memorize the 60-second TL;DR.** That's the answer you give in the room.

The biggest senior signal across all 5: **you go top-down, you don't guess, you name the systematic approach before the specific tools.**
