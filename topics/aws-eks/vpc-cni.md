# AWS VPC CNI — the deep-dive

> **Goal**: by the end you can answer **"You're hitting pod-density IP exhaustion on m5.large nodes. Walk through your options."** — naming the ENI/IP math, the warm-pool tuning knobs, prefix delegation mechanics, custom networking, security groups for pods, and when to switch CNI entirely.

> Start with the [simple version](./vpc-cni-simple.md) if you haven't — the parking lot analogy is the mental model.

---

## The senior framing — VPC CNI trades overlay complexity for IP exhaustion risk

Most CNIs (Flannel, Calico in VXLAN mode, Cilium in tunnel mode) use an overlay network. Pod IPs are virtual — typically a /24 or /16 per node from a pod CIDR range — encapsulated over the node's real IP. You can have thousands of pods per node without touching the VPC IP space.

AWS VPC CNI's design decision is the opposite: **every pod gets a real VPC secondary IP**. This gives you:
- Native VPC routing (no decap overhead, full security group and VPC flow log visibility per pod)
- Direct connectivity from any VPC resource (ALB, RDS, Lambda) to a pod IP without extra configuration

The cost: **you consume real VPC IPs**. VPCs have finite CIDR space. EC2 instances have a fixed ENI/IP ceiling per instance type. At scale, these two limits collide.

Understanding this trade-off is the core of every VPC CNI interview question.

---

## ENI / IP mechanics — what actually happens when a pod is scheduled

### The ENI warm pool

The `aws-node` DaemonSet (the VPC CNI plugin) runs on every node. It maintains a warm pool of pre-allocated ENIs and secondary IPs so that pod startup doesn't have to wait for an ENI attach (which takes ~10 seconds).

Lifecycle:
1. Node starts → `aws-node` attaches ENIs and assigns secondary IPs up to the warm pool target.
2. A pod is scheduled → `aws-node` assigns one pre-warmed IP to the pod's network namespace.
3. If the warm pool drops below target → `aws-node` attaches another ENI / allocates more IPs.
4. Pod terminates → IP returned to the warm pool.

### ENI / IP limits per instance type

| Instance type | Max ENIs | Max secondary IPs/ENI | Max pods (standard) | Max pods (prefix delegation) |
|---|---|---|---|---|
| t3.micro | 2 | 2 | 3 | 35 |
| t3.medium | 3 | 6 | 17 | 97 |
| m5.large | 3 | 10 | 29 | 479 |
| m5.xlarge | 4 | 15 | 58 | 958 |
| m5.2xlarge | 4 | 15 | 58 | 958 |
| m5.4xlarge | 8 | 30 | 234 | 3,838 |
| c5.large | 3 | 10 | 29 | 479 |
| r6i.4xlarge | 8 | 30 | 234 | 3,838 |

Formula (standard): `(max_ENIs × max_secondary_IPs_per_ENI) - 1` (subtract 1 for the primary node IP).

Note: EKS adds its own per-node pod limit via the `max-pods` kubelet flag, calculated from these AWS limits. You can see the per-node limit with `kubectl describe node | grep "max-pods"`.

---

## Warm pool tuning knobs

The `aws-node` DaemonSet reads these environment variables:

```bash
# How many free IPs to keep ready at all times
WARM_IP_TARGET=3          # default; keep 3 un-assigned IPs per node

# How many fully free ENIs to keep attached (in addition to WARM_IP_TARGET)
WARM_ENI_TARGET=1         # default; keep 1 fully-free ENI attached

# Floor: minimum IPs to always keep allocated regardless of current pod count
MINIMUM_IP_TARGET=0       # default: no floor
```

### WARM_IP_TARGET vs WARM_ENI_TARGET — which to tune and when

**High pod churn workloads** (batch jobs, CI runners that spin pods up and down frequently): increase `WARM_IP_TARGET` to match burst pod count. Without this, every pod start triggers a fresh IP allocation, adding 1-2 seconds of latency per pod.

**Low pod count nodes** (large instance type running a few heavy pods): set `WARM_ENI_TARGET=0` and `WARM_IP_TARGET=2`. Otherwise `aws-node` attaches multiple ENIs and holds IPs you'll never use — wasting VPC IP space.

**Burstable batch nodes** (Karpenter spot nodes that go from 0 to 50 pods in 10 seconds): set `WARM_IP_TARGET=30` or higher. The ENI attach time (~10s) is the bottleneck; if the warm pool is depleted, pods wait.

---

## Prefix delegation

Instead of allocating one IP per ENI slot, AWS allocates a `/28` prefix (16 IPs) per slot.

```bash
# Enable prefix delegation on the aws-node DaemonSet:
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1
```

With prefix delegation, `m5.large` (3 ENIs × 10 slots): instead of 30 secondary IPs, you get 30 `/28` prefixes = **480 IPs** (479 pods, subtracting the primary).

### The subnet fragmentation gotcha

Prefix delegation requires allocating contiguous `/28` (16-address) blocks from your subnet. A subnet that has had random allocations for years (spot instances launching and terminating, NAT gateway IPs, RDS instances) becomes fragmented — there may be no contiguous 16-address block left, even if there are 50 free IPs scattered around.

When this happens, `aws-node` falls back to individual IP allocation silently. You won't see an error; you just won't get the density improvement.

Fix: plan subnets for EKS nodes with prefix delegation from the start. Use a dedicated, large CIDR (e.g., `/22`) for node subnets. Keep node and non-node resources in separate subnets.

---

## Security groups for pods (SGP)

By default, all pods on a node share the node's security group(s). If you want a pod to have a different inbound/outbound SG — e.g., a payment-processing pod that needs access to an RDS instance that other pods must not reach — you'd normally need a separate node for that pod.

SGP changes this. It gives each pod its own ENI (via a feature called "ENI trunking"), with its own security group(s).

```yaml
# SecurityGroupPolicy CRD (installed by VPC CNI in SGP mode)
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: payment-service-sgp
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-processor
  securityGroups:
    groupIds:
      - sg-0abc1234  # payment-specific SG with RDS access
```

### How ENI trunking works

The node gets a "trunk" ENI. When a pod with an SGP is scheduled, VPC CNI attaches a "branch" ENI to the trunk and assigns the pod's specific SG to that branch ENI. The pod's traffic flows through the branch ENI, not the node's primary ENI.

**Instance type requirement**: ENI trunking requires instance types that support it (most Nitro-based types: `m5`, `c5`, `r5`, etc.). Not all instance types support it.

**Density impact**: branch ENIs count against the same per-instance ENI limit. If your node already has 3 ENIs attached, you can't add more branch ENIs without a larger instance type.

### When to use SGP vs NetworkPolicy

- **NetworkPolicy** (Calico, Cilium, or the built-in VPC CNI network policy): L3/L4 rules enforced in-cluster. Good for most workloads.
- **SGP**: enforced at the VPC level. Necessary when you need the security group to be auditable from the AWS console/CloudTrail, or when the target resource (RDS, ElastiCache) only accepts connections via SG rules.

Use SGP only when the compliance or isolation requirement is explicit. It adds operational complexity (branch ENI limits, trunk ENI management, SGP CRD).

---

## Custom networking

Custom networking routes pod IPs through a secondary subnet, separate from the node's primary subnet.

**Why you'd want this**:
- Primary subnet is running out of IPs but you can't resize it (it's a shared VPC).
- Regulatory requirement to put pod traffic on a specific subnet with specific route table rules.
- You want node-to-node traffic on one subnet and pod-to-external traffic on another.

```bash
# Enable custom networking in aws-node:
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# Create an ENIConfig per AZ pointing to the secondary subnet:
kubectl apply -f - <<EOF
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a  # must match AZ name
spec:
  subnet: subnet-0pod1a        # secondary subnet
  securityGroups:
    - sg-0abc1234
EOF
```

Nodes label themselves with the AZ (`topology.kubernetes.io/zone`), and `aws-node` finds the correct `ENIConfig` by matching the label.

Custom networking is a surgical tool. It doesn't fix the per-node ENI ceiling. It shifts *where* the IPs come from, not how many you can use per node.

---

## The density vs cost trade-off: VPC CNI vs overlay CNIs

| Dimension | VPC CNI (native IP) | Cilium / Calico overlay |
|---|---|---|
| Pod IP routing | Native VPC — no encap | Encapsulated (VXLAN or Wireguard) |
| Pod-to-pod latency | Lower (no decap) | Higher (~5-10% overhead for VXLAN) |
| Pod IP exhaustion | Yes — real VPC IPs | No — virtual pod CIDR |
| SG per pod | Yes (SGP) | No (must use NetworkPolicy) |
| VPC flow logs | Per-pod visibility | Node-level only |
| Complexity | Simple (AWS manages) | Higher (custom CNI to operate) |
| When to switch | When IP exhaustion can't be solved with prefix delegation or subnet expansion | When you need overlay density or advanced NetworkPolicy (L7, FQDN) |

The practical decision: **try prefix delegation first**. It solves 95% of IP exhaustion problems. Move to overlay only if you've exhausted prefix delegation and can't expand subnets — or if you need L7 NetworkPolicy (Cilium only).

---

## The interview answer in 60 seconds

> "First, confirm it's IP exhaustion: `kubectl describe pod <pending>` will show 'Insufficient free addresses in subnet' in the events. Then work through the options.
>
> Option 1 is prefix delegation — one env var change on `aws-node`, goes from 29 to ~479 pods on `m5.large`. Requires the subnet to have contiguous `/28` blocks available; fragmented subnets silently fall back to single-IP mode.
>
> Option 2 is instance type — move to a type with better ENI density. `m5.4xlarge` gives 234 pods standard vs 29 for `m5.large`. But you're still capped.
>
> Option 3 is custom networking — route pod IPs through a secondary subnet with more CIDR space. Doesn't fix the per-node ENI ceiling, just shifts where IPs come from.
>
> Option 4 is switching to an overlay CNI like Cilium. Pod IPs become virtual — no longer consuming VPC space. Trade-off: you lose native VPC routing, SG per-pod, and VPC flow log per-pod visibility.
>
> The senior answer: prefix delegation first (least disruption, biggest impact), right-size instance types for pod density, and design new clusters with dedicated node subnets using `/22` CIDRs with prefix delegation in mind from day one."

---

## Self-test drills

### 1. You're hitting pod-density IP exhaustion on m5.large nodes. Walk through your options.

**Reference answer:** (see the interview answer above, expanded) — confirm via `kubectl describe`, then: prefix delegation (fast, big impact, subnet caveat), instance type upsizing, custom networking, overlay CNI switch. Priority order matters in the answer.

### 2. What does WARM_IP_TARGET do, and when would you increase it?

**Reference answer:**
- `WARM_IP_TARGET` tells `aws-node` how many un-assigned secondary IPs to keep ready on the node at all times.
- Default is 3 — fine for most workloads.
- Increase it for high-churn workloads (CI runners, batch jobs) that burst from 0 to N pods quickly. If the warm pool is depleted, each pod start waits for `aws-node` to allocate a new IP (~1-2s) or even attach a new ENI (~10s).
- Setting it too high on nodes with low pod counts wastes VPC IPs — `aws-node` holds IPs that will never be used.

### 3. What is ENI trunking, and when does security groups for pods make sense over NetworkPolicy?

**Reference answer:**
- ENI trunking: a trunk ENI on the node acts as a multiplexer. Branch ENIs (each with their own SG) are attached to the trunk for pods that have a `SecurityGroupPolicy`.
- Use SGP when: (a) the downstream resource (RDS, ElastiCache) accepts connections only via SG rules; (b) compliance requires the security control to be visible from the AWS console / CloudTrail, not just K8s; (c) the isolation requirement is per-pod, not per-deployment.
- Use NetworkPolicy for everything else — it's simpler, doesn't consume ENI slots, and works across all instance types.

### 4. Why does prefix delegation silently fail on fragmented subnets?

**Reference answer:**
- Prefix delegation requests a contiguous `/28` (16-address) block from the subnet.
- A subnet that has had years of random single-IP allocations (spot instances, NAT gateways, RDS) has fragmented free space — there may be 50 free IPs, but no contiguous 16-address block.
- When no `/28` is available, `aws-node` silently falls back to single-IP allocation. No error, no metric increment. You don't get the density benefit.
- Detection: compare actual IPs allocated per ENI vs expected 16. Or check CloudWatch metrics for `aws-node` allocation failures.
- Prevention: dedicated node subnets with clean `/22` CIDRs, planned from day one.

---

## Further reading / watching

- **VPC CNI GitHub**: [github.com/aws/amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s) — `docs/prefix-and-ip-target.md` for warm pool tuning; `docs/custom-networking.md` for ENIConfig.
- **EKS Best Practices — Networking**: [aws.github.io/aws-eks-best-practices/networking/vpc-cni/](https://aws.github.io/aws-eks-best-practices/networking/vpc-cni/) — the most complete single-page reference with diagrams.
- **EC2 ENI limits**: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) — the authoritative table. Bookmark it.
- **AWS blog — "Increase the amount of available IP addresses"**: search exact title. The prefix delegation announcement; explains the math clearly.
- **Cilium vs VPC CNI comparison**: Cilium docs at [docs.cilium.io/en/stable/network/kubernetes/aws](https://docs.cilium.io/en/stable/network/kubernetes/aws/) — the overlay vs native routing modes.

---

## The 4 dimensions (senior framing)

- **Tech**: ENI warm pool is the core mechanism — understand WARM_IP_TARGET, WARM_ENI_TARGET, MINIMUM_IP_TARGET. Prefix delegation = 16× density; requires contiguous /28 in subnet. SGP = per-pod security group via ENI trunking. Custom networking = separate subnet for pods. Overlay as last resort.
- **People**: developers rarely hit this directly — it surfaces as pods stuck Pending. On-call needs to know the `kubectl describe pod` check and the difference between IP exhaustion vs CPU/memory exhaustion. Subnet CIDR planning is a platform team decision that devs never see — but affects them badly when it goes wrong. Document it in an ADR when you pick the subnet strategy.
- **CI/CD**: `aws-node` is a DaemonSet deployed as a managed EKS addon. Upgrades should go through the addon upgrade path (not manual kubectl). Test prefix delegation changes in a non-production cluster first — enabling it requires node recycling for existing nodes.
- **Operations**: monitor `awscni_assigned_ip_addresses` and `awscni_total_ip_addresses` via CloudWatch or the VPC CNI Prometheus endpoint. Alert when `(total - assigned) / total < 10%` (warm pool nearly depleted). Runbook for pod-Pending: check events → check CIDR exhaustion in subnet (`aws ec2 describe-subnets --subnet-ids <id>`) → check `aws-node` DaemonSet logs for allocation errors.
