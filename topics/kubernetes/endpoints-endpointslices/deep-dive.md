# Endpoints vs EndpointSlices — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why did Kubernetes introduce EndpointSlices, and what scale problem do they solve?"** — naming the O(N²) update pattern in the legacy model, the per-slice sharding strategy, how kube-proxy and ingress controllers consume them, and the topology-aware routing feature.

> Start with the [simple version](./simple.md) if you haven't read it. The office directory analogy is the spine.

---

## The senior framing — this is a data model problem disguised as a networking problem

The legacy Endpoints object is a good example of "works fine at a hundred pods; falls apart at a thousand." The API design stored all pod IPs in a single object. That's a classic wide-object anti-pattern — any write to any field requires a full object rewrite and re-sync.

The interesting thing: the failure mode isn't that routing breaks. It's that the **control plane gets saturated**. The API server's etcd writes spike. kube-proxy on every node re-syncs its entire iptables chain. CPU goes up, latency goes up, but traffic still flows — just slower. The problem is invisible until it isn't.

EndpointSlices are a pure data model change (sharding into bounded objects) that eliminates the wide-object problem. The routing behavior is identical from the application's perspective.

---

## The legacy Endpoints object — what it looked like and why it hurt

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service       # same name as the Service, always
  namespace: default
subsets:
- addresses:
  - ip: 10.0.1.1
    nodeName: node-1
    targetRef:
      kind: Pod
      name: my-pod-abc123
  - ip: 10.0.1.2
    nodeName: node-2
    targetRef:
      kind: Pod
      name: my-pod-def456
  - ip: 10.0.1.3           # ... and so on for every pod
    nodeName: node-1
  notReadyAddresses:         # pods that are registered but not ready
  - ip: 10.0.1.99
  ports:
  - port: 8080
    protocol: TCP
```

Every write to this object:

1. The endpoint controller serializes the entire object.
2. Writes it to etcd (full object, not a diff).
3. API server pushes the update via watch to every kube-proxy and every other controller watching Endpoints (ingress controllers, service meshes, external-dns, etc.).
4. Each watcher deserializes the full object and processes it.

With a 1,000-pod Service that has 50 pod churns per minute (rolling deploys, autoscaling): **50 full object updates × 500 nodes = 25,000 object syncs per minute, each containing 1,000 pod records.** That's the O(N²) problem.

---

## EndpointSlices — the design

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-xvz2k   # generated name, multiple slices per service
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
endpoints:
- addresses:
  - 10.0.1.1
  conditions:
    ready: true
    serving: true      # serving = ready or gracefully terminating
    terminating: false
  nodeName: node-1
  targetRef:
    kind: Pod
    name: my-pod-abc123
    namespace: default
- addresses:
  - 10.0.1.2
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node-2
  targetRef:
    kind: Pod
    name: my-pod-def456
    namespace: default
  # ... up to 100 endpoints per slice
ports:
- name: http
  port: 8080
  protocol: TCP
```

Key design decisions:

- **Max 100 endpoints per slice** (configurable via `--max-endpoints-per-slice` on the controller-manager).
- **Slices are owned by the Service** (owner reference). Deleting the Service deletes all its slices.
- **One slice change = one object write + one watch event**. 50 pod churns × 500 nodes = 50 slice writes × (the nodes watching the relevant slices). Still bounded.
- **The `serving` condition**: a pod that is terminating but still handling requests has `ready: false, serving: true`. kube-proxy can use this to keep routing to it during graceful shutdown. Legacy Endpoints had no equivalent.

---

## The `EndpointSliceMirroring` controller

Legacy clients (anything using the v1 Endpoints API) would break if Endpoints objects disappeared. The mirroring controller solves this:

- Watches all EndpointSlices for a Service.
- Synthesizes a matching v1 Endpoints object (the old format) and keeps it in sync.
- Caps at 1,000 endpoints in the mirrored object (a hard limit in the API). If you have >1,000 pods, the Endpoints object is **truncated**. EndpointSlices have no such limit.

This means: `kubectl get endpoints my-service` still works for anything under 1,000 pods. Above 1,000 pods, only `kubectl get endpointslices` gives the full picture. This is a real gotcha in large deployments — people check `kubectl get endpoints` and see only 1,000 addresses, think the service is down, when actually there are 1,500 pods and everything is fine.

---

## How kube-proxy consumes EndpointSlices

kube-proxy watches EndpointSlices (not Endpoints) since K8s 1.19+ (enabled by default in 1.21+).

The watch is filtered: kube-proxy on `node-1` receives watch events for all EndpointSlice changes across the cluster. On each event, it:

1. Checks which Service the slice belongs to.
2. Rebuilds its local view of that Service's endpoints (collecting all slices for that Service).
3. Rewrites the iptables/IPVS rules for that Service.

The iptables rewrite is still a full rewrite per Service (not per endpoint), but the trigger — the watch event — now arrives on every pod change, not just on changes to the monolithic object. With IPVS mode, individual endpoint updates are O(1).

---

## Topology-aware routing

EndpointSlices carry topology hints that enable routing preference by zone:

```yaml
endpoints:
- addresses: ["10.0.1.1"]
  conditions:
    ready: true
  hints:
    forZones:
    - name: us-east-1a   # prefer this endpoint for traffic from us-east-1a
  nodeName: node-zone-a
  zone: us-east-1a
```

When enabled (`service.kubernetes.io/topology-mode: auto` annotation on the Service), kube-proxy on nodes in `us-east-1a` will prefer endpoints that have a `forZones` hint for `us-east-1a`. Cross-zone traffic is only used if local endpoints are exhausted.

**Why this matters**: in a multi-AZ cluster, cross-AZ traffic costs money (AWS charges ~$0.01/GB for cross-AZ data transfer) and adds latency. Topology hints can cut both.

**Gotcha**: the hint system only kicks in when endpoint distribution across zones is balanced enough (within 3:1 ratio). If you have 10 pods in us-east-1a and 1 in us-east-1b, the system disables hints and falls back to cluster-wide routing. Watch the `EndpointSliceProxying` metrics to know if hints are active.

---

## What happens to in-flight connections on a slice update?

When a pod IP is removed from an EndpointSlice (pod dying), kube-proxy removes it from the routing table. Existing TCP connections **already established** to that pod are not affected — they continue until the pod process closes the socket or the connection drops. Only **new** connections stop being routed there.

This is why the `preStop: sleep` pattern matters (see pod-lifecycle.md): you want no new connections being established to the dying pod during the drain window. The existing connections then finish naturally.

---

## The interview answer in 60 seconds

> "The original Endpoints object stored all pod IPs for a Service in one API object. Any pod change — new pod, dying pod, readiness failure — triggered a full object rewrite and a watch event pushed to every kube-proxy and every ingress controller on every node. With a large Service and frequent pod churn, this is O(N²): N pods × N watchers × full-object payload. The control plane gets saturated before routing actually breaks.
>
> EndpointSlices shard that data into bounded objects, capped at 100 endpoints each. One pod IP change updates one slice. One slice change sends one watch event. The cost per pod churn is now constant regardless of how many pods the Service has.
>
> Modern consumers — kube-proxy since 1.21, all major ingress controllers, Istio, Linkerd — use EndpointSlices directly. The legacy Endpoints object still exists but is synthesized by a mirroring controller for backward compat. Above 1,000 pods, the mirrored Endpoints is truncated — that's a real gotcha in large deployments.
>
> EndpointSlices also added topology hints for zone-aware routing, and the `serving` condition to handle gracefully-terminating pods — both things the legacy format couldn't express."

---

## Self-test drills

### 1. Why did Kubernetes introduce EndpointSlices, and what scale problem do they solve?

**Reference answer**: Single Endpoints object = O(N²) update cost at scale. EndpointSlices shard to ≤100 endpoints per slice; per-pod-change update cost is O(1). Full answer above in the 60-second version.

### 2. I have a Service with 1,200 pods. A teammate runs `kubectl get endpoints my-service` and says only 1,000 pod IPs are listed. Is the service broken?

**Reference answer**: No, it's a mirroring truncation. The mirroring controller caps the synthesized Endpoints object at 1,000 entries. The actual routing is driven by EndpointSlices, which have no such limit. Run `kubectl get endpointslices -l kubernetes.io/service-name=my-service` to see all 1,200 entries across multiple slice objects.

### 3. What is the `serving` condition on an endpoint, and how is it different from `ready`?

**Reference answer**: `ready` means the pod passed its readiness probe. `serving` means the pod is either ready OR gracefully terminating (still handling requests during shutdown). kube-proxy can use `serving` to keep routing to a terminating pod while it drains, rather than immediately dropping it the moment `deletionTimestamp` is set. Legacy Endpoints had no way to express this distinction.

### 4. What are topology hints and when do they fail to activate?

**Reference answer**: Topology hints are per-endpoint zone annotations in EndpointSlices. kube-proxy on a node in zone A prefers endpoints with a hint for zone A. This reduces cross-AZ traffic costs and latency. Hints are disabled automatically when the endpoint distribution across zones is too unbalanced (>3:1 ratio) — the system falls back to cluster-wide routing to avoid starving a zone. Watch the `EndpointSliceProxying` metrics to know if hints are active.

---

## Further reading

- [K8s docs — EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [K8s docs — Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [KEP-0752: EndpointSlices](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/0752-endpointslices) — the original design KEP, worth reading for the problem statement
- See also: `services.md` (how kube-proxy uses endpoints), `ingress-vs-gateway-api.md` (L7 routing above Services)

---

## The 4 dimensions (senior framing)

- **Tech**: EndpointSlices shard to ≤100 endpoints; `serving` condition for graceful drain; topology hints for zone-local routing; mirroring controller truncates at 1,000 for legacy Endpoints. kube-proxy consumes slices by default since 1.21.
- **People**: the "1,000-pod truncation" gotcha causes unnecessary incident noise. Document it in your runbook: "If `kubectl get endpoints` shows exactly 1,000 entries, check endpointslices for the real count." Also: engineers use `kubectl get endpoints` reflexively; add a note to your team's debugging guide to use `get endpointslices` as the primary source of truth.
- **CI/CD**: if you're building a custom controller or operator that reads endpoint data, use the `discovery.k8s.io/v1` EndpointSlice API, not `v1/Endpoints`. Operators built on the legacy API will silently stop seeing endpoints above 1,000. Add a lint check or review gate for any controller code using the old Endpoints client.
- **Operations**: monitor API server write throughput per object type. A spike in `endpoints` object write latency at pod-churn time is the legacy-Endpoints scale symptom. With EndpointSlices, the write pattern shifts to many small objects — watch `endpointslices` write QPS instead. Topology hints require zone labels on nodes (`topology.kubernetes.io/zone`) — verify this in your node provisioning pipeline.
