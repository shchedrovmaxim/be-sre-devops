# Upgrade flow — the simple version (the bridge repair analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./upgrade-flow.md) becomes easy.

This doc only explains **one idea**:

> **A K8s upgrade is like replacing planks on a bridge while cars are still driving over it. The order matters. If you do it wrong, the cars fall through.**

---

## Replacing planks on a live bridge

The bridge has two parts: the support structure (control plane) and the planks (nodes). Here's the rule:

1. **Fix the support structure first.** If you replace planks before the support structure is sound, planks might not fit.
2. **Replace planks one at a time, not all at once.** Otherwise you have no bridge.
3. **Let the cars finish crossing before you remove a plank.** That's what graceful drain is.

| Bridge world | K8s upgrade world |
|---|---|
| The support structure (girders, cables) | **Control plane** (apiserver, etcd, scheduler, controller-manager) |
| Individual planks | **Worker nodes** |
| Replacing the support structure first | **Upgrade control plane before nodes** |
| Cars finishing before plank is removed | **Pod drain before node upgrade** |
| Two cars can't cross if there's only one lane | **PDB: minAvailable ensures pods stay up during drain** |

**The fundamental rule: control plane first, then nodes. Never the other way around.**

---

## The version skew policy — the guardrails

K8s has strict rules about which versions can talk to each other:

| Component | Allowed skew |
|---|---|
| **kubelet** | Can be up to **2 minor versions behind** the apiserver |
| **kubectl** | Can be up to **1 minor version** ahead or behind the apiserver |
| **kube-proxy** | Should match the apiserver version |

Example: you're upgrading from 1.27 → 1.28:
- **Upgrade apiserver/control plane to 1.28 first**
- Your nodes (kubelet) at 1.27 are still within the 2-version skew — they keep working
- Upgrade nodes to 1.28 at your own pace

If you tried to upgrade a node to 1.28 before the control plane, the 1.28 kubelet would be talking to a 1.27 apiserver — that's going forward, not backward. The skew policy only allows kubelets *behind* the apiserver, not ahead.

---

## The four-step upgrade

```
1. CHECK
   ├── Run deprecation API check (kubectl deprecations or pluto)
   └── Check addon compatibility: CNI, CoreDNS, kube-proxy

2. UPGRADE CONTROL PLANE (cloud: click a button; kubeadm: kubeadm upgrade apply)
   └── Nodes still on old version — that's fine (within skew)

3. UPGRADE NODES one by one (or group by group)
   ├── kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   ├── Upgrade kubelet and kubectl on the node
   └── kubectl uncordon <node>

4. UPGRADE ADDONS (CNI, CoreDNS, kube-proxy, Ingress controller)
   └── They need to be compatible with the new K8s version
```

---

## The things that break

**Deprecated API removal**: K8s removes deprecated APIs after ~3 minor versions. If you're using `extensions/v1beta1/Ingress` and K8s 1.22 removes it, your apply will fail after the upgrade. Check with `pluto` before upgrading.

**Addon incompatibility**: CoreDNS, CNI plugins (Calico, Cilium), and the Ingress controller all have K8s version requirements. Check their compatibility matrices.

**PDB blocking drain**: if you have `minAvailable: 100%` (or `maxUnavailable: 0` set incorrectly), `kubectl drain` will block because it can't evict any pods. Fix: allow at least 1 pod to be unavailable during drain.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What goes first in an upgrade? | Control plane always first. |
| How far can a kubelet lag behind the apiserver? | 2 minor versions (e.g., apiserver at 1.28, kubelet at 1.26 is still valid). |
| What does `kubectl drain` do? | Marks the node unschedulable, evicts all pods (respecting PDBs), waits for them to reschedule elsewhere. |
| What is `pluto`? | A tool that scans your manifests for deprecated/removed API versions before you upgrade. |
| Why upgrade CNI/CoreDNS after? | Because they depend on K8s APIs. Upgrading K8s can remove APIs those addons relied on. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through a safe K8s minor version upgrade. What's the order, what's the skew policy, what breaks?"**

**Reference answer (intuitive version):**

"Control plane first, always. Before touching anything, I'd run deprecation checks with `pluto` to find any resources using removed APIs — those break silently on the first apply after upgrade. Then upgrade the control plane (on EKS, that's a button in the console or a Terraform change). Nodes still run the old kubelet — that's within the 2-minor-version skew, so they keep working. Then drain and upgrade nodes one at a time or in rolling groups: `kubectl drain`, upgrade kubelet, `kubectl uncordon`. PDBs control how many pods can be missing during drain, so I'd check there are no `minAvailable: 100%` PDBs that would block draining. Last, upgrade addons: CNI, CoreDNS, kube-proxy, Ingress controllers — they need to match the new K8s version. The most common breakage I've seen is deprecated API removal breaking Helm charts."

---

## Further reading

- [K8s version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [pluto — deprecated API scanner](https://github.com/FairwindsOps/pluto)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

---

## Next: the deep-dive

When the bridge analogy feels obvious, jump to [`upgrade-flow.md`](./upgrade-flow.md). The deep-dive covers:

- The skew policy in detail (kubelet, kubectl, kube-proxy)
- Pre-upgrade deprecation checks and the API removal timeline
- Blue-green node groups vs in-place rolling upgrades
- PDB role in node drains
- Addon compatibility matrices
- Managed K8s upgrade vs kubeadm
- 4 self-test drills + 4-dimensions framing
