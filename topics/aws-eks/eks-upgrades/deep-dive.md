# EKS upgrades — the deep-dive

> **Goal**: by the end you can answer **"Walk me through a zero-downtime EKS minor version upgrade across the control plane and 100 nodes."** — naming the pre-flight checks (deprecated APIs, addon matrix, PDB audit), the control plane upgrade mechanics, the exact addon upgrade order, the kubelet skew policy, and the in-place rolling vs blue-green node strategy trade-offs.

> Start with the [simple version](./simple.md) if you haven't — the highway resurfacing analogy is the mental model.

---

## The senior framing — EKS upgrades are mostly about pre-flight, not the upgrade itself

The EKS control plane upgrade is a single API call. AWS does the work in 10-15 minutes. The upgrade itself almost never fails.

What causes incidents:
1. Deprecated API objects in Helm charts or GitOps manifests that become `removed` in the new version
2. Addons left on the old version — they call removed APIs and silently fail
3. PodDisruptionBudgets that block node drains indefinitely
4. Workloads that don't gracefully shut down (missing `preStop` hooks, tight `terminationGracePeriodSeconds`)
5. Insufficient node capacity during the rolling node upgrade (drained pods have nowhere to go)

The upgrade itself is the easy part. The pre-flight is where you earn your money.

---

## Phase 1 — Pre-flight

### Deprecated API check

Kubernetes removes deprecated APIs in minor versions. A cluster upgrade from 1.27 to 1.28 removes any beta APIs that were deprecated in 1.26 or earlier.

The practical problem: Helm charts installed 18 months ago may still be using `policy/v1beta1` PodDisruptionBudget or `batch/v1beta1` CronJob. The EKS upgrade will refuse if any objects in etcd use removed APIs (in some cases it will proceed and then break those objects).

**How to check:**

Option 1 — Prometheus metric (if you have it):
```bash
kubectl get --raw /metrics | grep apiserver_requested_deprecated_resources
```
This shows a count of API calls to deprecated endpoints over the last scrape window. Not perfect (only catches *active* API calls, not stored objects) but useful.

Option 2 — `pluto` (Fairwinds, recommended):
```bash
pluto detect-all-in-cluster --target-versions k8s=v1.28
pluto detect-helm --target-versions k8s=v1.28
```
Scans all live resources and Helm release manifests. Reports which objects use APIs removed in 1.28. The output is actionable: upgrade the Helm chart, or update the manifest.

Option 3 — `kubectl convert` (built-in, for live objects):
```bash
kubectl get cronjob --all-namespaces -o yaml | kubectl convert --local -f - --output-version batch/v1
```
Converts stored objects to the target API version, showing you what would break.

### Addon compatibility matrix

Before the control plane upgrade, check which addon versions are compatible with the target K8s version. The matrix is at [docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) — look for "Amazon EKS add-on versions and compatible Kubernetes versions."

Key addons and why they matter:

| Addon | Why it needs upgrading |
|---|---|
| VPC CNI (`aws-node`) | Calls K8s API for pod networking; deprecated APIs removed in new version break IP allocation |
| CoreDNS | Calls K8s API for service discovery; API version changes can break queries |
| kube-proxy | Enforces service routing; must match the control plane version for correct iptables/ipvs rules |
| EBS CSI Driver | Calls K8s API for PVC management; breaking API changes cause volumes to not attach |

Check the matrix before the upgrade. Pin the target addon versions in your Terraform or CI pipeline.

### PodDisruptionBudget audit

Run before the node upgrade phase:
```bash
kubectl get pdb --all-namespaces -o json | jq '
  .items[] |
  select(
    (.spec.minAvailable == "100%") or
    (.spec.maxUnavailable == 0) or
    (.spec.maxUnavailable == "0%")
  ) |
  "\(.metadata.namespace)/\(.metadata.name): minAvailable=\(.spec.minAvailable // "n/a"), maxUnavailable=\(.spec.maxUnavailable // "n/a")"
'
```

Any PDB with `minAvailable: 100%` or `maxUnavailable: 0` on a deployment with 1 replica will block node drain forever. The `kubectl drain` command will time out or hang indefinitely.

Fix: temporarily set `maxUnavailable: 1` during the upgrade, or scale the deployment to 2+ replicas so the PDB allows 1 disruption.

### Capacity check

Managed node group rolling upgrade terminates nodes one batch at a time. You need spare capacity to absorb the drained pods.

Formula: `spare_capacity_needed = pods_per_node × nodes_in_batch`. For 100 nodes with 50 pods each, draining 20% (20 nodes) at once means 1,000 pods need to reschedule onto 80 nodes. Check that the remaining 80 nodes have headroom.

---

## Phase 2 — Control plane upgrade

```bash
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.28 \
  --region us-east-1
```

Or via Terraform: change the `version` attribute in `aws_eks_cluster` and apply.

What AWS does during the control plane upgrade:
1. Launches new control plane nodes (EC2 instances running the API server, scheduler, controller manager) in AWS's managed infrastructure.
2. Takes etcd snapshots.
3. Migrates the API server to the new version, validates etcd state.
4. Promotes new control plane nodes.
5. Updates the kubeconfig endpoint certificate if needed.

Duration: 10-15 minutes typically. Up to 25-30 minutes for large clusters or during AWS regional capacity pressure.

What you observe:
- The EKS console shows "UPDATING" status.
- The Kubernetes API server may return brief errors (5xx) during the handoff — typically a few seconds. In-flight connections time out and retry.
- `kubectl get nodes` keeps working (reads from etcd via the API server).
- Running pods are completely unaffected — kubelet talks to the API server but can run pods without it temporarily.

What to monitor during the upgrade:
- API server availability: `kubectl version` in a loop or a Prometheus alert on `kube_apiserver_up == 0`.
- Cluster autoscaler / Karpenter — may have a brief reconciliation delay after the API server handoff.
- etcd backup — trigger a manual backup before starting (or verify the automated backup ran within the last hour).

---

## Phase 3 — Addon upgrades

Immediately after the control plane upgrade completes, before touching nodes.

```bash
# Check current and available addon versions:
aws eks describe-addon-versions \
  --kubernetes-version 1.28 \
  --addon-name vpc-cni \
  --query 'addons[].addonVersions[].addonVersion'

# Upgrade:
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.1-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

**Order matters within addons**: upgrade VPC CNI first (networking depends on it), then kube-proxy (service routing), then CoreDNS (DNS), then storage addons (EBS/EFS CSI). In practice the order only matters if one addon is truly critical to another's startup — most environments can upgrade them in parallel.

**The `--resolve-conflicts OVERWRITE` flag**: if you've manually edited an addon's DaemonSet or ConfigMap, `OVERWRITE` will replace your changes with the default addon config. Use `PRESERVE` if you have custom patches — but then you must manually reconcile conflicts.

**Managing addons in Terraform** (recommended):

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "vpc-cni"
  addon_version            = "v1.18.1-eksbuild.1"
  resolve_conflicts_on_update = "OVERWRITE"
}
```

Use Renovate or Dependabot to auto-open PRs when new addon versions are published. Review and merge weekly, not at upgrade time.

---

## Phase 4 — Node upgrades

### Strategy 1: Managed node group in-place rolling

AWS replaces nodes in rolling batches. Configuration:

```hcl
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.subnet_ids
  version         = "1.28"

  update_config {
    max_unavailable_percentage = 20  # drain 20% of nodes at a time
    # OR: max_unavailable = 3         # drain 3 nodes at a time (absolute)
  }
}
```

What AWS does per batch:
1. Cordons the node (no new pods scheduled).
2. Drains the node (evicts all non-DaemonSet pods, respects PDB).
3. Terminates the EC2 instance.
4. Launches a replacement with the new AMI and kubelet version.
5. Node joins cluster, becomes Ready.
6. Next batch starts.

**max_unavailable_percentage** controls speed vs disruption. For 100 nodes:
- 10%: 10 nodes at a time → ~50 min (5 min per batch)
- 20%: 20 nodes at a time → ~25 min
- 33%: 33 nodes at a time → ~15 min, more disruptive

For production at business hours, 10-20% is conservative. During maintenance windows, 30-33% is acceptable if you have spare capacity.

### Strategy 2: Blue-green node group

Create a new node group on the new K8s version. Migrate pods. Delete the old group.

```bash
# 1. Create new node group (1.28) — aws_eks_node_group in Terraform or eksctl
# 2. Taint old nodes to prevent new pod scheduling:
kubectl taint nodes -l eks.amazonaws.com/nodegroup=old-group \
  upgrade=true:NoSchedule

# 3. Cordon old nodes:
kubectl cordon -l eks.amazonaws.com/nodegroup=old-group

# 4. Drain old nodes batch by batch:
for node in $(kubectl get nodes -l eks.amazonaws.com/nodegroup=old-group -o name); do
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --timeout=5m
done

# 5. Verify all pods are on new nodes:
kubectl get pods --all-namespaces -o wide | grep old-group  # should be empty

# 6. Delete old node group (Terraform destroy or eksctl delete nodegroup)
```

**When to use blue-green over in-place rolling:**
- Stateful workloads (databases, Kafka) where you want to control the exact drain sequence.
- Large clusters where in-place rolling is too slow (blue-green lets you drain faster because you have double capacity).
- When you want a quick rollback path — if the new node group has issues, the old one is still there.
- For Karpenter-managed nodes: Karpenter doesn't use managed node groups; create a new NodePool pointing to the new AMI and let Karpenter drain old nodes via its disruption mechanism.

---

## The kubelet skew policy

Kubernetes supports kubelet versions up to N-2 behind the control plane. This means:

| Control plane | Allowed kubelet versions |
|---|---|
| 1.28 | 1.28, 1.27, 1.26 |
| 1.28 | NOT 1.25 or earlier |

This policy exists so you can:
- Run a cluster with mixed kubelet versions during a rolling upgrade.
- Have some nodes on 1.26 while the control plane is on 1.28.
- Upgrade nodes over multiple maintenance windows without urgency.

**But**: kubelet does NOT support being *ahead* of the control plane. A 1.29 kubelet talking to a 1.28 control plane is unsupported and will likely misbehave. This is why you must upgrade the control plane before nodes.

Check current kubelet versions:
```bash
kubectl get nodes -o wide  # shows KERNEL-VERSION; for kubelet: kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'
```

---

## Putting it all together — the full upgrade checklist

```
1. Pre-flight (T-1 week):
   [ ] Run `pluto detect-all-in-cluster` → fix or update deprecated API objects
   [ ] Check addon compatibility matrix → pin target addon versions
   [ ] Audit PDBs → fix any that would block drain
   [ ] Check cluster capacity (nodes × pods headroom for drain)
   [ ] Verify etcd backup is recent

2. Upgrade day:
   [ ] Announce to teams (API blip possible)
   [ ] aws eks update-cluster-version → wait for ACTIVE
   [ ] Monitor API server during upgrade
   [ ] Update EKS addons (VPC CNI, kube-proxy, CoreDNS, EBS CSI)
   [ ] Verify addons are healthy (kubectl get ds -n kube-system, kubectl get pods -n kube-system)
   [ ] Update node group version → rolling drain begins
   [ ] Monitor drain progress; watch for stuck drains (PDB blocks)
   [ ] Verify all nodes on new version: kubectl get nodes

3. Post-upgrade:
   [ ] Run a smoke test suite
   [ ] Check SLO dashboards (error rate, latency) — no regression
   [ ] Update Terraform state to reflect new versions
   [ ] Update your upgrade runbook with any surprises
```

---

## The interview answer in 60 seconds

> "I'd run it in four phases.
>
> Pre-flight: run `pluto` to find deprecated APIs in Helm charts and live objects. Check the EKS addon compatibility matrix for VPC CNI, CoreDNS, and kube-proxy — pin the target versions. Audit PDBs for anything with `minAvailable: 100%` on single-replica deployments — those will block node drains indefinitely.
>
> Control plane: one API call, AWS handles it in 10-15 minutes. API server may blip briefly. Monitor API availability and Karpenter reconciliation.
>
> Addons: immediately after control plane completes. Upgrade VPC CNI, kube-proxy, CoreDNS via `aws eks update-addon`. Don't skip this — old addons calling removed APIs is the most common silent failure mode post-upgrade.
>
> Nodes: managed node group rolling update with `max_unavailable_percentage: 20` for 100 nodes. AWS drains 20 at a time, replaces with new AMI. Watch for stuck drains; any PDB that blocks drain halts the whole batch. I'd consider blue-green if stateful workloads make sequential draining risky.
>
> The kubelet skew policy gives us N-2 headroom — nodes can be 2 minor versions behind the control plane — so we can spread the node upgrade over multiple windows if needed."

---

## Self-test drills

### 1. Walk me through a zero-downtime EKS minor version upgrade across the control plane and 100 nodes.

**Reference answer:** Four phases — pre-flight (deprecated APIs, addon matrix, PDB audit, capacity check), control plane upgrade (one API call, 10-15 min, AWS-managed), addon upgrades (VPC CNI → kube-proxy → CoreDNS → storage), node rolling upgrade (managed node group, 20% batches, watch for PDB-blocked drains). Order is mandatory: CP → addons → nodes.

### 2. What checks do you run before an EKS control plane upgrade?

**Reference answer:**
- `pluto detect-all-in-cluster --target-versions k8s=v1.28` — finds deprecated API objects.
- Check addon compatibility matrix — which VPC CNI / CoreDNS / kube-proxy versions work with the target K8s version.
- PDB audit — any `minAvailable: 100%` or `maxUnavailable: 0` on low-replica deployments will block drains.
- Capacity check — ensure enough headroom for the drain batch size.
- Etcd backup — verify a recent snapshot exists.

### 3. Why do addons need to be upgraded, and what happens if you forget?

**Reference answer:**
- Addons like VPC CNI and CoreDNS call the Kubernetes API. If an API version they depend on was removed in the new K8s version, calls fail silently or with errors.
- VPC CNI failing: pod IP allocation breaks → new pods get stuck Pending with IP allocation errors.
- kube-proxy on old version: iptables rules may be inconsistent with new K8s service objects → traffic drops or misroutes.
- CoreDNS on old version: DNS resolution may fail for new service patterns.
- Symptoms appear gradually, not immediately — makes post-upgrade debugging painful. Always upgrade addons right after the control plane.

### 4. When would you choose blue-green node upgrade over in-place rolling?

**Reference answer:**
- Stateful workloads (databases, Kafka): you want to control exactly which nodes drain when, in what sequence.
- Large clusters (>200 nodes) where in-place rolling is too slow for your maintenance window.
- When you want a quick rollback: old node group still exists; if new nodes have issues, drain new group and reschedule onto old group.
- Karpenter-managed nodes: Karpenter doesn't use managed node groups; blue-green via NodePool swap is the natural approach.
- Trade-off: double cost during migration (two node groups running), more operational steps, requires careful label/taint management to route pods to the right group.

---

## Further reading / watching

- **EKS upgrade docs**: [docs.aws.amazon.com/eks/latest/userguide/update-cluster.html](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) — the official step-by-step.
- **EKS Best Practices — Upgrades**: [aws.github.io/aws-eks-best-practices/upgrades/](https://aws.github.io/aws-eks-best-practices/upgrades/) — the most complete single-page checklist with gotchas; read this annually before upgrade season.
- **`pluto` (Fairwinds)**: [github.com/FairwindsOps/pluto](https://github.com/FairwindsOps/pluto) — deprecated API detection; add to your pre-upgrade CI pipeline.
- **EKS addon versions**: [docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) — the compatibility matrix table; bookmark the version section.
- **`kubent` (kube-no-trouble)**: [github.com/doitintl/kube-no-trouble](https://github.com/doitintl/kube-no-trouble) — alternative to `pluto` for deprecated API detection; some teams prefer it.

---

## The 4 dimensions (senior framing)

- **Tech**: mandatory order — CP → addons → nodes. Kubelet N-2 skew. `pluto` for deprecated APIs. `max_unavailable_percentage` controls drain batch size. Blue-green for stateful or when rollback is needed. PDB interaction with drains is the most common upgrade stall.
- **People**: communicate before upgrade day — brief API blip, pods will reschedule during node drain. On-call needs to know that a stuck drain (pod won't evict) is a PDB issue, not an AWS issue. Write the upgrade runbook before you start, not after. Upgrade skills compound — the team that does this quarterly is fast; the team that does it annually is anxious.
- **CI/CD**: pin addon versions in Terraform and let Renovate open weekly PRs for upgrades. Run `pluto` in a pre-merge CI step on Helm chart changes. Test the full upgrade procedure in a non-production cluster first (staging EKS cluster, same version). GitOps (ArgoCD/Flux) may need a brief pause during CP upgrade if it's aggressively reconciling — some teams pause ArgoCD sync during the upgrade window.
- **Operations**: monitor API server availability before/during/after. Check SLO dashboards post-upgrade — watch for unexpected error rate increases. Verify addon pod health: `kubectl get pods -n kube-system`. Check node versions after upgrade: `kubectl get nodes`. Have a rollback plan (for control plane: EKS doesn't support downgrade — rollback means blue-green with old nodes ready. For nodes: keep old nodegroup until stable). Schedule upgrades quarterly to stay within 2 minor versions of current; letting it drift creates larger risk on each jump.
