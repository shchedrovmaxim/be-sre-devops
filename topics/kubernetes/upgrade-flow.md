# Upgrade flow — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through a safe K8s minor version upgrade. What's the order, what's the skew policy, what breaks?"** — naming the version skew rules, the deprecation check, addon compatibility, blue-green vs rolling nodes, and PDB's role in drains.

> Start with the [simple version](./upgrade-flow-simple.md) if you haven't read it. The bridge repair analogy is the spine.

---

## The senior framing — upgrades are the highest-risk routine operation

A bad K8s upgrade can:
1. Remove an API version your Helm charts depend on → all applies fail until charts are patched
2. Break a CNI plugin → all pod-to-pod networking stops
3. Get nodes stuck with an incompatible kubelet → pods can't be scheduled to those nodes
4. Leave PDBs blocking drain forever → nodes never get upgraded, you're stuck in a mixed-version limbo

This is why the upgrade procedure is the procedure most worth investing in. Senior engineers have runbooks for it, test it in staging first, and understand exactly what can fail and why.

---

## The version skew policy

### The rules

```
kube-apiserver (control plane):              v1.28
kube-controller-manager:                    ≤ v1.28   (within 1 minor of apiserver)
kube-scheduler:                             ≤ v1.28   (within 1 minor of apiserver)
kubelet (nodes):                            ≥ v1.26   (within 2 minor versions behind apiserver)
kube-proxy:                                 ≥ v1.26   (within 2 minor versions behind apiserver)
kubectl:                                    v1.27–v1.29 (within 1 minor ahead or behind apiserver)
```

Official source: [kubernetes.io/docs/setup/release/version-skew-policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)

**Why kubelets can lag 2 minor versions**: upgrading nodes takes time — you have to drain, upgrade, uncordon. During the control plane upgrade window, nodes are temporarily on an older version. The 2-version skew gives you the operational breathing room to upgrade CP first without immediately breaking all nodes.

**The direction constraint**: kubelets must be behind or equal to the apiserver. A kubelet 1 minor version ahead of the apiserver is not supported (the kubelet might use API features the apiserver doesn't have yet).

### Version skew in practice

Upgrading 1.26 → 1.27:

```
Before:
  Control plane: 1.26
  Nodes: 1.26

Step 1 (upgrade CP):
  Control plane: 1.27
  Nodes: 1.26   ← within 2-version skew, still valid

Step 2 (upgrade nodes):
  Control plane: 1.27
  Nodes: 1.27   ← fully upgraded
```

If you're 2 versions behind (upgrading 1.25 → 1.27), you must do two separate upgrade cycles: 1.25 → 1.26 → 1.27. **K8s only supports single-minor-version upgrades** for the control plane.

---

## Pre-upgrade checks — do this before touching anything

### 1. Deprecated API check

K8s deprecates APIs and removes them after ~3 minor versions. The most impactful removal: in K8s 1.22, `extensions/v1beta1`, `networking.k8s.io/v1beta1` (Ingress), and `rbac.authorization.k8s.io/v1beta1` were removed.

```bash
# pluto — the standard tool for this
pluto detect-helm -o wide             # check all Helm releases
pluto detect-files -d ./k8s-manifests # check file-based manifests

# output:
NAME                    NAMESPACE  KIND     VERSION           REMOVED  DEPRECATED  REPL AVAILABLE IN
ingress-nginx           default    Ingress  extensions/v1beta1 true    true        networking.k8s.io/v1  v1.22.0
```

The `apiserver_requested_deprecated_apis` Prometheus metric also shows which APIs are being actively called — useful for finding things pluto misses (dynamically generated manifests).

```promql
apiserver_requested_deprecated_apis{group="extensions",version="v1beta1",resource="ingresses"} > 0
```

If this fires after an upgrade, something is still using the removed API — find it and fix it before the next upgrade.

### 2. Addon compatibility check

Each addon has its own version compatibility matrix:

| Addon | Where to check |
|---|---|
| Calico | [Calico K8s support matrix](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements) |
| Cilium | [Cilium K8s support](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/) |
| CoreDNS | [CoreDNS changelog](https://github.com/coredns/coredns/releases) |
| AWS Load Balancer Controller | [ALBC compatibility](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/installation.md) |
| Karpenter | [Karpenter compatibility](https://karpenter.sh/docs/upgrading/compatibility/) |

**The most dangerous gap**: upgrading K8s without upgrading the CNI. If the new K8s version uses an API the CNI version doesn't know about, network policy enforcement can break silently.

### 3. Check current PDB configuration

```bash
# Find PDBs that would block draining
kubectl get pdb -A -o json \
  | jq -r '.items[] | select(.status.disruptionsAllowed == 0) |
            [.metadata.namespace, .metadata.name,
             (.spec.minAvailable // "N/A"),
             (.spec.maxUnavailable // "N/A")] | @tsv'
```

A PDB with `minAvailable: 100%` or `maxUnavailable: 0` and a Deployment with replicas=1 will block drain forever (you can't remove the only pod without violating the PDB). Fix before upgrading.

---

## The upgrade order

```
Phase 1: PRE-CHECKS
├── pluto scan (deprecated API check)
├── addon compatibility check
├── PDB audit (find blocking PDBs)
└── Take etcd snapshot (for self-managed) or verify cloud backup

Phase 2: UPGRADE CONTROL PLANE
├── EKS/GKE/AKS: update cluster version in Terraform/console
│   Managed clusters handle: apiserver, etcd, scheduler, controller-manager
└── kubeadm: kubeadm upgrade plan → kubeadm upgrade apply v1.28.0

Phase 3: UPGRADE ADDONS
├── CoreDNS (kubeadm does this automatically)
├── kube-proxy (kubeadm does this automatically)
├── CNI plugin (manual: Helm upgrade)
├── Metrics server, cert-manager, ingress controller (manual: Helm upgrade)
└── Wait for all addon pods to be Running/Ready

Phase 4: UPGRADE NODES
├── Option A: Blue-green node groups (preferred for large clusters)
└── Option B: In-place rolling (per-node drain/upgrade/uncordon)

Phase 5: POST-UPGRADE VALIDATION
├── kubectl get nodes (all at new version, all Ready)
├── kubectl get pods -A (no crashlooping, no pending)
├── Smoke test critical user journeys
└── Monitor error rates for 30 minutes
```

---

## Blue-green node groups vs in-place rolling upgrade

### Blue-green (preferred for production)

1. Create a new Auto Scaling Group (or node group) with nodes running the new K8s version.
2. Cordon all old nodes.
3. Let the cluster scheduler place new pods on new nodes.
4. Drain old nodes (workloads migrate to new nodes automatically).
5. Delete old node group.

```bash
# On EKS with managed node groups (Terraform)
resource "aws_eks_node_group" "v1_28" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers-v1-28"
  version         = "1.28"
  # ... rest of config
}

# After creating the new group:
# Cordon old nodes, wait for pods to reschedule, then drain and delete old group
```

**Advantages**:
- Old and new nodes coexist; if the new nodes have issues, you can roll back by uncordoning old nodes.
- No manual drain-per-node; the scheduler handles workload placement.
- Works naturally with Karpenter (just update the node pool version and Karpenter will provision new nodes on the new version and drain old ones).

**Disadvantages**:
- Temporarily 2× the node count → cost spike during migration.
- Requires ASG/node group management tooling.

### In-place rolling (kubeadm, smaller clusters)

```bash
# For each node (repeat per node):

# 1. Cordon (stop new pods from being scheduled here)
kubectl cordon <node-name>

# 2. Drain (evict all pods, respecting PDBs)
kubectl drain <node-name> \
  --ignore-daemonsets \       # DaemonSet pods can't be evicted normally
  --delete-emptydir-data \    # evict pods with emptyDir volumes
  --timeout=300s

# 3. SSH to the node and upgrade kubelet
# On Ubuntu/kubeadm:
apt-get update
apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
systemctl daemon-reload
systemctl restart kubelet

# 4. Uncordon (allow pods to be scheduled here again)
kubectl uncordon <node-name>

# 5. Wait for node to show Ready and pods to reschedule before doing the next node
kubectl get node <node-name> --watch
```

**Advantages**: simpler infrastructure; no cost spike; works on bare metal.

**Disadvantages**: manual per-node; if something goes wrong mid-drain, you're in a mixed state; slow for large clusters.

---

## PDB's role in node drains

During `kubectl drain`, K8s uses the **Eviction API** (not `kubectl delete pod`). The Eviction API respects PDBs.

If evicting a pod would violate the PDB (e.g., minAvailable=2 and only 2 pods running → evicting 1 would leave 1, violating minAvailable=2), the eviction returns 429 and drain blocks.

```bash
# Drain is blocking? Check which pods are blocked:
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --dry-run=client

# Find the PDB that's blocking:
kubectl get pdb -A | grep -v "ALLOWED"   # 0 = no disruptions allowed
```

**Common trap**: `minAvailable: 1` on a Deployment with `replicas: 1`. The one pod is on the node being drained. The PDB won't allow evicting it (that would take running count to 0 < minAvailable 1). Drain hangs until the pod is rescheduled — but the pod can't reschedule because the node isn't drained yet. Deadlock.

Fix: ensure Deployments that need to drain cleanly have `replicas >= 2` during the upgrade window, or use `maxUnavailable: 1` in the PDB.

---

## The "I upgraded CP, addons broke" pattern

The most common real-world upgrade issue:

```
1.27 → 1.28 upgrade complete on control plane.
CoreDNS is still at version X (shipped with 1.27).
A new feature in kube-proxy 1.28 uses an API CoreDNS X doesn't understand.
DNS resolution for some services starts failing intermittently.
```

Or:

```
AWS Load Balancer Controller version Y doesn't support K8s 1.28's updated admission API.
All Service type: LoadBalancer operations fail after the upgrade.
```

**Prevention**: upgrade addons in the same change window as the control plane, or at least on the same day. Don't leave addon upgrades for "later."

**Diagnosis**:
```bash
# Find addon versions post-upgrade
kubectl get pods -A -o json \
  | jq -r '.items[] | select(.metadata.namespace == "kube-system") |
            [.metadata.name, (.spec.containers[0].image)] | @tsv'
```

---

## Managed K8s upgrade vs kubeadm

| Step | kubeadm | EKS managed node groups | GKE autopilot |
|---|---|---|---|
| CP upgrade | `kubeadm upgrade apply` (you run it) | Console/Terraform/eksctl (cloud does it) | Automatic |
| Node upgrade | `drain + apt-get + uncordon` per node | Update node group version + rollout | Automatic |
| etcd backup | You run `etcdctl snapshot save` | AWS handles it | Google handles it |
| Addon upgrade | Manual Helm upgrades | Manual (except CoreDNS/kube-proxy) | Mostly automatic |
| Rollback | Restore etcd snapshot + revert kubeadm | New node group at old version | Limited |
| Operator skill required | High | Medium | Low |

The managed K8s advantage is clear: the cloud handles the hard parts. The kubeadm advantage: full control, no cloud dependency, works on-prem.

---

## The interview answer in 60 seconds

> "Before touching anything: deprecation check with `pluto` to find removed APIs in Helm charts, check addon compatibility matrices (Calico, AWS LBC, Karpenter all have K8s version support tables), audit PDBs for ones that would block drain. Then: upgrade control plane first — on EKS that's a Terraform update, on kubeadm it's `kubeadm upgrade apply`. The version skew policy lets kubelets lag the apiserver by up to 2 minor versions, so nodes at 1.26 keep working while CP is at 1.28.
>
> Then upgrade nodes: I prefer blue-green node groups — create a new ASG/node group at the new version, cordon old nodes, let the scheduler move workloads naturally, then drain and delete old nodes. This gives a rollback path if anything is wrong with the new nodes. Alternatively, drain-upgrade-uncordon per node for smaller clusters or kubeadm.
>
> Then upgrade addons — do this the same day, not 'later.' The most common post-upgrade failure I've seen: addons left at old versions that use removed APIs or incompatible features. Monitor error rates and DNS resolution for 30 minutes post-upgrade."

---

## Self-test drills

### 1. Walk me through a safe K8s minor version upgrade.

**Reference answer**: pluto check → CP upgrade → nodes (blue-green or rolling drain) → addon upgrades → validation. Version skew: kubelet can be 2 minor versions behind CP. See 60-second answer above.

### 2. A `kubectl drain` is blocked. Walk me through diagnosing it.

**Reference answer**: check `kubectl get pdb -A` — find any PDB with `ALLOWED: 0`. That PDB is blocking eviction. Common cause: `minAvailable: 1` on a Deployment with replicas=1 — the only pod can't be evicted without violating the PDB. Fix: scale to 2 replicas before draining (or set `maxUnavailable: 1` in the PDB). Also check for pods with `priorityClassName: system-node-critical` — they can't be evicted from nodes being drained without `--force`.

### 3. What's the version skew policy? Why can't you upgrade nodes before the control plane?

**Reference answer**: kubelet must be within 2 minor versions *behind* the apiserver. kubectl can be 1 minor version ahead or behind. If you upgraded a node before the CP (node at 1.28, CP at 1.27), the 1.28 kubelet would be ahead of the apiserver — unsupported and potentially using APIs the CP doesn't have. Always: CP first, then nodes, within the skew window.

### 4. What's the most likely thing to break after a K8s minor version upgrade?

**Reference answer**: (1) deprecated API removal — manifests/Helm charts using removed API versions fail silently until someone applies them post-upgrade; check with `pluto` pre-upgrade. (2) Addon incompatibility — CNI plugins, ingress controllers, cert-manager all have K8s version requirements; upgrading K8s without upgrading addons leaves the system in an inconsistent state. (3) PDB blocking drain — misconfigured PDBs that don't allow any disruption block the node upgrade process permanently.

---

## Further reading

- [K8s version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [pluto](https://github.com/FairwindsOps/pluto) — deprecated API detection
- [kubeadm upgrade guide](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [EKS managed node group upgrade](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html)
- [Karpenter upgrade guide](https://karpenter.sh/docs/upgrading/)

---

## The 4 dimensions (senior framing)

- **Tech**: version skew (kubelet ≤2 minor behind CP, kubectl ±1); control plane before nodes; blue-green node groups preferred; `kubectl drain` uses Eviction API + PDB; `pluto` for deprecated API detection; addon upgrade mandatory same window; `apiserver_requested_deprecated_apis` metric for ongoing monitoring.
- **People**: K8s upgrades are a team operation. Production upgrades require: pre-agreed maintenance window communicated to on-call and service owners; runbook reviewed by at least two engineers; post-upgrade validation sign-off from an application team rep. Don't let one engineer do a CP upgrade alone at 3pm on a Friday.
- **CI/CD**: upgrade staging first, wait 48 hours, then upgrade production. Staging should run the same addon versions as production so it's a real canary. Run your full integration test suite against staging post-upgrade. Pin K8s version in Terraform (don't use `latest`) so upgrades are deliberate, not accidental. Automate the `pluto` check as a CI gate that runs weekly — don't wait for upgrade day to find removed APIs.
- **Operations**: monitor the upgrade while it's happening: `kubectl get nodes --watch` (nodes going from old to new version), pod error rate, DNS resolution (CoreDNS health), ingress traffic. Have a rollback plan documented: for managed clusters, creating a node group at the old version and cordoning new nodes. For kubeadm, the rollback is an etcd restore (hence the pre-upgrade snapshot). Alert on `kubectl drain` timeouts — a drain that takes >10 minutes usually means a stuck PDB or a pod ignoring SIGTERM.
