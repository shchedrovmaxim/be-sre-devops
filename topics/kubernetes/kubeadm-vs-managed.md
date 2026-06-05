# kubeadm vs managed K8s (EKS/GKE/AKS) — the deep-dive

> **Goal**: by the end you can answer the killer question — **"When would you run kubeadm-managed K8s instead of EKS/GKE/AKS?"** — naming the narrow set of valid reasons, the full responsibility matrix, cost trade-offs, and the meta-tools (cluster-api, Karmada) for multi-cluster fleets.

> Start with the [simple version](./kubeadm-vs-managed-simple.md) if you haven't read it. The restaurant vs catering analogy is the spine.

---

## The senior framing — the question isn't "can we," it's "should we"

Teams run kubeadm when they shouldn't more often than the reverse. The failure mode is: "we want full control" + "we'll have time to manage it" → six months later, the team is spending 30% of their time on K8s infrastructure instead of product.

The honest accounting of what you own when you run kubeadm:

- **Control plane high availability**: you manage the etcd cluster (3/5 members, quorum, disk performance), the apiserver HA (load balancer, multiple replicas), the scheduler/controller-manager leader election.
- **TLS certificate rotation**: control plane certs expire. kubeadm auto-renews them annually, but you have to run the renewal and restart components. If you forget, the cluster dies.
- **Kubernetes upgrades**: you execute the upgrade procedure manually, per node. There's no "click to upgrade."
- **Security patching**: when a K8s CVE drops (and they do), you patch it manually. Cloud providers patch managed clusters automatically.
- **etcd backups**: you set up automated snapshots and test restores.
- **Networking**: you choose and configure the CNI, set up pod CIDR ranges, handle network policy enforcement.

On EKS/GKE/AKS, all of these disappear from your list.

---

## The full responsibility matrix

| Responsibility | kubeadm | EKS | GKE Standard | GKE Autopilot |
|---|---|---|---|---|
| apiserver HA | You | AWS | Google | Google |
| etcd management | You | AWS | Google | Google |
| etcd backups | You | AWS | Google | Google |
| Control plane upgrades | You | You (button/API) | You (button/API) | Google |
| TLS cert rotation | You (kubeadm auto-renews) | AWS | Google | Google |
| Security patches | You | AWS | Google | Google |
| Worker node provisioning | You | You (EC2/ASG/Karpenter) | You (GCE/MIG) | Google |
| Worker node upgrades | You | You | You | Google |
| CNI selection | You | Preconfigured (VPC CNI, optionally Cilium) | Preconfigured (VPC-native) | Google |
| Ingress/LB provisioning | You set up CCM | AWS CCM (ALBC) built in | GKE Ingress built in | Google |
| Node auto-scaling | You (CA) | Karpenter/CA | GKE Autopilot / CA | Google |
| RBAC / IAM integration | You configure everything | aws-auth / Access Entries | GKE IAM integration | GKE IAM integration |
| Cost model | VMs only | VMs + $0.10/hr per cluster | VMs + per-cluster fee | Per-pod resource pricing |

---

## Control plane cost analysis

### The EKS control plane fee

EKS charges **$0.10/hour per cluster** (~$73/month) for the control plane. At one or two clusters, this is noise. At 50+ clusters (common in a large enterprise with dev/staging/prod clusters per team), the math changes:

```
50 clusters × $73/month = $3,650/month = $43,800/year
```

Compared to running kubeadm on EC2 (no per-cluster fee, just the control plane VMs):

```
3 EC2 m5.xlarge for control plane: ~$480/month per cluster
50 clusters (kubeadm): $24,000/month
50 clusters (EKS): $3,650/month (control plane) + VM costs for nodes
```

Wait — at 50 clusters, EKS is actually **cheaper** than kubeadm (because 3 × m5.xlarge for control plane VMs costs more than $73/month). The economics flip: kubeadm is cheaper at very large scale only if you run control planes on very small/cheap VMs or co-locate control planes, both of which add complexity.

**The real kubeadm cost saving**: only when you run a large number of clusters on bare metal where there's no cloud control plane fee overhead — you're paying only for the metal, not per-cluster.

### The ops time cost

The true cost of kubeadm is engineer time:
- Initial setup: 40-80 hours for production-grade setup
- Ongoing maintenance: 4-8 hours/month per cluster (upgrades, cert rotation, incident response)
- At $150/hr engineering cost, 8 hours/month × 12 months = $14,400/year per cluster

One cluster: kubeadm ops cost ≈ $14,400/year. EKS fee ≈ $876/year. The "free" infrastructure is rarely free.

---

## Networking: the biggest self-managed gotcha

On EKS, AWS VPC CNI is preconfigured. Pod IPs are real VPC IPs — no overlay network, native routing, native security groups for pods.

On kubeadm, **you choose and install the CNI yourself**:

```bash
# Example: installing Calico on a kubeadm cluster
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# You must also configure: pod CIDR, MTU, encapsulation mode, NetworkPolicy support
```

Things you're responsible for on kubeadm:
- Choosing between Calico, Cilium, Flannel, Weave, Antrea, etc.
- Configuring the pod CIDR to not conflict with node or VPC CIDRs
- Ensuring NetworkPolicy is supported and configured
- Handling MTU issues (common with overlay networks on certain EC2 instance types)
- Upgrading the CNI independently of K8s upgrades

On GKE and EKS, a significant subset of this is preconfigured and tested by the cloud provider.

---

## When kubeadm IS the right answer

### 1. Air-gapped environments

If your cluster can never reach the internet (classified networks, SCIF, some finance/defense environments), managed K8s often doesn't work — the control plane management plane calls home to cloud APIs. kubeadm on bare metal or private VMs is the standard here.

```bash
# Air-gapped kubeadm: pre-pull all images to a private registry
kubeadm config images pull --image-repository my-registry.internal

# Install using local packages (no internet access)
apt-get install -y kubelet=1.28.0-00 kubeadm=1.28.0-00 kubectl=1.28.0-00
```

### 2. Sovereign cloud / data residency requirements

Some jurisdictions require that infrastructure management data (API calls, control plane communications) stays within the country/region. Some sovereign cloud providers (e.g., OVHCloud, Deutsche Telekom's Open Telekom Cloud) offer K8s but not EKS/GKE/AKS. kubeadm or kubeadm-based distributions (RKE2, K3s, Talos) are the answer here.

### 3. Full control plane customization

EKS lets you pass some custom flags to the apiserver, but not all. If you need:
- Custom admission plugins not supported by managed K8s
- Non-standard etcd encryption configuration
- Specific apiserver audit policy that managed K8s doesn't expose
- The ability to compile custom apiserver builds (rare but exists in research contexts)

### 4. Bare metal with specific hardware requirements

GPU clusters for ML workloads, high-performance networking (InfiniBand, SR-IOV), specific NUMA topology requirements — some hardware configurations don't work well under cloud VM abstraction. On bare metal with kubeadm, you have direct hardware access.

---

## Cluster API (CAPI) and Karmada — managing fleets

When you need to manage many kubeadm clusters (or a mix of managed and self-managed), you need a meta-layer.

### Cluster API

Cluster API is a K8s-native tool that manages cluster lifecycle declaratively. You define clusters as K8s custom resources; CAPI creates and manages the infrastructure.

```yaml
# A Cluster object in CAPI (creates a kubeadm cluster on AWS)
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-prod-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-prod-cp
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: my-prod-aws
```

CAPI providers exist for AWS, GCP, Azure, vSphere, bare metal (via Metal3), and more. The management cluster (itself a K8s cluster) runs the CAPI controllers. This is how large enterprises manage hundreds of kubeadm clusters with GitOps tooling.

### Karmada

Karmada distributes workloads across multiple clusters (both managed and kubeadm). You deploy to Karmada; it propagates to the right clusters based on policies.

```yaml
# PropagationPolicy: deploy to all prod clusters
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: payments-propagation
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: payments-api
  placement:
    clusterAffinity:
      matchLabels:
        environment: production
    spreadConstraints:
    - spreadByField: cluster
      maxGroups: 3
      minGroups: 2
```

Karmada is the answer to "we have 20 clusters in different regions and need to deploy consistently across all of them without 20 separate Argo CD instances."

---

## Kubeadm distributions worth knowing

Several distributions package kubeadm-based K8s with production-ready defaults:

| Distribution | Key features | Use case |
|---|---|---|
| **RKE2** (Rancher) | FIPS compliance, hardened by default, Rancher integration | Regulated industries, enterprises using Rancher |
| **K3s** | Lightweight (~70MB binary), ARM support | Edge, IoT, single-node, resource-constrained |
| **Talos Linux** | Immutable OS, API-driven, no SSH | Security-focused, GitOps-first clusters |
| **Vanilla kubeadm** | Maximum flexibility | When you need direct control |

---

## The interview answer in 60 seconds

> "Managed K8s — EKS, GKE, AKS — is the right answer for the overwhelming majority of use cases. The cloud provider owns the control plane: etcd management, apiserver HA, TLS cert rotation, security patching, and upgrades are all handled. You focus on nodes and workloads. The cost is ~$73/month per cluster on EKS — trivial at one cluster; notable at 50+, but still usually cheaper than the engineering time to manage the control plane yourself.
>
> The narrow cases where kubeadm makes sense: air-gapped environments where you can't reach cloud provider APIs; sovereign cloud or on-prem deployments where regulatory requirements prevent using AWS/GCP/Azure; and very specific control plane customization needs (custom admission plugins, non-standard etcd configuration) that managed K8s doesn't expose.
>
> For managing fleets of kubeadm clusters, Cluster API is the standard — you define clusters as K8s CRDs and CAPI handles provisioning across cloud providers and bare metal. Karmada handles multi-cluster workload distribution. At very large scale (100+ clusters), both tools together give you GitOps-driven cluster fleet management."

---

## Self-test drills

### 1. When would you run kubeadm instead of EKS/GKE/AKS?

**Reference answer**: air-gapped networks, sovereign cloud / data residency requirements, on-prem bare metal, very specific control plane customization. For everything else, managed K8s. See the decision tree above.

### 2. What do you own on EKS that you don't on GKE Autopilot?

**Reference answer**: on EKS, you own: worker nodes (EC2 instances), node groups, node upgrades, addons (CNI beyond VPC CNI defaults, ingress controllers, cert-manager, etc.), Karpenter or CA configuration. On GKE Autopilot, even the nodes are managed by Google — you only define pod specs. Autopilot bills by pod resource requests, not by VMs.

### 3. What is Cluster API and when would you use it?

**Reference answer**: CAPI is a K8s-native tool that manages cluster lifecycle declaratively — clusters are K8s custom resources. You use it when managing many clusters (e.g., one per team, or per region), especially if those clusters are self-managed (kubeadm). It provides: consistent cluster provisioning across cloud providers, GitOps-friendly cluster management, automated node group scaling, and upgrade automation. The management cluster is itself a K8s cluster running the CAPI controllers.

### 4. At what scale does kubeadm become more economical than EKS?

**Reference answer**: it rarely does, because the EKS control plane fee ($73/month) is much less than the engineering cost of running control plane VMs and the ops time for maintenance. The calculation only flips if you're on bare metal (no per-VM cost, no control plane VM cost) at very large scale. For cloud-based clusters, EKS/GKE/AKS is almost always the economical choice when you factor in engineer hours.

---

## Further reading

- [EKS pricing](https://aws.amazon.com/eks/pricing/)
- [GKE cluster types](https://cloud.google.com/kubernetes-engine/docs/concepts/types-of-clusters)
- [Cluster API docs](https://cluster-api.sigs.k8s.io/)
- [Karmada](https://karmada.io/)
- [Talos Linux](https://www.talos.dev/)
- [RKE2](https://docs.rke2.io/)

---

## The 4 dimensions (senior framing)

- **Tech**: control plane ownership is the core distinction; kubeadm responsibility matrix (etcd, TLS certs, upgrades, networking); EKS/GKE/AKS responsibility matrix; CAPI for fleet management; Karmada for multi-cluster workload distribution; kubeadm distributions (RKE2 for regulated industries, K3s for edge, Talos for security-first). The skew toward managed K8s is correct for most organizations.
- **People**: the "we want full control" instinct often comes from engineers who want to learn by doing, not from a real business requirement. That's a valid learning goal — but it shouldn't be the production answer. Separate learning clusters (on kubeadm, on a personal account) from production infrastructure decisions. Push back politely but clearly when kubeadm is proposed for production without a specific business requirement.
- **CI/CD**: managed K8s makes CI/CD simpler — cluster provisioning via Terraform (aws_eks_cluster, google_container_cluster), predictable addon compatibility, and cloud provider-managed upgrade buttons reduce the CD surface area. kubeadm requires more custom automation: Ansible/Terraform for control plane provisioning, kubeadm bootstrap, CNI deployment, all as CI steps. CAPI is the GitOps-native answer to automating kubeadm cluster lifecycle.
- **Operations**: managed K8s shifts your on-call focus from "is the control plane healthy?" to "are my workloads healthy?" You never page on etcd quorum loss; you page on application error rates. This is the right shift — it's the work that creates customer value. The SLA from AWS/GCP/Azure for control plane uptime is typically 99.9-99.95%. That's your baseline; operational effort can focus above that layer.
