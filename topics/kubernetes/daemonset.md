# DaemonSet — one pod per node (deep-dive)

> **Goal**: by the end you can answer the killer question — **"When would you use a DaemonSet, and how do they get scheduled on tainted nodes?"** — naming common use cases, the scheduling mechanics, toleration requirements for control-plane and system taints, update strategies, and the HostPath-vs-sidecar tradeoff.

> Start with the [simple version](./daemonset-simple.md) if you haven't read it. The security-guard analogy is the spine of this topic.

---

## Mental model: node-scoped rather than replica-scoped

A Deployment says: "run N replicas somewhere in the cluster." K8s picks nodes.
A DaemonSet says: "run one pod on **every node** (or every eligible node)." The count scales with your node count.

This distinction matters because:
- Node count ≠ replica count. A 100-node cluster with 3 replicas of your API is fine. The same cluster needs 100 instances of the log forwarder.
- Node-level agents need access to the node's filesystem, network namespace, or devices — things that don't make sense in a replicated-service model.

---

## Scheduling mechanics

Normal pods go through the Kubernetes scheduler. The scheduler finds nodes that fit (resources, affinity, taints) and assigns pods. DaemonSet pods work differently:

- The DaemonSet controller creates pods with `.spec.nodeName` **pre-filled** — bypassing the scheduler queue.
- This means DaemonSet pods are not subject to scheduler priority or preemption in the normal sense.
- It also means they bypass scheduling constraints that would normally prevent placement — unless you explicitly configure tolerations.

When a new node joins the cluster, the DaemonSet controller spots it via the Node watch, creates a pod spec with that node's name pre-filled, and the pod appears within seconds. No re-schedule, no waiting for the scheduler cycle.

---

## Tolerations — how DaemonSet pods run on tainted nodes

The key system taints and what to tolerate:

```yaml
spec:
  template:
    spec:
      tolerations:
      # Run on control-plane nodes
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule

      # Also tolerate the older master label (pre-K8s 1.24 compatibility)
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule

      # Run on nodes with disk pressure
      - key: node.kubernetes.io/disk-pressure
        operator: Exists
        effect: NoSchedule

      # Run on nodes that aren't ready yet (for bootstrap agents)
      - key: node.kubernetes.io/not-ready
        operator: Exists
        effect: NoExecute

      # Run on nodes with memory pressure
      - key: node.kubernetes.io/memory-pressure
        operator: Exists
        effect: NoSchedule

      # Run on nodes with uninitialized state (cloud provider bootstrap)
      - key: node.kubernetes.io/uninitialized
        operator: Exists
        effect: NoSchedule
```

K8s's own system DaemonSets (kube-proxy, Calico node, etc.) tolerate `not-ready` and `uninitialized` because they need to bootstrap even before the node is fully ready. A Fluentd forwarder, on the other hand, probably doesn't need to run on a not-ready node.

**Rule of thumb**:
- CNI plugins and network agents: tolerate everything (node can't be Ready without them)
- Log forwarders, metrics exporters: tolerate `control-plane` and `disk-pressure`; skip `not-ready`/`uninitialized`
- Your custom agents: add only the taints you actually need

---

## Scoping to a subset of nodes

Run on only nodes with a specific label:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        accelerator: gpu        # only GPU nodes get this DaemonSet
```

Or use `nodeAffinity` for more complex rules:

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["c5.xlarge", "c5.2xlarge"]
```

Use case: GPU drivers as a DaemonSet only on GPU nodes. Storage agents only on nodes with specific instance types. Windows-specific agents only on Windows nodes.

---

## Update strategy

```yaml
spec:
  updateStrategy:
    type: RollingUpdate        # or OnDelete
    rollingUpdate:
      maxUnavailable: 1        # how many nodes can be updating at once
                               # can also be a percentage: "10%"
```

### `RollingUpdate` (default)

When you update the pod template, the DaemonSet controller terminates pods on nodes one at a time (or `maxUnavailable` at a time), waits for the new pod to be Running, moves to the next node.

`maxUnavailable: 1` means at most 1 node at a time has a non-running DaemonSet pod during the update. With 100 nodes, that's 100 sequential single-node updates. For `maxUnavailable: 10%`, 10 nodes update in parallel — faster, but 10 nodes' worth of DaemonSet functionality is temporarily absent.

**Practical advice**: for node-critical agents (CNI, kube-proxy), keep `maxUnavailable: 1`. For observability agents (log forwarder, metrics exporter), a higher value is fine — a brief gap in logs is acceptable.

### `OnDelete`

K8s does NOT automatically roll out changes. The new spec is stored, but existing pods keep running the old version. Only when you manually delete a pod does K8s re-create it with the new spec.

Use `OnDelete` when:
- You want to canary one node manually before rolling out cluster-wide
- The agent is a kernel module or low-level component where an automatic rollout is too risky
- You need human sign-off per-node before proceeding

Workflow with `OnDelete`:
```bash
# Update the DaemonSet spec
kubectl set image daemonset/my-agent app=my-agent:2.0

# Manually delete one node's pod — the new pod will use the new image
kubectl delete pod my-agent-abc12 -n kube-system

# Validate the new pod is working as expected on that node
kubectl logs my-agent-xyz99 -n kube-system

# Continue to the next node
kubectl delete pod my-agent-def34 -n kube-system
```

---

## Common use cases

| Use case | Example | Why DaemonSet |
|---|---|---|
| Network plugin | Calico node, Cilium agent, Flannel | CNI must run on every node; manages node-level iptables/eBPF |
| Log forwarding | Fluent Bit, Fluentd, Filebeat | Reads logs from the node's `/var/log/containers` — must run on every node |
| Node metrics | Prometheus `node-exporter` | Exposes OS-level metrics (CPU, memory, disk) from each node |
| Cluster storage agent | Portworx, Rook Ceph OSD | Storage drivers that manage node-attached disks |
| Security agent | Falco, Sysdig | Node-level syscall monitoring via kernel |
| GPU device plugin | NVIDIA Device Plugin | Registers GPU resources with kubelet on each GPU node |
| Service mesh data plane | Istio/Envoy (legacy pre-sidecar-injection) | Less common now; native sidecar injection replaced DaemonSet for this |

---

## HostPath vs. proper node-level access

A question that comes up: "Can't I just use a Deployment with a `hostPath` volume to access node filesystem?"

Technically yes. But here's why DaemonSet is right:

| | Deployment with hostPath | DaemonSet |
|---|---|---|
| Pods per node | Might be 0 (if not scheduled there) or >1 | Exactly 1 |
| New node joins | Pod only scheduled if resources fit / affinity matches | Pod automatically placed on new node |
| Intent in YAML | Implicit (you hope the scheduler places correctly) | Explicit ("one per node") |
| Review in kubectl | `kubectl get pods -o wide` to see what landed where | `kubectl get ds` shows DESIRED/CURRENT/READY per DS |

Using a Deployment for node-level agents is fragile — nodes can have zero agents or multiple agents if scheduling decisions go wrong. DaemonSet gives an explicit, reviewable guarantee.

The caveat: if your "agent" doesn't actually need to run on every node, a Deployment + affinity is the right tool. DaemonSet is for genuine "one per node" semantics.

---

## The complete annotated YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node.kubernetes.io/disk-pressure
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            memory: 128Mi           # intentionally no CPU limit — see resource-mgmt doc
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config
```

Key callouts:
- `tolerations` for `control-plane` ensures the log forwarder also runs on control-plane nodes.
- `hostPath` volumes for `/var/log` and the container log directory — this is legitimate for a log forwarder; it genuinely needs to read the node's log directories.
- No CPU limit intentionally — log forwarders can burst during high-write periods. CPU throttling causes log drops. Memory limit is set because OOM is less recoverable than CPU slowness. (See [resource-mgmt-requests-limits.md](./resource-mgmt-requests-limits.md).)

---

## The interview answer in 60 seconds

> "DaemonSets are for workloads that have a 1-per-node requirement rather than a replica-count requirement. Classic examples: log forwarders that read from node-level log directories, node metrics exporters, CNI agents, and GPU device plugins. You can't use a Deployment for these because the scheduler might land 0 or 2 pods on a given node — a DaemonSet guarantees exactly 1.
>
> The DaemonSet controller bypasses the normal scheduler queue by pre-filling `.spec.nodeName` on the pod. When a new node joins, the controller sees it and creates the pod immediately.
>
> For tainted nodes like control-plane: the control-plane nodes have a `node-role.kubernetes.io/control-plane:NoSchedule` taint. Any pod that doesn't explicitly tolerate this taint won't land there. DaemonSet pods for infrastructure agents (node-exporter, Fluent Bit) should add that toleration — two lines of YAML. CNI plugins and kube-proxy tolerate even `node.kubernetes.io/not-ready` because they need to run before the node reaches Ready state.
>
> Update strategy: `RollingUpdate` is the default — roll out one node at a time. `OnDelete` if you need manual per-node sign-off."

---

## Self-test

### 1. When would you use a DaemonSet, and how do they get scheduled on tainted nodes?

**Reference answer:** See the 60-second answer above. Key points: 1-per-node requirement, node-level resource access, DaemonSet controller bypasses scheduler via pre-filled nodeName, toleration needed for each taint the node carries.

### 2. A new node joined your cluster but the DaemonSet pod doesn't appear on it. What do you check?

**Reference answer:** Check node labels — if the DaemonSet has `nodeSelector`, the new node must have matching labels. Check node taints — if the new node has taints that the DaemonSet doesn't tolerate, pods won't be placed. `kubectl describe node <new-node>` shows labels and taints. `kubectl describe ds <name>` shows events. Also check: are there resource requests that the new node can't satisfy?

### 3. You need to update the Fluent Bit DaemonSet image on one node to test a new config before rolling out to all nodes. How?

**Reference answer:** Switch the DaemonSet to `updateStrategy: type: OnDelete`. Apply the image update to the DaemonSet spec. K8s won't automatically touch existing pods. Manually delete the pod on the target test node (`kubectl delete pod <pod> -n logging`). The DaemonSet controller creates a new pod with the new image on that node. Validate. When satisfied, manually delete pods on other nodes one by one (or flip back to `RollingUpdate`).

### 4. Why is a DaemonSet preferred over a Deployment with `hostPath` for node-level agents?

**Reference answer:** A Deployment's pod count is fixed regardless of node count. In a 100-node cluster, a 3-replica Deployment with `hostPath` leaves 97 nodes with no agent — you'd have to manually set `replicas: 100` and update it every time the cluster scales. Even then, scheduling might cluster multiple pods on one node and leave others empty. DaemonSet expresses the intent ("one per node") explicitly, enforces it automatically on new nodes, and shows the correct desired/current state in `kubectl get ds`. Using a Deployment for node agents is a leaky abstraction.

---

## Further reading / watching

- **Kubernetes Docs — DaemonSet**: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- **Kubernetes Docs — Taints and Tolerations**: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- **Fluent Bit Kubernetes installation** — real DaemonSet YAML: https://docs.fluentbit.io/manual/installation/kubernetes
- **NVIDIA Device Plugin** — a GPU DaemonSet with complex tolerations: https://github.com/NVIDIA/k8s-device-plugin

---

## The 4 dimensions (senior framing)

- **Tech**: DaemonSet controller pre-fills `nodeName` bypassing the scheduler; `tolerations` required per taint; `nodeSelector`/`nodeAffinity` to scope to a subset; `RollingUpdate` with `maxUnavailable` for controlled rollout; `OnDelete` for manual canary rollout on node-critical agents.
- **People**: Developers often don't know about DaemonSets and try to write Deployments for infrastructure agents. Surface this during architecture reviews. The "one per node" semantics should be a first-class concept in your team's mental model of K8s workload types.
- **CI/CD**: DaemonSet rollouts on large clusters take time (`maxUnavailable: 1` means N sequential node updates). Budget for this in deployment pipelines for infrastructure agents. Gate on DaemonSet rollout completion the same way you gate on Deployment rollout status: `kubectl rollout status daemonset/<name>`.
- **Operations**: Monitor DaemonSet "number desired" vs. "number ready" — a gap means some nodes don't have the agent running (new node joining, pod crash loop, resource exhaustion). Alert on this. For log forwarders specifically, also monitor buffer fill and forwarding lag — a node's DaemonSet pod can be Running but functionally degraded.
