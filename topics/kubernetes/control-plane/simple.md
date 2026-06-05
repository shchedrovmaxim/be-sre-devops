# Control plane components — the simple version (the restaurant kitchen analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **The control plane is the kitchen of a restaurant. The apiserver is the pass-through window. etcd is the order book. The scheduler decides which chef handles each dish. The controller-manager makes sure everything that's supposed to exist, exists.**

That's the whole system. Everything else is just precision on top.

---

## The restaurant

A busy restaurant:

| Restaurant world | K8s control plane |
|---|---|
| The order-taking window between dining room and kitchen | **kube-apiserver** — the only way in or out; validates every order |
| The order book / ticket rail | **etcd** — the permanent record of every order (past and present) |
| The head chef who assigns dishes to kitchen stations | **kube-scheduler** — assigns pods to nodes |
| The kitchen manager who enforces "we always have 3 soups ready" | **kube-controller-manager** — reconciles actual vs desired state |
| The general manager who handles the dining room's infrastructure (reservations, parking) | **cloud-controller-manager** — provisions AWS/GCP/Azure resources (LBs, nodes) |

---

## The five components (three lines each)

### kube-apiserver

The front door. **Everything talks to the apiserver.** kubectl, the scheduler, the controllers, the kubelets — none of them talk to each other directly. They all talk to the apiserver.

It authenticates every request, runs admission controllers (including Kyverno webhooks), validates the objects, and then stores them in etcd.

### etcd

The database. Not a cache, not a replica — the actual source of truth. Every K8s object lives here.

The apiserver is the only component that reads from and writes to etcd. Nobody else touches etcd directly.

### kube-scheduler

Watches for pods with no node assigned. Picks a node for each pod based on resources, taints, affinity rules, and topology spread. Updates the pod's `spec.nodeName`. That's it.

It doesn't start the pod — that's kubelet's job. The scheduler just decides *where*.

### kube-controller-manager

A single binary running ~40 controllers in goroutines. Each controller watches the apiserver for its resource type and reconciles actual state with desired state.

- **ReplicaSet controller**: "I see 2 pods running but the spec says 3. Create 1 more."
- **Node controller**: "This node hasn't reported in for 5 minutes. Mark its pods as Unknown."
- **Job controller**: "The job completed successfully. Mark it as Complete."

### cloud-controller-manager

The bridge between K8s and your cloud provider's API. Handles:
- Provisioning cloud load balancers when you create a `Service type: LoadBalancer`
- Marking nodes as deleted when a cloud VM is terminated
- Attaching EBS volumes when a PVC is bound

On EKS/GKE/AKS, you don't run this yourself — the cloud runs it for you.

---

## The `kubectl apply` flow in 30 seconds

```
kubectl apply -f deployment.yaml
    │
    ▼
1. kube-apiserver: authenticate (who are you?) → authorize (RBAC: can you create Deployments?)
   → admission (Kyverno: does this meet our policies?) → validate schema
   → write to etcd
    │
    ▼
2. Deployment controller (in kube-controller-manager): sees new Deployment in etcd
   → creates a ReplicaSet
    │
    ▼
3. ReplicaSet controller: sees new ReplicaSet
   → creates N Pod objects (just objects in etcd, not running yet)
    │
    ▼
4. kube-scheduler: sees pods with no nodeName
   → picks a node for each → updates pod's spec.nodeName in etcd
    │
    ▼
5. kubelet on the chosen node: sees a pod assigned to it
   → tells the container runtime to pull the image and start the container
   → reports status back to apiserver
```

The key insight: **each step is decoupled**. The scheduler doesn't know about Deployments. The controller-manager doesn't know about nodes. The kubelet doesn't know about ReplicaSets. They each watch the apiserver for their piece of state and act.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Who talks to etcd? | Only the kube-apiserver. Nothing else. |
| What does the scheduler do? | Assigns pods to nodes. That's it. |
| What are "controllers"? | Watch loops that reconcile actual state with desired state. |
| What does cloud-controller-manager do? | Bridges K8s and the cloud API (LBs, node registration, volumes). |
| What happens if the apiserver is down? | No new pods can be scheduled; `kubectl` stops working. Existing workloads keep running. |
| What happens if etcd is down? | The apiserver can't read or write state. Everything that requires the API stops. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what each control plane component does, and how a request to `kubectl apply` flows through them."**

**Reference answer**: use the 5-step flow above. Key points: apiserver is the single entry point; etcd is the only database; scheduler assigns pods to nodes (doesn't start them); controller-manager runs reconcile loops (desired vs actual); cloud-controller-manager handles cloud resources. The decoupled, watch-based design is what makes K8s resilient.

---

## Further reading

- [K8s control plane components docs](https://kubernetes.io/docs/concepts/overview/components/)
- [Controller pattern](https://kubernetes.io/docs/concepts/architecture/controller/)

---

## Next: the deep-dive

When the kitchen analogy feels obvious, jump to [`control-plane.md`](./deep-dive.md). The deep-dive covers:

- Exactly what each component does at the API level
- The watch/reconcile pattern in code terms
- HA control plane setup (multiple apiserver replicas, etcd quorum)
- How managed K8s exposes (or hides) each component
- The request flow in full detail including admission webhooks
- 4 self-test drills + 4-dimensions framing
