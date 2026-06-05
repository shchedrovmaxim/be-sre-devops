# DaemonSet — the simple version (the security guard analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./daemonset.md) becomes easy.

This doc only explains **one thing**:

> **A DaemonSet makes sure every node (or a specific subset of nodes) runs exactly one copy of a pod. When a new node joins the cluster, it automatically gets the pod. When a node leaves, the pod goes with it.**

---

## The building security guard

A company has multiple office floors (nodes). They want exactly one security guard on every floor — no more, no less. Doesn't matter how many employees are on each floor (how many other pods are running). The floor is what matters, not the workload.

| In the building world | In the K8s world |
|---|---|
| One guard per floor | One pod per node |
| A new floor opens → a guard is immediately assigned | A new node joins → a DaemonSet pod is immediately scheduled |
| A floor closes → the guard assigned there is dismissed | A node is removed → its DaemonSet pod is terminated |
| Guards can also patrol restricted executive floors with the right badge | DaemonSet pods can run on tainted nodes (e.g., control plane) with the right tolerations |
| Some guards patrol only specific zones ("east wing only") | DaemonSet can be scoped to a subset of nodes via `nodeSelector` or `nodeAffinity` |

The company doesn't say "I need 3 guards" (that's a Deployment). They say "I need **one guard per floor**." The headcount scales automatically with the number of floors.

---

## The two confusing concepts

### 1. How DaemonSet pods get onto tainted nodes (like control plane)

Control plane nodes have a taint to keep regular workloads off:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

Normally, pods that don't explicitly tolerate this taint won't land on control-plane nodes.

DaemonSets frequently need to run on **every** node — including control plane — for things like log shippers and node metrics exporters. To do this, add a toleration:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

This is the DaemonSet version of "giving the guard a badge to access restricted floors."

### 2. The update strategy — rolling vs. on-delete

A Deployment automatically rolls out changes to all pods in a controlled surge-drain sequence.

A DaemonSet has two update options:

- **`RollingUpdate`** (default): K8s terminates and re-creates pods one-at-a-time across nodes. You can control how many nodes update simultaneously with `maxUnavailable`.
- **`OnDelete`**: K8s does nothing automatically. Old pods run until you manually delete them, at which point the DaemonSet re-creates them with the new spec.

Use `OnDelete` when updates need to be gated by human sign-off — e.g., a kernel module agent where you want to test on one node manually before touching the rest.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What guarantees does DaemonSet make? | One pod per node (or per eligible node if filtered) |
| What happens when a new node is added? | The DaemonSet controller automatically schedules a pod on it |
| How do you run a DaemonSet on a subset of nodes? | `nodeSelector` or `nodeAffinity` on the pod template |
| How do you get DaemonSet pods on control-plane nodes? | Add the right `toleration` for the control-plane taint |
| What's the default update strategy? | `RollingUpdate` — `maxUnavailable: 1` by default (one node at a time) |
| Common use cases? | Log forwarders (Fluentd, Fluent Bit), node metrics (node-exporter), CNI agents, device plugins |
| Can you have fewer than one pod per node? | No. But you can exclude nodes via nodeAffinity or taint/toleration. |

---

## Self-test (one question — the killer one)

Out loud:

> **"When would you use a DaemonSet, and how do they get scheduled on tainted nodes?"**

**Reference answer (intuitive version):**

"Use a DaemonSet when you need exactly one pod per node — regardless of how many replicas of your app you're running. The classic use cases are infrastructure-level agents: log forwarders (like Fluent Bit), node metrics exporters (node-exporter for Prometheus), CNI plugins, and device drivers. These need to run on every node because they interact with node-level resources — the kernel, the filesystem, the network stack.

For tainted nodes like control plane: by default, control-plane nodes have a `NoSchedule` taint that blocks regular pods. DaemonSet pods can run there by adding a matching `toleration` to the pod template. Kubernetes's own system DaemonSets (kube-proxy, CNI agents) do this automatically. For your own DaemonSets, you add the toleration explicitly — it's just two lines of YAML. The scheduler then sees the taint is tolerated and allows the pod to land on that node."

---

## Further reading / watching

- **Kubernetes Docs — DaemonSet**: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- **Fluent Bit on Kubernetes** — a real-world DaemonSet deployment: https://docs.fluentbit.io/manual/installation/kubernetes
- **node-exporter DaemonSet** — the Prometheus community's chart is a good reference YAML

---

## Next: the deep-dive

When the building-guard analogy feels solid, jump to [`daemonset.md`](./daemonset.md). The deep-dive covers:

- The exact scheduling mechanics (how DaemonSet bypasses the normal scheduler)
- Tolerations in detail for all the common node taints (control-plane, not-ready, disk-pressure)
- Update strategy deep dive — `maxUnavailable` math, `OnDelete` use cases
- HostPath vs. proper sidecar tradeoff
- The "don't I just need a Deployment with hostNetwork?" trap
- 4 self-test drills + 4-dimensions framing
