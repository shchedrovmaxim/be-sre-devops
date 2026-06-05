# VolumeSnapshots — the simple version (the save-game analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **A VolumeSnapshot is a point-in-time copy of a disk. It's like a save point in a video game. You can go back to it — but only to where the game was when you saved, not to a perfectly consistent mid-level state.**

---

## The save-game analogy

You're playing a video game. You hit "Save" at the end of level 3. 

- If you die in level 4, you reload and go back to the start of level 4 (the save point).
- The save captures your inventory, position, and state **at the moment you pressed Save**.
- If you had unsaved progress mid-battle, that's lost.
- The save is a snapshot of the game state, not a live copy of the running game.

| In the game world | In the K8s / storage world |
|---|---|
| Hitting "Save" | Creating a `VolumeSnapshot` |
| The save file | `VolumeSnapshotContent` (the actual snapshot stored in the cloud) |
| The "Save Profile" type (which slot, cloud save vs local) | `VolumeSnapshotClass` |
| Reloading from a save | Restoring: creating a new PVC from the snapshot |
| Unsaved mid-battle progress | In-flight database writes not yet flushed to disk |
| Save-game corruption if you save mid-glitch | Application-inconsistent snapshot (data written mid-transaction) |

The critical gotcha: **a snapshot is point-in-time, not application-consistent.** If your database has a transaction in progress when the snapshot is taken, the snapshot may capture a torn write — half of a transaction written, half not. On restore, the database has to run recovery. For truly consistent backups, you need to quiesce the application first (tell the DB to flush and pause writes, then snapshot, then resume).

---

## The three CRDs you need to know

VolumeSnapshots require three custom resources:

```
VolumeSnapshotClass  →  defines HOW to take snapshots (which driver)
VolumeSnapshot       →  your request ("take a snapshot of this PVC")
VolumeSnapshotContent → the actual snapshot (auto-created, or pre-provisioned)
```

Same pattern as StorageClass → PVC → PV. The class is the template; the snapshot is your request; the content is the thing itself.

---

## Taking a snapshot (3 steps)

**Step 1**: ensure the VolumeSnapshotClass exists:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete   # or Retain
```

**Step 2**: create the snapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup-2024-01-15
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: db-data-0
```

**Step 3**: wait for it to be ready:

```bash
kubectl wait volumesnapshot/db-backup-2024-01-15 \
  --for=jsonpath='{.status.readyToUse}'=true --timeout=5m
```

---

## Restoring from a snapshot

Create a new PVC and reference the snapshot as a `dataSource`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data-restored
spec:
  storageClassName: gp3-encrypted
  dataSource:
    name: db-backup-2024-01-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi   # must be >= the snapshot's original size
```

K8s calls `CreateVolumeFromSnapshot` on the CSI driver. A new cloud disk is created pre-populated with the snapshot data. The PVC becomes Bound. You can use it like any other PVC.

---

## The "not application-consistent" gotcha

This is the most important senior point.

A volume snapshot captures the state of the **block device**. It doesn't know about the application's logic. If your Postgres is in the middle of a commit when the snapshot fires:

- Some pages might be written.
- The write-ahead log (WAL) might be mid-way through being flushed.
- On restore, Postgres will run crash recovery and likely succeed — but the recovered state might not match what you expected.

For truly consistent backups:
1. Tell Postgres to flush: `SELECT pg_start_backup(...)` or `CHECKPOINT`.
2. Take the snapshot.
3. Resume: `SELECT pg_stop_backup()`.

Or use a backup tool like Velero, which can call pre/post snapshot hooks. Or use Postgres's own pg_basebackup for logical backups. VolumeSnapshots are a useful tool but not a complete backup solution on their own.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is a VolumeSnapshot? | A point-in-time copy of a PVC's block device. Created via the CSI snapshot API. |
| What are the 3 CRDs? | `VolumeSnapshotClass` (how to snapshot), `VolumeSnapshot` (your request), `VolumeSnapshotContent` (the actual snapshot data). |
| How do you restore? | Create a new PVC with `.spec.dataSource` pointing to the snapshot. |
| Are snapshots application-consistent? | No. The block device is snapshotted, not the application state. Use app-level quiesce for consistency. |
| Does the snapshot cost money? | Yes — cloud snapshots (EBS snapshots) are stored in S3 and charged per GiB-month. Usually much cheaper than a full volume clone. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through how you'd build a daily snapshot-based backup story for a stateful workload using only K8s primitives."**

**Reference answer (intuitive version):**

"I'd use three K8s primitives: VolumeSnapshotClass (defines the driver to use), VolumeSnapshot resources (one per day), and a CronJob to create them. The CronJob runs daily, creates a VolumeSnapshot pointing to the PVC, and waits for `readyToUse: true`. I'd also add a cleanup CronJob that deletes snapshots older than my retention window.

The big caveat: the snapshots are point-in-time block-level, not application-consistent. For Postgres or MySQL, I'd add a pre-snapshot hook that runs `CHECKPOINT` or suspends writes briefly before the snapshot fires. Velero can orchestrate these hooks. Without that, the snapshots are crash-consistent — usable, but you might lose the last few seconds of commits."

---

## Further reading

- [K8s docs — Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Velero docs](https://velero.io/docs/) — the standard K8s backup tool that uses VolumeSnapshots under the hood
- [Deep-dive: volume-snapshots.md](./deep-dive.md) — covers the full CRD model, deletionPolicy, cross-namespace restore limitations, Velero integration, cost, and the quiesce pattern
