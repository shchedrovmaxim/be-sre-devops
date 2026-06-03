# Kubernetes

The platform. Every senior SRE role assumes deep K8s. Focus on the **internals** (why a scheduling decision happens, why a pod is OOMKilled) — not just the YAML.

## Recommended order

### Scheduling (⚠️ rejection gap)
1. Taints + tolerations (effects: NoSchedule / PreferNoSchedule / NoExecute)
2. Node affinity (required vs preferred)
3. Pod affinity (co-location) vs anti-affinity (HA spread)
4. `topologySpreadConstraints` — usually cleaner than anti-affinity
5. priorityClass + preemption
6. The mental model: "can pod even run here (taints) → does pod want here (affinity) → spread evenly (topology)"

### Resource management
7. requests vs limits
8. QoS classes (Guaranteed / Burstable / BestEffort) + eviction order
9. CPU throttling vs starvation; memory limits ↔ cgroup `memory.max`

### Pod lifecycle
10. Pod phases + init containers
11. Native sidecars (K8s 1.29+)
12. Probes: liveness / readiness / startup
13. terminationGracePeriodSeconds + SIGTERM
14. PodDisruptionBudget

### Workload controllers
15. Deployment + ReplicaSet (rolling update mechanics)
16. StatefulSet (ordered start/stop, stable IDs, storage)
17. DaemonSet, Job, CronJob
18. HorizontalPodAutoscaler

### Networking & services
19. Service types (ClusterIP / NodePort / LoadBalancer / ExternalName / Headless)
20. Endpoints vs EndpointSlices
21. kube-proxy modes
22. NetworkPolicy
23. Ingress vs Gateway API
24. CoreDNS

### Storage
25. CSI + StorageClass
26. PVC / PV lifecycle, reclaim policies
27. Volume expansion + snapshots

### Security
28. RBAC (Role / ClusterRole / Binding)
29. ServiceAccounts + token projection
30. PodSecurityStandards (Privileged / Baseline / Restricted)
31. securityContext (runAsNonRoot, readOnlyRootFilesystem, caps)

### Cluster ops
32. etcd performance + backup/restore
33. Control plane components
34. Upgrade flow (CP → nodes)

## Files

| Sub-topic | File | Date covered |
|---|---|---|
| Scheduling — taints, affinity, anti-affinity, topology spread + the 3-question mental model | [`scheduling-affinity.md`](./scheduling-affinity.md) | 2026-06-03 |

## Why this matters

K8s is table stakes. Senior interviews probe internals: "what does the scheduler do when this YAML is applied?", "walk me through what happens when a pod OOMs," "explain how RBAC actually resolves a request."

## Hands-on environment

- `kind` or `minikube` for a local cluster
- EKS / GKE for cloud-flavored work
- Local tools: `kubectl`, `k9s`, `stern`
