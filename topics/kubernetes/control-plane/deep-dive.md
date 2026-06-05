# Control plane components — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through what each control plane component does, and how a request to `kubectl apply` flows through them."** — naming the 5 components, the decoupled watch pattern, admission webhooks, and HA considerations.

> Start with the [simple version](./simple.md) if you haven't read it. The restaurant kitchen analogy is the spine.

---

## The senior framing — decoupled watch loops are what make K8s resilient

The K8s control plane design principle is **level-based triggering** (not edge-based). Each component watches the apiserver for its resource type and **continuously reconciles actual state with desired state**. If a reconcile fails, it retries. If the component crashes and restarts, it re-watches and re-reconciles. The system heals itself.

This is fundamentally different from an imperative orchestrator that sends commands ("start this container on that machine"). K8s is declarative: you write the desired state; the control plane figures out how to get there and keeps trying.

**Senior interview framing**: "K8s controllers implement the operator pattern. The operator pattern is just 'watch → diff → act,' implemented in a reconcile loop. Everything in the control plane follows this pattern."

---

## kube-apiserver — the single entry point

The apiserver is the only component that:
- Reads from etcd
- Writes to etcd
- Accepts external connections (kubectl, kubelet, controllers)

Everything else talks to the apiserver via HTTP/2 (REST + watch streaming). The apiserver validates, authenticates, authorizes, and stores.

### The request pipeline

Every request to the apiserver passes through:

```
kubectl apply → apiserver
    │
    ├── 1. Transport security (TLS)
    ├── 2. Authentication (who are you?)
    │       - Client cert, bearer token, OIDC JWT, webhook authenticator
    ├── 3. Authorization (can you do this?)
    │       - RBAC, ABAC, Node authorizer, webhook authorizer
    ├── 4. Admission controllers (should we allow this? should we mutate it?)
    │       - MutatingWebhookConfiguration (e.g., Kyverno mutation, Istio sidecar injection)
    │       - ValidatingWebhookConfiguration (e.g., Kyverno validation, PSS enforcement)
    │       - Built-in admission controllers (ResourceQuota, LimitRanger, NamespaceLifecycle, ...)
    ├── 5. Schema validation (is the object's structure valid?)
    ├── 6. Write to etcd
    └── 7. Return the stored object to the caller
```

**Mutating admission runs before validating admission** — mutation first, then validation on the mutated object.

### Watch mechanism

After creating an object, watchers are notified via long-lived HTTP streams:

```bash
# What kubectl watch actually does internally:
GET /api/v1/pods?watch=true&resourceVersion=12345
# The apiserver streams events (ADDED, MODIFIED, DELETED) as they happen
```

This is how every controller stays updated — they maintain a watch stream to the apiserver. The informer pattern in client-go caches these events locally and triggers reconcile functions.

### HA apiserver

In production, you run multiple apiserver replicas behind a load balancer. Each is stateless (etcd holds the state). Any replica can handle any request.

```
kubectl → load balancer (NLB, HAProxy, etc.)
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
apiserver-1  apiserver-2  apiserver-3
    └───────────┴───────────┘
                │
               etcd cluster (3 or 5 members)
```

Managed K8s (EKS, GKE, AKS): the cloud provider runs multiple apiserver replicas. You don't manage this.

---

## etcd — the state store

See `etcd-basics.md` for the full deep-dive.

Key point for the control plane discussion: **etcd is the only persistent store**. All other components are stateless. etcd stores:
- All K8s API objects (pods, services, deployments, secrets, configmaps, ...)
- API server bootstrap data
- RBAC policies
- Lease objects (leader election)

The apiserver uses **optimistic concurrency** (resource versions) to handle concurrent writes. Every object in etcd has a `resourceVersion`. If two components try to update the same object simultaneously, the second write fails because the resourceVersion is stale — the component must re-read and retry.

---

## kube-scheduler — the pod placement engine

The scheduler watches for pods with `spec.nodeName = ""`. For each such pod, it runs a pipeline:

```
Filtering phase: eliminate nodes that can't run the pod
    - Not enough CPU/memory (requests vs node allocatable)
    - Taints that the pod doesn't tolerate
    - Node affinity requirements not met
    - Pod affinity/anti-affinity rules not met
    - TopologySpreadConstraint violations

Scoring phase: rank remaining nodes
    - Least requested (spread load)
    - Node affinity preference score
    - Image locality (node already has the image cached)
    - Topology spread preferences

Result: pick the highest-scoring node
    - Write spec.nodeName to the pod object in etcd
```

The scheduler itself does nothing to start the pod. It just updates `spec.nodeName`. Kubelet on the chosen node notices the assignment and does the actual work.

**Scheduler extensibility**: custom schedulers, scheduler plugins, and `schedulerName` field in pod specs allow routing pods to different scheduler implementations. Karpenter, for example, implements its own scheduling logic outside the main scheduler.

---

## kube-controller-manager — the reconciliation engine

A single binary (`kube-controller-manager`) runs ~40 controllers as goroutines. Each controller:
1. Lists and watches its resource type via the apiserver
2. When something changes, calls its `Reconcile(key)` function
3. The reconcile function computes the diff and makes API calls to close the gap

Key controllers and what they do:

| Controller | Trigger | Action |
|---|---|---|
| **ReplicaSet controller** | ReplicaSet created/updated; Pod created/deleted | Create or delete pods to match `replicas` |
| **Deployment controller** | Deployment created/updated | Create/update ReplicaSets; manage rollout |
| **StatefulSet controller** | StatefulSet created/updated | Create pods in order, with stable names |
| **DaemonSet controller** | DaemonSet created/updated; Node added | Ensure one pod per matching node |
| **Job controller** | Job created | Create pods; track completions |
| **Node controller** | Node stops reporting (kubelet heartbeat) | After grace period: mark pods Unknown, then evict |
| **Endpoints controller** | Pod created/deleted/ready state changes | Update Service's Endpoints object |
| **EndpointSlice controller** | Same as above | Update EndpointSlices (replaces Endpoints in 1.21+) |
| **Namespace controller** | Namespace deleted | Delete all resources in the namespace |
| **ServiceAccount controller** | Namespace created | Create `default` ServiceAccount |
| **PersistentVolume controller** | PVC created | Bind PVC to matching PV; provision if StorageClass used |
| **GC (garbage collector)** | Owner reference chain broken | Delete objects whose owner is gone |
| **TTL controller** | Job/Pod completes | Delete after `ttlSecondsAfterFinished` |
| **CronJob controller** | Time passes | Create Jobs according to schedule |

The controller-manager process also handles leader election using a Lease object in etcd — only one instance is the active leader at a time, even if multiple replicas are running.

---

## cloud-controller-manager — the cloud bridge

Runs controllers that interact with the cloud provider's API. In K8s 1.11+, it's a separate binary from the kube-controller-manager:

| Controller | What it does |
|---|---|
| **Node controller** | Checks cloud API to verify node still exists; if the VM is terminated, removes the node from K8s |
| **Route controller** | Configures routes in the cloud VPC for pod CIDR ranges (some CNIs use this) |
| **Service controller** | Creates/updates cloud load balancers when a Service with `type: LoadBalancer` is created |

On managed clusters (EKS, GKE, AKS), the cloud-controller-manager is run by the provider. This is what makes `type: LoadBalancer` Services just work — the provider's CCM provisions the NLB/ALB/Google Cloud LB for you.

On self-managed clusters with cloud VMs (e.g., kubeadm on EC2), you must configure and run the AWS cloud-controller-manager yourself for LB provisioning to work.

---

## The full `kubectl apply` flow

```
kubectl apply -f deployment.yaml
    │
    ▼
[ kube-apiserver ]
    ├── Auth: is the user's token/cert valid?
    ├── RBAC: can this user create/update Deployments in this namespace?
    ├── MutatingWebhooks: Kyverno adds labels? Istio injects sidecar?
    ├── ValidatingWebhooks: Kyverno enforces securityContext?
    ├── Schema validation: does this Deployment look correct?
    └── Write Deployment to etcd
    │
    ▼
[ Deployment controller (in kube-controller-manager) ]
    Watches: sees new/changed Deployment in etcd
    Action: create or update a ReplicaSet to match
    │
    ▼
[ kube-apiserver ] ← Deployment controller writes ReplicaSet to etcd
    │
    ▼
[ ReplicaSet controller (in kube-controller-manager) ]
    Watches: sees new/changed ReplicaSet
    Action: create N Pod objects (no nodeName yet)
    │
    ▼
[ kube-apiserver ] ← ReplicaSet controller writes Pods to etcd
    │
    ▼
[ kube-scheduler ]
    Watches: sees Pods with no nodeName
    Action: filters nodes, scores, picks best, writes nodeName to pod
    │
    ▼
[ kube-apiserver ] ← scheduler writes nodeName to Pod in etcd
    │
    ▼
[ kubelet on chosen node ]
    Watches: sees a Pod assigned to this node
    Action: pull image → create container → start container
    Reports: updates Pod status (Running, Ready) back to apiserver
    │
    ▼
[ Endpoints/EndpointSlice controller ]
    Watches: sees Pod becomes Ready
    Action: adds pod IP to Service's EndpointSlice
    │
    ▼
[ kube-proxy on all nodes ]
    Watches: sees EndpointSlice updated
    Action: updates iptables/IPVS rules to route Service traffic to new pod IP
```

**Total roundtrip**: typically under 5 seconds for a pod to go from `kubectl apply` to `Running`, on a healthy cluster. Bottlenecks: image pull time, admission webhook latency (Kyverno webhook adds ~10-50ms per pod), etcd write latency.

---

## The interview answer in 60 seconds

> "Five components. **etcd** is the only database — all cluster state lives there. **kube-apiserver** is the only entry point to etcd and the only thing that talks to it; it handles auth, RBAC, admission webhooks (including Kyverno), schema validation, and stores to etcd. **kube-scheduler** watches for pods with no node assigned and picks a node — it writes the node name to the pod spec, but doesn't start the pod. **kube-controller-manager** runs ~40 controllers in one binary; each watches the apiserver for its resource type and reconciles actual vs desired state — the Deployment controller creates ReplicaSets, the ReplicaSet controller creates Pods, the Node controller evicts pods from dead nodes. **cloud-controller-manager** bridges K8s and the cloud API — it provisions load balancers when you create a type: LoadBalancer Service, and removes nodes when cloud VMs are terminated.
>
> The `kubectl apply` flow: apiserver validates and stores the Deployment → Deployment controller creates a ReplicaSet → ReplicaSet controller creates Pods → scheduler assigns Pods to nodes → kubelet on each node starts the containers → Endpoints controller adds the pod IP to the Service → kube-proxy on all nodes updates iptables. Each step is decoupled — they all watch the apiserver, not each other."

---

## Self-test drills

### 1. Walk me through what each control plane component does and how `kubectl apply` flows through them.

**Reference answer**: see the 5-component breakdown and the full flow above. Key: only apiserver talks to etcd; controllers are watch loops; scheduler only assigns nodeName; kubelet starts the container; cloud-CCM handles LBs.

### 2. What happens if the kube-scheduler crashes?

**Reference answer**: existing pods keep running (kubelet manages them). New pods that need placement pile up in Pending state (nodeName not assigned). Once the scheduler restarts, it re-watches and processes the backlog. The cluster is degraded but not broken for existing workloads.

### 3. What is the admission webhook pipeline, and where does Kyverno fit?

**Reference answer**: after RBAC, before writing to etcd. Two phases: mutating webhooks run first (can modify the object), then validating webhooks (can accept or reject, cannot modify). Kyverno implements both: mutating policies inject defaults (add labels, set securityContext fields); validating policies block non-compliant objects. Each webhook is a HTTPS callback from the apiserver to an external service — Kyverno's webhook server must be available, or it becomes a control plane availability dependency.

### 4. What does the Endpoints/EndpointSlice controller do and why does it matter for service traffic?

**Reference answer**: it watches Pod ready state and updates the EndpointSlice (the set of pod IPs backing a Service). kube-proxy then reads EndpointSlices to update iptables rules. If the Endpoints controller is slow or the EndpointSlice controller is lagging, kube-proxy doesn't know about new pods — traffic doesn't reach them. This is the source of the "kube-proxy is eventually consistent" behavior that causes request drops during rolling updates.

---

## Further reading

- [K8s control plane components](https://kubernetes.io/docs/concepts/overview/components/)
- [Controller pattern](https://kubernetes.io/docs/concepts/architecture/controller/)
- [kube-scheduler design](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Admission controllers reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Programming K8s (O'Reilly)](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/) — the definitive guide to writing controllers and operators

---

## The 4 dimensions (senior framing)

- **Tech**: 5 components; only apiserver talks to etcd; scheduler only assigns nodeName; controller-manager is ~40 goroutines running reconcile loops; admission webhooks (mutating before validating) are the extension point for Kyverno and Istio; EndpointSlice controller is the critical path for service traffic; cloud-CCM bridges K8s to cloud APIs.
- **People**: control plane component failures have very different blast radii. Scheduler down: pods pile up pending (noisy but not catastrophic). Controller-manager down: no new pods created (catastrophic for scaling). Apiserver down: `kubectl` stops working (very visible). Etcd down: everything stops. Understanding this matrix helps on-call triage — "what's the minimum we need to recover to serve existing traffic?" (answer: nothing, workloads keep running; control plane is for management, not the data plane).
- **CI/CD**: Kyverno admission webhooks are on the critical path of every deploy. If the Kyverno webhook service is down and `failurePolicy: Fail`, all applies fail. Use `failurePolicy: Ignore` for auditing-only policies, `Fail` for hard enforcement. Test Kyverno webhook availability in your CD health checks. A Kyverno upgrade that breaks the webhook server will stop all your deployments.
- **Operations**: monitor apiserver request latency (p99 should be <1s; spikes indicate etcd or webhook issues), admission webhook latency per webhook (Kyverno adds real latency under load), scheduler queue depth (pending pods count — a growing queue is a sign of scheduling bottleneck), controller work queue depth (each controller has an internal queue; a growing queue means reconciles are slower than events). Alert on these before user impact.
