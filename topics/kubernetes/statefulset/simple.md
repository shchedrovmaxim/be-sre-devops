# StatefulSet — the simple version (the reserved seat analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **A StatefulSet is for workloads that care about their identity. Each pod has a permanent name, a permanent hostname, and its own dedicated storage — and they start and stop in strict order.**

---

## The reserved seat on a train

A Deployment is like a train with open seating. Passengers (pods) can sit anywhere. If a passenger leaves, a new passenger takes any available seat. Seats don't have names. Passengers don't have names. They're interchangeable.

A StatefulSet is like a train with **reserved seats**. Each seat has a name: seat-0, seat-1, seat-2. Each seat has its own luggage locker (a PersistentVolume). If passenger seat-1 leaves the train, **a new passenger will get the same seat-1 with the same locker**. The seat number is the identity.

| In the train world | In the K8s world |
|---|---|
| Reserved seat with a name | Pod with a stable name: `app-0`, `app-1`, `app-2` |
| Each seat's luggage locker | Each pod's PersistentVolumeClaim (its own storage) |
| Boarding in seat order (0, 1, 2...) | Pods start in order: `app-0` first, then `app-1`, then `app-2` |
| Leaving in reverse seat order (2, 1, 0) | Pods terminate in reverse order: `app-2` first, then `app-1` |
| If you break your locker, a replacement passenger gets the same locker | PVCs survive pod deletion — new pod gets the same PVC |
| Seat-1's locker stays empty when seat-1 is vacant | **PVC is NOT deleted when you scale down** (this is a gotcha) |

---

## The two confusing concepts

### 1. The headless Service — how pods find each other

A StatefulSet needs a "headless" Service (one with `clusterIP: None`). This is what creates stable DNS names for each pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  clusterIP: None        # ← headless
  selector:
    app: my-db
```

With this, every pod gets a predictable DNS name:
```
my-db-0.my-db.default.svc.cluster.local
my-db-1.my-db.default.svc.cluster.local
my-db-2.my-db.default.svc.cluster.local
```

A Deployment's pods get random names (`my-api-7f9d4b-abc`). A StatefulSet's pods always have the same name (`my-db-0`). This is what lets a Kafka broker know it is broker-0 and not broker-2, and lets other brokers always reach it at the same address.

### 2. The PVC-stays-after-scale-down gotcha

When you scale a StatefulSet from 3 to 2 replicas, pod `app-2` is terminated. Its PVC is **not deleted**. The data is still there.

This is intentional — you might scale back up, and the pod should get its data back.

But if you scale down permanently and forget about it, you end up with orphaned PVCs quietly accruing cloud storage costs forever.

```bash
# After scaling down, check for orphaned PVCs:
kubectl get pvc -l app=my-db
# You'll still see pvc-my-db-2 even though my-db-2 is gone
```

To clean up: delete the PVCs manually after you've confirmed you no longer need the data.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What problem does StatefulSet solve? | Stable identity (name + DNS + storage) across restarts and rescheduling |
| What's the pod naming convention? | `<statefulset-name>-0`, `<statefulset-name>-1`, ... |
| Why does a StatefulSet need a headless Service? | To create stable DNS names per pod: `pod-0.svc.ns.svc.cluster.local` |
| What is `volumeClaimTemplates`? | A per-pod PVC template — each pod gets its own unique PVC |
| Do PVCs get deleted when pods are deleted? | No. PVCs survive. Even when you scale down. |
| What order do pods start in? | 0, 1, 2... — each waits for the previous to be Ready |
| What order do pods stop in? | Reverse: N-1, N-2, ... 0 |
| When can you bypass the ordered start? | Set `podManagementPolicy: Parallel` — all pods start simultaneously |

---

## Self-test (one question — the killer one)

Out loud:

> **"What problems does StatefulSet solve that Deployment doesn't, and what are the gotchas?"**

**Reference answer (intuitive version):**

"A Deployment treats pods as interchangeable — any pod can replace any other. That's great for stateless services. But databases and message brokers need identity. A Kafka broker needs to know it's broker-0 and have its own log directory. A Postgres replica needs a stable hostname so the primary knows where to send WAL. A StatefulSet gives each pod a stable name, a stable DNS hostname, and its own PersistentVolumeClaim that survives pod restarts.

The three gotchas. **First**: you need a headless Service for the DNS names to work — forget it and pods can't find each other. **Second**: pods start in strict order (0, 1, 2) — pod 1 won't start until pod 0 is Ready. This is correct for most databases but can slow down initial deployment. If you don't need ordering, set `podManagementPolicy: Parallel`. **Third** — and this trips people up: when you scale down, the PVCs are NOT deleted. The data stays on disk. If you're permanently removing replicas, you have to clean up the PVCs manually or you'll accumulate cloud storage costs forever."

---

## Further reading / watching

- **Kubernetes Docs — StatefulSets**: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- **Kubernetes Docs — Headless Services**: https://kubernetes.io/docs/concepts/services-networking/service/#headless-services
- **StatefulSet basics (Kubernetes by Example)**: https://kubernetesbyexample.com/statefulsets/
- Good deep-dive talk: search "StatefulSet internals KubeCon" on YouTube

---

## Next: the deep-dive

When the reserved-seat analogy clicks, jump to [`statefulset.md`](./deep-dive.md). The deep-dive covers:

- Ordered start and stop — exactly why and when it matters
- The headless Service requirement in full detail
- `volumeClaimTemplates` — how PVCs are created and named
- Rolling update partitions (canary-style updates on StatefulSets)
- The "can't roll back if PVC schema changed" trap
- `podManagementPolicy: Parallel` — when to use it
- 4 self-test drills + 4-dimensions framing
