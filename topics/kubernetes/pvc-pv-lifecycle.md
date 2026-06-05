# PVC / PV Lifecycle + Reclaim Policies — the deep-dive

> **Goal**: by the end you can answer the killer question — **"I deleted a PVC and now my StatefulSet pod can't come back. Walk me through what might have happened with reclaim policies."** — covering the 4 PV phases, reclaim policy effects on each, the StatefulSet PVC retention gotcha, orphaned PV recovery, finalizer behavior, and how `kubectl delete pvc --force` differs from normal deletion.

> Start with the [simple version](./pvc-pv-lifecycle-simple.md) if you haven't read it. The apartment rental analogy is the spine.

---

## The senior framing — PVC/PV is a lifecycle contract, not just storage plumbing

Mid-level engineers think of PVC as "how to request a disk." Senior engineers know PVC/PV encodes a lifecycle contract: who owns the data, what happens when the workload dies, and who is responsible for cleanup. Getting this wrong is a data-loss incident waiting to happen.

The three hard questions are:
1. **What happens to the volume when the PVC is deleted?** (reclaimPolicy)
2. **What protects against accidental PVC deletion?** (finalizers + RBAC)
3. **What does a StatefulSet do with PVCs when it's deleted?** (the answer surprises most people)

---

## The full PV state machine

```
              (provisioner creates PV)
                        │
                        ▼
                   [Available]   ← No PVC bound yet
                        │
                        │  (PVC created, binder matches PV to PVC)
                        ▼
                    [Bound]      ← PVC is actively using the PV
                        │
                        │  (PVC deleted)
                        ▼
                   [Released]   ← PVC gone, PV (and data) still exists
                        │
              ┌─────────┴──────────────────┐
              │                            │
              ▼                            ▼
        reclaimPolicy: Delete    reclaimPolicy: Retain
              │                            │
              ▼                            ▼
     Provisioner deletes        PV stays in Released.
     backing volume.            Human must intervene.
     PV object deleted.         PV can be re-used
                                after claimRef cleared.
```

The `Failed` phase is a terminal state entered when reclamation encounters an error — for example, the provisioner tried to delete the backing volume but the cloud API returned an error. Manual intervention is required to either fix the underlying issue or delete the PV manually.

---

## reclaimPolicy in depth

### Delete

```yaml
reclaimPolicy: Delete  # set on StorageClass, inherited by dynamically provisioned PVs
```

When PVC is deleted:
1. PV status → `Released`.
2. The `external-provisioner` sidecar sees the Released PV.
3. It calls `DeleteVolume` on the CSI controller.
4. The CSI driver calls the storage API (e.g., `ec2:DeleteVolume`).
5. The PV object is deleted from etcd.

**Timeline**: deletion of the backing volume is asynchronous. There's a window between "PVC deleted" and "EBS volume actually gone" — the PV still shows `Released` during this window.

**Data loss scenario**: a developer does `kubectl delete pvc db-data` thinking they're freeing space. The EBS volume with production data is deleted. StorageClass had `reclaimPolicy: Delete`. There's no recycle bin.

**Mitigation**: for production data, use `Retain`. Protect PVCs in prod namespaces with RBAC (only cluster-admin can delete PVCs in `prod` namespace). Alert on PVC deletion events.

### Retain

When PVC is deleted:
1. PV status → `Released`.
2. Nothing else happens automatically. The backing volume (EBS volume) survives.
3. The PV object remains in etcd in `Released` state.

**To reuse the PV** (recover the data with a new PVC):

```bash
# Step 1: clear the old claim reference
kubectl patch pv pvc-uuid-1234 -p '{"spec":{"claimRef": null}}'
# PV status → Available

# Step 2: Create a PVC that references this specific PV by name
# (StatefulSet scenario: recreate the PVC with the exact original name)
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data-statefulset-0   # must match what the StatefulSet expects
spec:
  storageClassName: ""           # empty string = use this exact PV, no dynamic provisioning
  volumeName: pvc-uuid-1234      # bind to this specific PV
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
```

**The `Recycle` policy**: deprecated and removed. It used to run `rm -rf /` on the volume before rebinding. Replaced by the VolumeSnapshot-based backup/restore pattern.

---

## The StatefulSet PVC retention gotcha

StatefulSets manage PVCs through their VolumeClaimTemplates. Important behaviors:

**Scaling down does NOT delete PVCs.** Scale from 3 to 0: pods terminate, PVCs survive. Scale back to 3: same PVCs are reused. Same data.

**Deleting the StatefulSet does NOT delete PVCs (default behavior).** You delete the StatefulSet, the pods die, but the PVCs remain. This is intentional — StatefulSets assume you care about the data.

**K8s 1.27+ adds `persistentVolumeClaimRetentionPolicy`** to control this:

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete     # delete PVCs when StatefulSet is deleted
    whenScaled: Retain      # keep PVCs when scaling down (default behavior)
```

Options: `Delete` or `Retain` for each lifecycle event.

**The fresh-start trap**: a developer deletes and recreates a StatefulSet expecting a clean slate. The old PVCs are still there. The new StatefulSet binds to the same PVCs. The old data is back. Surprising if you expected fresh storage.

**To get a truly fresh start**: delete the StatefulSet, then delete the PVCs:

```bash
kubectl delete statefulset my-db
kubectl delete pvc -l app=my-db   # or enumerate by name
```

---

## The finalizer mechanism — what prevents accidental deletion

When a PVC is being used by a pod, Kubernetes adds the finalizer `kubernetes.io/pvc-protection` to the PVC:

```yaml
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
```

Running `kubectl delete pvc` sets `.metadata.deletionTimestamp` but the actual deletion is blocked until the finalizer is removed. The `PVCProtectionController` only removes the finalizer when no running pods reference the PVC.

**Result**: `kubectl delete pvc` on a mounted volume → PVC goes to `Terminating` state → stays there until the pod terminates → then PVC is actually deleted.

**Force deletion**: `kubectl delete pvc --grace-period=0 --force` removes the object from etcd without waiting for the finalizer. **Dangerous**: the mount on the node still exists (at `/var/lib/kubelet/pods/...`). The pod still has the filesystem. But the K8s object is gone. The volume is orphaned from K8s's perspective. The CSI driver may not clean up properly. Don't do this unless you know exactly what you're doing.

---

## Orphaned PV recovery — when a PV is stuck in Released

If `reclaimPolicy: Retain` and you need to recover the volume:

```bash
# 1. Find the PV
kubectl get pv | grep Released

# 2. Note the backing volume ID (for AWS EBS, it's in the PV's .spec.csi.volumeHandle)
kubectl get pv <name> -o jsonpath='{.spec.csi.volumeHandle}'

# 3. Clear the claimRef so the PV becomes Available
kubectl patch pv <name> --type=json \
  -p='[{"op": "remove", "path": "/spec/claimRef"}]'

# 4. PV status → Available. Now you can bind a new PVC to it.
# Create PVC with storageClassName: "" and volumeName: <pv-name>
```

If the backing volume was deleted externally (in AWS console) but the PV object still exists in K8s, the PV status will be `Released` but the volume is gone. You need to delete the PV object manually:

```bash
# Remove the finalizer first if it's blocking deletion
kubectl patch pv <name> -p '{"metadata":{"finalizers":null}}'
kubectl delete pv <name>
```

---

## VolumeSnapshot-based backup/restore

The modern backup pattern uses VolumeSnapshots (the CSI snapshot capability):

```yaml
# Take a snapshot of an existing PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup-2024-01-15
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: db-data-0
```

Restore to a new PVC:

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
      storage: 20Gi
```

See `volume-snapshots.md` for the full snapshot workflow.

---

## The interview answer in 60 seconds

> "When you delete a PVC, what happens depends entirely on the StorageClass's reclaimPolicy. With Delete (the dynamic provisioning default), deleting the PVC also deletes the PV and the backing volume — the data is gone. With Retain, the backing volume survives but the PV moves to Released status — K8s won't bind new PVCs to it until you manually clear the claimRef.
>
> For a StatefulSet, the gotcha is that deleting the StatefulSet does NOT delete its PVCs. They stay around. If the reclaimPolicy is Delete and you then manually delete the PVC — gone. If it's Retain, the PV is Released and you need to manually recover it by clearing the claimRef and rebinding.
>
> If the pod can't come back, check: does the PVC exist? If not — did the StorageClass have reclaimPolicy: Delete and someone deleted the PVC? Recreating the PVC won't get the data back. If the PVC does exist but the pod won't bind — check the PV status. If it's Released with Retain policy, clear the claimRef manually. If it's Bound to a different PVC, you have a conflict.
>
> The protection mechanism: the pvc-protection finalizer prevents deleting a PVC while a pod is using it. Force deletion bypasses this and is dangerous."

---

## Self-test drills

### 1. I deleted a PVC and now my StatefulSet pod can't come back. Walk me through what might have happened with reclaim policies.

**Reference answer**: full coverage in 60-second answer + StatefulSet section above. Key: `Delete` policy = data gone, pod can't recover. `Retain` policy = data safe, PV is Released, need to clear claimRef and rebind.

### 2. I want a StatefulSet to start fresh when I delete and recreate it. How do I do that?

**Reference answer**: You must explicitly delete the PVCs. StatefulSet deletion does NOT cascade to PVCs (default behavior). `kubectl delete pvc -l app=<label>` after deleting the StatefulSet. For ongoing behavior in K8s 1.27+, set `persistentVolumeClaimRetentionPolicy.whenDeleted: Delete` on the StatefulSet spec.

### 3. A PVC is stuck in Terminating. What's blocking it?

**Reference answer**: The `pvc-protection` finalizer is set. A pod is still using the PVC. Find the pod: `kubectl get pods -o json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<name>")'`. The PVC will delete once the pod terminates. If the pod is stuck: debug the pod termination first. Last resort: `kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'` removes the finalizer — but do this only if you know no pod is actually using it.

### 4. I have a Released PV from a previous workload. I want to reuse it. What's the procedure?

**Reference answer**: (1) Verify the backing volume still exists in the cloud. (2) `kubectl patch pv <name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'` to clear the old claim reference. (3) PV moves to Available. (4) Create a new PVC with `storageClassName: ""` and `volumeName: <pv-name>` to bind specifically to this PV. If the original data is sensitive, wipe it first (mount to a debug pod and run shred/dd before rebinding to the real workload).

---

## Further reading

- [K8s docs — PersistentVolumes lifecycle](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#lifecycle-of-a-volume-and-claim)
- [K8s docs — StatefulSet PVC retention policy](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#persistentvolumeclaim-retention)
- [K8s docs — Storage Object in Use Protection](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection)
- See also: `csi-architecture.md` (how volumes are provisioned), `storageclass.md` (reclaimPolicy at the class level), `volume-snapshots.md` (backup/restore)

---

## The 4 dimensions (senior framing)

- **Tech**: 4 PV phases (Available → Bound → Released → Failed); `Delete` removes backing volume; `Retain` preserves it; StatefulSet PVCs survive StatefulSet deletion by default (K8s 1.27+ adds `persistentVolumeClaimRetentionPolicy`); `pvc-protection` finalizer blocks PVC deletion while mounted; orphaned Released PVs need manual claimRef clearing.
- **People**: developers routinely delete StatefulSets expecting a clean slate and are surprised the old data is still there. Write this into your onboarding docs. Also write: "if you want to delete a StatefulSet and its data, you must also delete the PVCs." Conversely, make it clear that PVCs are not backed up by default — Retain just means the disk survives; it doesn't mean the data is safe from disk failure.
- **CI/CD**: Kyverno policy: deny PVC deletion in production namespaces unless the request comes from a service account in the `sre` group. Alert on `kubectl delete pvc` events in prod (Kubernetes audit log → alerting). In Helm charts for databases: always set `reclaimPolicy: Retain` explicitly rather than relying on StorageClass default. Add a pre-delete hook that creates a VolumeSnapshot before allowing a StatefulSet to be deleted.
- **Operations**: monitor for PVs in `Released` state > 7 days (orphaned volumes that cost money). Monitor for PVs in `Failed` state (manual intervention needed). Runbook for "StatefulSet pod won't start": check PVC exists → check PVC Bound → check PV phase → check VolumeAttachment → check CSI driver logs. The most common cause of "pod stuck in Pending after StatefulSet delete+recreate" is a Released PV whose claimRef still points to the old PVC.
