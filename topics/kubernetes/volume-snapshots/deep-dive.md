# VolumeSnapshots — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through how you'd build a daily snapshot-based backup story for a stateful workload using only K8s primitives."** — covering the three CRDs, driver requirements, snapshot → PVC restore flow, the application-consistency caveat, Velero integration, cost, and cross-namespace limitations.

> Start with the [simple version](./simple.md) if you haven't read it. The save-game analogy is the spine.

---

## The senior framing — snapshots are infrastructure backups, not application backups

The distinction matters enormously:

**Infrastructure backup** (what VolumeSnapshots give you): a point-in-time copy of the block device. If the disk fails or the PVC is accidentally deleted, you can restore to any snapshot point. Fast, cheap, no understanding of the application.

**Application backup** (what you actually need for databases): a consistent, recoverable state of the application's data. For a database, this means all committed transactions are captured, no partial writes, the backup is usable without crash recovery. This requires the application to participate in the backup process.

VolumeSnapshots do the first. They're a building block for the second, but only if you add application-level quiescing.

This is the gotcha that trips up junior/mid engineers: they see "volume snapshot" and think "database backup." Senior engineers say "crash-consistent backup" — which is the correct framing.

---

## The CRD model

Three custom resource definitions (CRDs), deployed separately from core K8s (install via the CSI snapshot controller and snapshot CRDs):

```
VolumeSnapshotClass  →  like StorageClass: which driver, how to behave
VolumeSnapshot       →  like PVC: your request for a snapshot
VolumeSnapshotContent → like PV: the actual snapshot object (auto or pre-provisioned)
```

### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete   # Delete: remove underlying snapshot when VSC is deleted
                         # Retain: keep underlying snapshot even if VSC is deleted
parameters:
  # Driver-specific. For EBS: no extra parameters needed.
  # For Ceph: may include pool, csi.storage.k8s.io/snapshotter-secret-name, etc.
```

`deletionPolicy` controls what happens to the VolumeSnapshotContent (and the underlying snapshot) when you delete the VolumeSnapshot:

- **Delete**: deletes the VolumeSnapshotContent and the underlying cloud snapshot. Common for automatically-managed snapshots.
- **Retain**: keeps the VolumeSnapshotContent and the cloud snapshot even after the VolumeSnapshot object is deleted. Use when you want to keep snapshots beyond their K8s lifecycle (e.g., for compliance).

### VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-daily-2024-01-15
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: db-data-0    # the PVC to snapshot
    # OR, to pre-provision from an existing cloud snapshot:
    # volumeSnapshotContentName: existing-content-name
status:
  readyToUse: true          # set when snapshot is complete and usable
  restoreSize: 20Gi         # size of the original volume
  boundVolumeSnapshotContentName: snapcontent-uuid   # the associated VSC
  creationTime: "2024-01-15T02:00:00Z"
  error: null               # set if snapshot failed
```

### VolumeSnapshotContent

Auto-created by the snapshot controller. You don't usually write this by hand.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: snapcontent-uuid
spec:
  volumeSnapshotClassName: csi-aws-vsc
  driver: ebs.csi.aws.com
  deletionPolicy: Delete
  source:
    volumeHandle: vol-0abc123def456789    # the cloud volume ID that was snapshotted
  volumeSnapshotRef:
    name: db-daily-2024-01-15
    namespace: production
status:
  snapshotHandle: snap-0xyz789abc012     # the cloud snapshot ID (EBS snapshot ID)
  readyToUse: true
  creationTime: 1705280400000000000      # nanoseconds epoch
  restoreSize: 21474836480               # bytes
```

---

## Driver requirements

Not all CSI drivers support snapshots. The driver must implement the `CREATE_DELETE_SNAPSHOT` controller capability.

```bash
# Check if the driver supports snapshots
kubectl get csidriver ebs.csi.aws.com -o jsonpath='{.spec.volumeLifecycleModes}'
# Should include: Persistent

# More direct: check the driver docs or look for VolumeSnapshotClass examples in the driver repo
```

Also required: the snapshot controller and CRDs must be installed (they're separate from the CSI driver itself):

```bash
# Install snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml

# Install the snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

---

## The snapshot → PVC restore flow

When you create a PVC with a snapshot as the dataSource:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-restored
  namespace: production
spec:
  storageClassName: gp3-encrypted
  dataSource:
    name: db-daily-2024-01-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi   # must be >= restoreSize from the snapshot
```

The CSI flow:

1. `external-provisioner` sees the new PVC with a `dataSource`.
2. Calls `CreateVolume` on the CSI controller, passing `volume_content_source.snapshot.snapshot_id = snap-0xyz789abc012`.
3. The CSI driver calls the cloud API (AWS: `ec2:CreateVolume --snapshot-id snap-0xyz789abc012`).
4. A new cloud volume is created, pre-populated with snapshot data.
5. A PV is created for the new volume. PVC becomes Bound.

**Restore time**: depends on the cloud provider. AWS EBS snapshots restore lazily — the volume is available immediately, but blocks are fetched from S3 on first access. Reads to unrestored blocks have higher latency. For time-sensitive restores, pre-warm the volume by reading all blocks before starting the application.

---

## Cross-namespace snapshot restore — the limitation

VolumeSnapshots are namespace-scoped. The VolumeSnapshot must be in the **same namespace** as the PVC that uses it as a dataSource.

To restore a snapshot from namespace A into namespace B:

1. You must copy the VolumeSnapshotContent (cluster-scoped object) to the target namespace.
2. OR: delete the VolumeSnapshot in namespace A (with `deletionPolicy: Retain` to keep the VolumeSnapshotContent), then pre-provision a new VolumeSnapshot in namespace B pointing to the existing VolumeSnapshotContent.
3. OR: use a backup tool like Velero, which handles cross-namespace restores natively.

This limitation is real and bites teams doing cross-environment (prod → staging) data copies. Velero is the practical solution.

---

## Building a daily snapshot CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-snapshot-daily
  namespace: production
spec:
  schedule: "0 2 * * *"     # 2am daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa   # needs create/list/delete on VolumeSnapshots
          restartPolicy: OnFailure
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              DATE=$(date +%Y-%m-%d)
              SNAPSHOT_NAME="db-daily-${DATE}"

              # Create the snapshot
              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: ${SNAPSHOT_NAME}
                namespace: production
              spec:
                volumeSnapshotClassName: csi-aws-vsc
                source:
                  persistentVolumeClaimName: db-data-0
              EOF

              # Wait for readyToUse
              kubectl wait volumesnapshot/${SNAPSHOT_NAME} \
                --for=jsonpath='{.status.readyToUse}'=true \
                --timeout=10m -n production

              # Delete snapshots older than 7 days
              kubectl get volumesnapshot -n production \
                --sort-by=.metadata.creationTimestamp \
                -o name | head -n -7 | xargs -r kubectl delete -n production
```

**RBAC for the snapshot ServiceAccount**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-manager
  namespace: production
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["create", "get", "list", "delete", "watch"]
```

---

## Application-consistent snapshots: the quiesce pattern

For databases, "crash-consistent" (what a plain VolumeSnapshot gives you) is usually good enough — the database can recover from a crash point. But "application-consistent" (all committed transactions included, no crash recovery needed) is better.

**Postgres quiesce pattern**:

```yaml
# Use a Velero pre-backup hook, or build your own with a pre-snapshot Job:
containers:
- name: quiesce-pre
  image: postgres:15
  command:
  - psql
  - -c
  - "CHECKPOINT;"   # flush dirty pages to disk
  env:
  - name: PGPASSWORD
    valueFrom:
      secretKeyRef: ...
```

After the CHECKPOINT, take the snapshot immediately. The WAL is in a clean state; no crash recovery needed on restore.

**MySQL/InnoDB**: `FLUSH TABLES WITH READ LOCK` then snapshot then `UNLOCK TABLES`.

**MongoDB**: `db.fsyncLock()` then snapshot then `db.fsyncUnlock()`.

**Velero** can call these via pre/post hooks defined in annotations on the pod:

```yaml
# On the database pod:
annotations:
  pre.hook.backup.velero.io/command: '["/bin/sh", "-c", "psql -c CHECKPOINT"]'
  post.hook.backup.velero.io/command: '[]'
```

---

## Cost considerations

VolumeSnapshot storage cost = the cloud provider's snapshot storage cost.

AWS EBS: snapshots are stored in S3 (deduped, compressed). Incremental after the first. Cost is typically $0.05/GiB-month. A 100 GiB database with 7-day retention: ~7 snapshots × 100 GiB (first) + incremental delta for subsequent ones. If your data changes slowly, the weekly storage cost might be 100 GiB + 6 × 5 GiB delta = 130 GiB × $0.05 = $6.50/month.

**The storage class for the snapshot itself**: snapshots exist in the cloud provider's snapshot store, not in a K8s StorageClass. The VolumeSnapshotClass points to the CSI driver, not a StorageClass.

**Cost trap**: snapshots with `deletionPolicy: Retain` accumulate forever after the K8s object is deleted. You'll pay for cloud snapshots that have no corresponding K8s object. Audit: `aws ec2 describe-snapshots --owner self` and compare against `kubectl get volumesnapshotcontents`.

---

## Velero — the practical integration

Velero uses VolumeSnapshots as its storage backend (in CSI mode) and adds:

- **Scheduled backups**: managed Backup resources with TTL.
- **Namespace filtering**: backup specific namespaces.
- **Pre/post hooks**: quiesce databases before snapshot.
- **Cross-namespace restore**: restore into a different namespace.
- **Resource filtering**: include/exclude specific resource types.
- **S3 offload**: backup Kubernetes resource manifests (Deployments, ConfigMaps, Secrets) to S3 alongside volume snapshots.

```yaml
# Velero Schedule (daily backup with 7-day TTL)
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-prod-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 168h0m0s      # 7 days
    includedNamespaces:
    - production
    snapshotVolumes: true
    hooks:
      resources:
      - name: db-quiesce
        includedNamespaces: [production]
        labelSelector:
          matchLabels:
            app: postgres
        pre:
        - exec:
            container: postgres
            command: ["/bin/sh", "-c", "psql -c 'CHECKPOINT;'"]
            timeout: 30s
```

---

## The interview answer in 60 seconds

> "A daily snapshot backup story using K8s primitives: three resources. VolumeSnapshotClass defines the driver (ebs.csi.aws.com) and deletion policy. A CronJob runs daily, creates a VolumeSnapshot resource pointing to the database PVC, waits for readyToUse: true, then prunes snapshots older than the retention window. Restore is a new PVC with a dataSource referencing the snapshot — the CSI driver creates a new cloud volume pre-populated from the snapshot.
>
> The critical caveat: VolumeSnapshots are crash-consistent, not application-consistent. The snapshot captures the block device state at a point in time. If there was an in-flight database transaction, the restored database has to run crash recovery. For true consistency, you quiesce the database first — for Postgres, that's `CHECKPOINT` before the snapshot fires.
>
> For anything beyond the basics — cross-namespace restores, pre/post hooks, S3 backup of K8s manifests — use Velero. It uses VolumeSnapshots under the hood but adds all the operational scaffolding.
>
> Cost: snapshots are cheap relative to full volume clones (incremental, deduplicated), but they accumulate with Retain policy. Audit cloud snapshots against K8s VolumeSnapshotContent objects to catch orphaned ones."

---

## Self-test drills

### 1. Walk me through how you'd build a daily snapshot-based backup story for a stateful workload using only K8s primitives.

**Reference answer**: full coverage in the 60-second answer. VolumeSnapshotClass → CronJob creates VolumeSnapshot → waits for readyToUse → prune old ones. Restore via PVC dataSource. Quiesce for app-consistency. Velero for production use.

### 2. My VolumeSnapshot shows readyToUse: false for 10 minutes. What do I check?

**Reference answer**: Check the snapshot controller pod logs (in the snapshot-controller namespace or kube-system). Check `kubectl describe volumesnapshot <name>` for status.error. Check the CSI driver's external-snapshotter container logs in the csi-controller pod. Common causes: cloud snapshot API error, IAM permission missing for the CSI driver, the VolumeSnapshotClass specifies the wrong driver name.

### 3. I deleted a VolumeSnapshot but the EBS snapshot in AWS still exists and I'm paying for it. Why?

**Reference answer**: The VolumeSnapshotClass has `deletionPolicy: Retain`. Deleting the VolumeSnapshot K8s object keeps the VolumeSnapshotContent (and the underlying cloud snapshot). You need to manually delete the VolumeSnapshotContent to trigger cloud snapshot deletion. Or: change the VolumeSnapshotClass to `deletionPolicy: Delete` for future snapshots, and manually delete the orphaned VolumeSnapshotContent objects.

### 4. Can I restore a VolumeSnapshot from the `production` namespace into the `staging` namespace?

**Reference answer**: Not directly via the dataSource API — the VolumeSnapshot must be in the same namespace as the target PVC. To cross namespaces: use Velero (designed for this), or manually: delete the VolumeSnapshot with `deletionPolicy: Retain` to keep the VolumeSnapshotContent, then pre-provision a new VolumeSnapshot object in the staging namespace pointing to the same VolumeSnapshotContent. Complex and error-prone — Velero is the right tool.

---

## Further reading

- [K8s docs — Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [CSI external-snapshotter GitHub](https://github.com/kubernetes-csi/external-snapshotter)
- [Velero docs](https://velero.io/docs/)
- [AWS EBS snapshot pricing](https://aws.amazon.com/ebs/pricing/)
- See also: `pvc-pv-lifecycle.md` (PVC/PV lifecycle, including Retain policy), `csi-architecture.md` (how CreateVolume from snapshot works at the driver level)

---

## The 4 dimensions (senior framing)

- **Tech**: 3 CRDs (VolumeSnapshotClass, VolumeSnapshot, VolumeSnapshotContent); crash-consistent by default, not application-consistent; `deletionPolicy: Delete|Retain` on VolumeSnapshotClass; restore via PVC dataSource; cross-namespace restore requires Velero or manual pre-provisioning; AWS EBS lazy restore (blocks fetched from S3 on first access, pre-warm for performance).
- **People**: developers often treat VolumeSnapshots as "database backups" without understanding the crash-consistency limitation. During an incident, a team restores from a snapshot expecting clean data and finds the database in an unexpected state (the crash recovery ran and rolled back incomplete transactions). Write a runbook that says: "snapshot restores are crash-consistent. For Postgres, this means the database will run recovery on first start — this is expected and safe, but may take minutes for large WAL replay." Set expectations before the 3am incident, not during it.
- **CI/CD**: automate snapshot testing: after each daily backup, create a test restore PVC from the most recent snapshot, mount it to a validation pod, run basic sanity checks (can we query the database, is the row count plausible), then delete the test PVC. This validates that your snapshots are actually restorable, not just that they were created. Alert on validation failures. Most teams skip this and discover broken backups during an actual incident.
- **Operations**: monitor VolumeSnapshot age and count. Alert on: snapshot older than 25 hours (daily job missed), snapshot count above 10 (retention not working), VolumeSnapshot in error state for >30 min. Audit orphaned cloud snapshots weekly (compare `aws ec2 describe-snapshots` to K8s VolumeSnapshotContents). Cost track: 100 GiB database with 7-day retention in EBS snapshots ≈ $5-10/month — generally acceptable, but multiply by number of databases and namespaces.
