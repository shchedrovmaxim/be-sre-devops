# StatefulSet — ordered start/stop, stable IDs, storage (deep-dive)

> **Goal**: by the end you can answer the killer question — **"What problems does StatefulSet solve that Deployment doesn't, and what are the gotchas?"** — naming stable identity, ordered lifecycle, per-pod PVCs, headless Service DNS, the PVC-survives-scale-down trap, partition-based updates, and the PVC schema rollback problem.

> Start with the [simple version](./simple.md) if you haven't read it. The reserved-seat-on-a-train analogy is the spine of this whole topic.

---

## Why StatefulSet exists

A Deployment treats pods as **fungible**: any pod can replace any other, IDs are random, and data can be stored on any volume. That's correct for stateless apps.

Distributed systems like databases and message queues need **identity**:

- A Kafka broker needs to know it's broker-0, not broker-1, because each broker owns specific partitions.
- A PostgreSQL replica needs a stable hostname so the primary knows where to send replication traffic.
- An Elasticsearch data node needs its own shard storage that survives rescheduling.

StatefulSet provides exactly three things that Deployment doesn't:

1. **Stable, persistent identity**: `pod-name-0`, `pod-name-1` — never changes, even after restarts.
2. **Stable network identity**: predictable DNS hostname per pod via headless Service.
3. **Stable per-pod storage**: one PVC per pod, outliving the pod.

---

## The headless Service — why it's non-negotiable

A normal Service (`clusterIP: some-IP`) load-balances traffic across all matching pods. With a StatefulSet you want to reach **individual** pods, not a random one.

A headless Service (`clusterIP: None`) skips the virtual IP and instead creates DNS A records that point directly to pod IPs:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db               # this name is the serviceName in the StatefulSet
spec:
  clusterIP: None            # ← what makes it headless
  selector:
    app: my-db
  ports:
  - port: 5432
```

Now DNS resolves individual pods:

```
my-db-0.my-db.default.svc.cluster.local  →  pod-0-IP
my-db-1.my-db.default.svc.cluster.local  →  pod-1-IP
my-db-2.my-db.default.svc.cluster.local  →  pod-2-IP
```

The pattern: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.

Compare to a Deployment's pod DNS: no stable DNS per pod (pods get random names; no DNS record per pod by default).

**What happens if you forget the headless Service?** The StatefulSet still creates pods, but the stable DNS names don't exist. Peers can't find each other by name. Distributed consensus (Raft, ZAB, Postgres replication) breaks.

The headless Service must be created **before** the StatefulSet, and its name must match `spec.serviceName` in the StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-db
spec:
  serviceName: "my-db"       # ← must match the headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: my-db
  template: ...
  volumeClaimTemplates: ...
```

---

## Ordered start and stop

### Start order

Pods are created sequentially: `my-db-0`, then `my-db-1`, then `my-db-2`. Each pod must reach `Running` and pass its readiness probe before the next one starts.

Why this matters for a database cluster:

1. `my-db-0` starts → becomes primary (or leader in Raft).
2. `my-db-1` starts → connects to `my-db-0.my-db` → joins as replica. `my-db-0` must be up and reachable.
3. `my-db-2` starts → same.

If pods started in parallel (like a Deployment), `my-db-1` might try to connect to `my-db-0` before it's ready — cluster initialization fails or requires complex retry logic.

### Stop order

Pods are terminated in **reverse** order: `my-db-2` first, then `my-db-1`, then `my-db-0`. Each pod waits for the previous one to fully terminate before the next starts.

Why: in most replication schemes, the leader/primary is pod-0. Stopping replicas first, then the primary is the graceful shutdown order. The reverse is also correct: never stop the primary while replicas are still running (they'd lose their sync source).

### `podManagementPolicy: Parallel`

When order doesn't matter (e.g., a fleet of stateful workers that don't coordinate with each other), set:

```yaml
spec:
  podManagementPolicy: Parallel
```

This makes StatefulSet behave like a Deployment for start/stop ordering: all pods start and stop simultaneously. Faster initial deployment, faster scale operations.

Default is `OrderedReady`. Only change it when you've confirmed your app doesn't require ordered startup.

---

## `volumeClaimTemplates` — per-pod storage

This is the killer feature. Each pod gets its own PVC, automatically provisioned by the StatefulSet controller:

```yaml
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "gp3"
      resources:
        requests:
          storage: 10Gi
```

The PVC names follow the pattern `<template-name>-<pod-name>`:
- `data-my-db-0`
- `data-my-db-1`
- `data-my-db-2`

When pod `my-db-1` is deleted and rescheduled (possibly on a different node), K8s attaches the same PVC `data-my-db-1` to the new pod. The data is there. The pod picks up exactly where it left off.

Compare to a Deployment with a shared PVC: all pods would fight over the same PVC (usually not possible with `ReadWriteOnce`), or you'd use `ReadWriteMany` (NFS/EFS) with all its consistency headaches.

---

## The PVC-survives-scale-down gotcha

This is a **real production gotcha** that trips teams regularly.

Scenario: you scale your StatefulSet from 3 to 2. Pod `my-db-2` is terminated. What happens to `data-my-db-2`?

**Nothing. The PVC is not deleted.**

This is intentional — if you scale back up to 3, pod `my-db-2` should get its data back. The StatefulSet controller deliberately does not delete PVCs when scaling down.

The problem: if you permanently scale down, you now have an orphaned PVC sitting on a cloud volume (AWS EBS, GCP PD, etc.) costing you money every month. The PVC has no pod, so nothing is using it, but it still exists and gets billed.

```bash
# After scaling down, always check for orphaned PVCs
kubectl get pvc -n my-namespace

# NAME             STATUS   VOLUME             CAPACITY   ...
# data-my-db-0     Bound    pvc-xxx            10Gi
# data-my-db-1     Bound    pvc-yyy            10Gi
# data-my-db-2     Bound    pvc-zzz            10Gi   ← orphaned! pod gone, PVC stays

# Clean up manually after confirming data is not needed:
kubectl delete pvc data-my-db-2 -n my-namespace
```

**K8s 1.27+ adds a `persistentVolumeClaimRetentionPolicy` field** (beta) to control this:

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain    # or Delete — what happens when StatefulSet is deleted
    whenScaled: Retain     # or Delete — what happens when scaling down
```

Default is `Retain` for both, matching historical behavior. Setting `whenScaled: Delete` automates PVC cleanup on scale-down — useful if you know you never want to scale back up without fresh storage.

---

## Rolling update partitions — StatefulSet canary

StatefulSets support partition-based rolling updates. Instead of updating all pods, you update only pods with index >= the partition number:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2          # only update pods with index >= 2
```

With 3 replicas and `partition: 2`, only `my-db-2` gets updated. `my-db-0` and `my-db-1` stay on the old version.

This is the StatefulSet canary pattern:
1. Set partition to N-1 (only update last pod)
2. Validate the updated pod
3. Reduce partition to N-2 (update second-to-last)
4. Validate
5. Reduce to 0 (update all)

To "freeze" an update mid-way (block all updates), set `partition` equal to the number of replicas. No pod index would satisfy `>= replicas`, so nothing updates.

---

## The "can't roll back if PVC schema changed" trap

This is a subtle but important gotcha.

Scenario: you upgrade your StatefulSet to a new image that runs a DB migration changing the data schema (e.g., drops a column, changes an enum). The migration runs in an init container and modifies the data in the PVCs. Then you discover the new version has a bug and run `kubectl rollout undo`.

Problem: the rollback brings back the old image. But the data in the PVCs has already been migrated — the old image doesn't understand the new schema. The old pod starts, tries to read the data, fails.

**This is why StatefulSet rollbacks are dangerous for stateful apps.** You can roll back the pod spec, but you cannot automatically roll back the PVC data.

Prevention strategies:
- Always use **forward-only, backward-compatible migrations** (add columns, never drop them until the old version is completely gone).
- Test rollback in staging before any production migration.
- Have a data backup before any schema-changing rollout.

---

## Complete annotated YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  clusterIP: None          # headless — creates per-pod DNS records
  selector:
    app: my-db
  ports:
  - port: 5432
    name: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-db
spec:
  serviceName: "my-db"     # must match the headless Service
  replicas: 3
  podManagementPolicy: OrderedReady   # default; use Parallel if ordering not needed
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0         # update all pods; set to N to freeze updates
  selector:
    matchLabels:
      app: my-db
  template:
    metadata:
      labels:
        app: my-db
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DATA
          value: /var/lib/postgresql/data
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "gp3"
      resources:
        requests:
          storage: 10Gi
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
```

---

## The interview answer in 60 seconds

> "StatefulSets solve three things Deployments can't. First, **stable identity** — pods get names like `my-db-0`, `my-db-1` that never change even after restarts or rescheduling. Second, **stable network identity** — via a headless Service, each pod gets a predictable DNS name (`my-db-0.my-db.default.svc.cluster.local`) so peers can always reach a specific pod. Third, **per-pod storage** — each pod gets its own PVC from `volumeClaimTemplates`, and that PVC survives pod deletion. The same pod gets the same data when it comes back.
>
> The gotchas are: headless Service must exist before the StatefulSet (no Service = no DNS). Pods start in order (0, 1, 2) and stop in reverse — use `podManagementPolicy: Parallel` only if your app doesn't need this. The big one: scaling down does NOT delete PVCs. Orphaned PVCs cost money; you have to clean them up manually. And you can't automatically roll back a StatefulSet if a migration changed the data schema on disk — the old image won't understand the new data format."

---

## Self-test

### 1. What's the difference between a headless Service and a regular Service?

**Reference answer:** A regular Service gets a ClusterIP (virtual IP). Traffic to that IP is load-balanced across pods; you can't target individual pods. A headless Service (`clusterIP: None`) skips the virtual IP and instead creates DNS A records per pod: `pod-name.service-name.namespace.svc.cluster.local → pod-IP`. Lets you reach individual pods by stable hostname.

### 2. I scaled my StatefulSet from 5 to 3. Three months later I'm getting an unexpected cloud bill. What happened?

**Reference answer:** PVCs from pods 3 and 4 still exist (StatefulSet controller never deletes PVCs on scale-down). They're consuming cloud storage with no pod attached. Fix: `kubectl get pvc` to find them, `kubectl delete pvc` to remove after confirming data is no longer needed. Going forward: set `persistentVolumeClaimRetentionPolicy.whenScaled: Delete` (K8s 1.27+) to automate cleanup.

### 3. When would you use `podManagementPolicy: Parallel`?

**Reference answer:** When pods don't need to coordinate their startup with each other. Example: a StatefulSet of 10 stateful worker processes that each own a partition of data (like Kafka consumers) but don't form a cluster and don't depend on each other's readiness during startup. `Parallel` lets them all start at once, cutting deployment time from 10 * readiness-delay to 1 * readiness-delay.

### 4. You want to canary a new version of your StatefulSet before rolling it to all replicas. How?

**Reference answer:** Use `rollingUpdate.partition`. Set partition to `replicas - 1` (e.g., `partition: 2` with 3 replicas). Only the last pod (`my-db-2`) will update. Validate it — check metrics, replication lag, etc. Then reduce partition by 1, validate again, repeat until partition = 0. If anything looks wrong, restore the old partition value to freeze further updates, and manually update the pod spec back.

---

## Further reading / watching

- **Kubernetes Docs — StatefulSets**: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- **Kubernetes Docs — PVC Retention Policy**: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#persistentvolumeclaim-retention
- **"StatefulSets internals" by Tim Hockin** — look for his KubeCon talks
- **Running Kafka on K8s StatefulSets** — the Confluent and Strimzi docs are good real-world examples
- **Operator pattern**: for production databases on K8s, use an Operator (Postgres Operator by Zalando, MongoDB Operator, etc.) — they wrap StatefulSet with day-2 operational logic that StatefulSet itself doesn't provide

---

## The 4 dimensions (senior framing)

- **Tech**: Stable identity via ordered names + headless Service DNS; per-pod PVCs via `volumeClaimTemplates`; ordered start/stop for cluster formation; partition-based canary updates; PVC retention policy (K8s 1.27+) to manage orphaned storage.
- **People**: Devs often assume StatefulSet is "just like Deployment but for databases." It's not. The PVC lifecycle difference, the rollback-with-schema-change trap, and the headless Service requirement all need to be explicitly taught. A team that has never run a production StatefulSet will hit these on day one. Pair-review the first StatefulSet rollout.
- **CI/CD**: StatefulSet rollbacks are NOT safe if migrations ran. Before any StatefulSet upgrade that touches schema: run in staging first, take a backup, have a runbook for manual data rollback. Don't put StatefulSet deploys on the same automatic fast-path as Deployment rollouts.
- **Operations**: Track orphaned PVCs in a weekly audit (a simple `kubectl get pvc --all-namespaces` pipeline). Monitor StatefulSet replica count vs. desired, and replication lag (for DBs) as a SLI. For production databases, strongly prefer an Operator over raw StatefulSets — Operators understand the application's semantics (primary promotion, backup, restore) in ways StatefulSet cannot.
