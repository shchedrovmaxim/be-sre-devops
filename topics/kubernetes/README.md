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
| Pod lifecycle — simple version (airplane-deplaning analogy for graceful shutdown) | [`pod-lifecycle-simple.md`](./pod-lifecycle-simple.md) | 2026-06-04 |
| Pod lifecycle — phases, termination sequence, SIGTERM, preStop, probes, PDB, init containers, native sidecars | [`pod-lifecycle.md`](./pod-lifecycle.md) | 2026-06-04 |
| Deployment + ReplicaSet rolling update — simple version | [`deployment-rolling-update-simple.md`](./deployment-rolling-update-simple.md) | 2026-06-05 |
| Deployment + ReplicaSet rolling update mechanics (maxSurge/maxUnavailable, pause/resume, Argo Rollouts) | [`deployment-rolling-update.md`](./deployment-rolling-update.md) | 2026-06-05 |
| StatefulSet — simple version (ordered start/stop, stable IDs) | [`statefulset-simple.md`](./statefulset-simple.md) | 2026-06-05 |
| StatefulSet — volumeClaimTemplates, headless Service, parallel pod mgmt, PVC retention gotcha | [`statefulset.md`](./statefulset.md) | 2026-06-05 |
| DaemonSet — simple version | [`daemonset-simple.md`](./daemonset-simple.md) | 2026-06-05 |
| DaemonSet — node-level workloads, tainted-node tolerations, update strategies | [`daemonset.md`](./daemonset.md) | 2026-06-05 |
| Job + CronJob — simple version | [`job-cronjob-simple.md`](./job-cronjob-simple.md) | 2026-06-05 |
| Job + CronJob — completions, backoffLimit, concurrencyPolicy, startingDeadlineSeconds, ttl, timezone | [`job-cronjob.md`](./job-cronjob.md) | 2026-06-05 |
| HorizontalPodAutoscaler — simple version | [`hpa-simple.md`](./hpa-simple.md) | 2026-06-05 |
| HorizontalPodAutoscaler — v2 model, metrics-server, custom metrics, behavior field, KEDA alternative | [`hpa.md`](./hpa.md) | 2026-06-05 |
| Requests vs limits + QoS classes + eviction — simple version | [`resource-mgmt-requests-limits-simple.md`](./resource-mgmt-requests-limits-simple.md) | 2026-06-05 |
| Requests vs limits + QoS classes + eviction order + kube/system-reserved, OOM ranking | [`resource-mgmt-requests-limits.md`](./resource-mgmt-requests-limits.md) | 2026-06-05 |
| priorityClass + preemption — simple version | [`priority-preemption-simple.md`](./priority-preemption-simple.md) | 2026-06-05 |
| priorityClass + preemption — preemption sequence, victim selection, PDB-doesn't-block-preemption trap | [`priority-preemption.md`](./priority-preemption.md) | 2026-06-05 |
| Service types (ClusterIP / NodePort / LB / ExternalName / Headless) — simple version | [`services-simple.md`](./services-simple.md) | 2026-06-05 |
| Service types — virtual IPs, externalTrafficPolicy, sessionAffinity, headless+StatefulSet | [`services.md`](./services.md) | 2026-06-05 |
| Endpoints vs EndpointSlices — simple version | [`endpoints-endpointslices-simple.md`](./endpoints-endpointslices-simple.md) | 2026-06-05 |
| Endpoints vs EndpointSlices — scale problem, sharding, topology-aware routing | [`endpoints-endpointslices.md`](./endpoints-endpointslices.md) | 2026-06-05 |
| Ingress vs Gateway API — simple version | [`ingress-vs-gateway-api-simple.md`](./ingress-vs-gateway-api-simple.md) | 2026-06-05 |
| Ingress vs Gateway API — Gateway/HTTPRoute model, role separation, ReferenceGrant, transition | [`ingress-vs-gateway-api.md`](./ingress-vs-gateway-api.md) | 2026-06-05 |
| CSI architecture — simple version | [`csi-architecture-simple.md`](./csi-architecture-simple.md) | 2026-06-05 |
| CSI architecture — controller/node plugins, sidecars, Create/Publish sequence, failure modes | [`csi-architecture.md`](./csi-architecture.md) | 2026-06-05 |
| StorageClass + dynamic provisioning — simple version | [`storageclass-simple.md`](./storageclass-simple.md) | 2026-06-05 |
| StorageClass — WaitForFirstConsumer vs Immediate, reclaim policy, default-class trap | [`storageclass.md`](./storageclass.md) | 2026-06-05 |
| PVC / PV lifecycle + reclaim policies — simple version | [`pvc-pv-lifecycle-simple.md`](./pvc-pv-lifecycle-simple.md) | 2026-06-05 |
| PVC / PV lifecycle — bind phases, Retain/Delete reclaim, StatefulSet retention, orphaned PV recovery | [`pvc-pv-lifecycle.md`](./pvc-pv-lifecycle.md) | 2026-06-05 |
| Volume expansion — simple version | [`volume-expansion-simple.md`](./volume-expansion-simple.md) | 2026-06-05 |
| Volume expansion — driver support, online vs offline resize, 2-phase resize, partial-resize state | [`volume-expansion.md`](./volume-expansion.md) | 2026-06-05 |
| VolumeSnapshot CRDs — simple version | [`volume-snapshots-simple.md`](./volume-snapshots-simple.md) | 2026-06-05 |
| VolumeSnapshot CRDs — Snapshot/SnapshotContent/SnapshotClass, restore flow, Velero, app-consistency | [`volume-snapshots.md`](./volume-snapshots.md) | 2026-06-05 |
| RBAC — simple version | [`rbac-simple.md`](./rbac-simple.md) | 2026-06-05 |
| RBAC — Role/ClusterRole/Bindings, aggregationRule, can-i, wildcard-verb smell, cloud-IAM integration | [`rbac.md`](./rbac.md) | 2026-06-05 |
| ServiceAccounts + token projection — simple version | [`serviceaccounts-simple.md`](./serviceaccounts-simple.md) | 2026-06-05 |
| ServiceAccounts — bound projected tokens, automountServiceAccountToken, IRSA/Workload Identity bridge | [`serviceaccounts.md`](./serviceaccounts.md) | 2026-06-05 |
| Pod Security Standards — simple version | [`pod-security-standards-simple.md`](./pod-security-standards-simple.md) | 2026-06-05 |
| Pod Security Standards — Privileged/Baseline/Restricted, namespace labels, Kyverno as successor | [`pod-security-standards.md`](./pod-security-standards.md) | 2026-06-05 |
| securityContext — simple version | [`security-context-simple.md`](./security-context-simple.md) | 2026-06-05 |
| securityContext — runAsNonRoot, readOnlyRootFilesystem, caps drop, seccomp, fsGroup, Kyverno enforcement | [`security-context.md`](./security-context.md) | 2026-06-05 |
| Image pull secrets — simple version | [`image-pull-secrets-simple.md`](./image-pull-secrets-simple.md) | 2026-06-05 |
| Image pull secrets — dockerconfigjson, IRSA+ECR, kubelet credential providers, pull-rate-limits | [`image-pull-secrets.md`](./image-pull-secrets.md) | 2026-06-05 |
| etcd basics — simple version | [`etcd-basics-simple.md`](./etcd-basics-simple.md) | 2026-06-05 |
| etcd basics — Raft, quorum math, NVMe + fsync latency, 8GB quota, snapshot/defrag, managed-K8s hiding | [`etcd-basics.md`](./etcd-basics.md) | 2026-06-05 |
| Control plane components — simple version | [`control-plane-simple.md`](./control-plane-simple.md) | 2026-06-05 |
| Control plane — apiserver/scheduler/CM/CCM, request flow, controller pattern, HA | [`control-plane.md`](./control-plane.md) | 2026-06-05 |
| K8s upgrade flow — simple version | [`upgrade-flow-simple.md`](./upgrade-flow-simple.md) | 2026-06-05 |
| K8s upgrade flow — version skew, CP→nodes, deprecation API check, addon compatibility, PDB role | [`upgrade-flow.md`](./upgrade-flow.md) | 2026-06-05 |
| kubeadm vs managed K8s — simple version | [`kubeadm-vs-managed-simple.md`](./kubeadm-vs-managed-simple.md) | 2026-06-05 |
| kubeadm vs managed K8s — ownership trade-offs, sovereign cloud, bare-metal, cluster-api/karmada | [`kubeadm-vs-managed.md`](./kubeadm-vs-managed.md) | 2026-06-05 |

## Why this matters

K8s is table stakes. Senior interviews probe internals: "what does the scheduler do when this YAML is applied?", "walk me through what happens when a pod OOMs," "explain how RBAC actually resolves a request."

## Hands-on environment

- `kind` or `minikube` for a local cluster
- EKS / GKE for cloud-flavored work
- Local tools: `kubectl`, `k9s`, `stern`
