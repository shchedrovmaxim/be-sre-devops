# kubeadm vs managed K8s — the simple version (the restaurant vs catering analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./kubeadm-vs-managed.md) becomes easy.

This doc only explains **one idea**:

> **Running kubeadm is like running your own restaurant kitchen. EKS/GKE/AKS is like hiring a catering company. The catering company frees you from kitchen management — but you pay more per meal and lose some control of the menu.**

---

## Do you want to run a kitchen?

| Running your kitchen (kubeadm) | Hiring caterers (EKS/GKE/AKS) |
|---|---|
| You buy the ovens, hire chefs, manage the kitchen | Cloud provider manages apiserver, etcd, scheduler, CCM |
| You fix the ovens when they break | Cloud SLA covers control plane availability |
| You decide exactly what's on the menu | You can customize nodes but not the control plane |
| Cheaper per meal at scale | Per-meal premium (~$0.10/hr for EKS control plane) |
| You can put the kitchen in any building | Your cluster can be in any cloud, on-prem, or air-gapped |

**The honest trade-off**: most companies running K8s on AWS/GCP/Azure should use managed K8s. The exceptions are narrow and specific.

---

## Who owns the control plane?

This is the core question.

```
kubeadm:
  You own:  apiserver, etcd, scheduler, controller-manager, TLS certs, upgrades
  Cloud owns: VMs (EC2), networking, storage

EKS/GKE/AKS:
  You own:  worker nodes, workloads, addons (CNI, ingress), application config
  Cloud owns: apiserver, etcd, scheduler, controller-manager, TLS certs, upgrades
```

Everything in "Cloud owns" for managed K8s is ops work that just disappears.

---

## The three reasons to use kubeadm

1. **You can't use a cloud provider.** Air-gapped environment, sovereign cloud, classified network, on-prem data center.

2. **You need full control over the control plane.** Custom admission plugins, specific etcd configuration, non-standard apiserver flags.

3. **The economics are right.** At very large scale (hundreds of nodes, many clusters), the per-cluster control plane fee adds up. Some companies run kubeadm internally and save on per-cluster costs.

Everything else? Use managed K8s.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Who manages etcd on EKS? | AWS. You can't access it directly. |
| What do I own on EKS? | Worker nodes, node groups, workloads, addons, IAM, VPC config. |
| When is kubeadm the right answer? | Air-gapped, on-prem, sovereign cloud, or very specific control plane customization needs. |
| What's the EKS control plane cost? | ~$0.10/hour per cluster (~$73/month). Small at one cluster; notable at 100 clusters. |
| What's cluster-api? | A K8s-native way to manage fleets of clusters (including kubeadm clusters) declaratively. |
| What does GKE Autopilot do? | Manages both control plane AND node provisioning. You only define workloads. |

---

## Self-test (one question — the killer one)

Out loud:

> **"When would you run kubeadm-managed K8s instead of EKS/GKE/AKS?"**

**Reference answer (intuitive version):**

"When you have to. The main reasons are: air-gapped environments where you can't reach cloud provider APIs; sovereign cloud or on-prem requirements where data residency rules prevent using AWS/GCP/Azure; and occasionally at very large scale where the per-cluster control plane fee becomes significant across hundreds of clusters. For everything else — the standard SaaS startup, the enterprise app team — managed K8s is the right call. The operational overhead of running etcd, upgrading the control plane manually, managing TLS cert rotation, and maintaining control plane HA is real work that pulls engineers away from higher-value problems."

---

## Further reading

- [EKS getting started](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [kubeadm setup](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Cluster API (CAPI)](https://cluster-api.sigs.k8s.io/)

---

## Next: the deep-dive

When the restaurant analogy feels obvious, jump to [`kubeadm-vs-managed.md`](./kubeadm-vs-managed.md). The deep-dive covers:

- The full responsibility matrix (who owns what)
- Upgrade complexity comparison
- Networking: CNI setup on kubeadm vs preconfigured on EKS
- Cost analysis: control plane fees vs ops time
- The bare-metal / sovereign cloud story
- Cluster API and Karmada for managing fleets
- 4 self-test drills + 4-dimensions framing
