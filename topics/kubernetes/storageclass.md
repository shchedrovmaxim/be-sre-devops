# StorageClass + Dynamic Provisioning — the deep-dive

> **Goal**: by the end you can answer the killer question — **"What's the difference between WaitForFirstConsumer and Immediate volume binding, and when does it matter?"** — explaining the zone-affinity failure mode with Immediate, how WaitForFirstConsumer uses scheduler topology information, the reclaimPolicy trade-offs, and the two-default-class trap.

> Start with the [simple version](./storageclass-simple.md) if you haven't read it. The hotel room booking analogy is the spine.

---

## The senior framing — StorageClass is a provisioning policy, not just a template

Mid-level engineers think of StorageClass as "which type of disk to use." Senior engineers know StorageClass controls four distinct behaviors:

1. **Where the volume is created** (zone topology, via volumeBindingMode).
2. **What happens when the volume is no longer needed** (reclaimPolicy).
3. **Whether the volume can grow** (allowVolumeExpansion).
4. **How the volume behaves at mount time** (mountOptions).

Getting any one of these wrong at StorageClass definition time is an operational problem you'll fight repeatedly — because every PVC backed by that class inherits the mistake.

---

## The StorageClass manifest in full

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
  annotations:
    # Only one StorageClass in the cluster should have this annotation
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/mrk-xxxx"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
mountOptions:
- discard       # enables TRIM for SSDs — small performance benefit
```

### Key fields explained

**`provisioner`**: which CSI driver creates the volumes. Must match the driver's name exactly (`ebs.csi.aws.com`, `disk.csi.azure.com`, `pd.csi.storage.gke.io`, `rbd.csi.ceph.com`).

**`parameters`**: driver-specific key-value pairs passed to the CSI `CreateVolume` call. Completely opaque to Kubernetes — the driver interprets them.

**`reclaimPolicy`**: what happens to the PV (and the underlying volume) when the PVC is deleted. Values: `Delete` (PV and backing volume are deleted), `Retain` (PV and backing volume survive, status goes to `Released`). `Recycle` is deprecated and removed.

**`volumeBindingMode`**: `Immediate` or `WaitForFirstConsumer`. Controls when provisioning happens.

**`allowVolumeExpansion`**: if `true`, allows PVCs to request more storage than originally provisioned. The resize is handled by the external-resizer sidecar + CSI driver. Can only grow, never shrink.

**`mountOptions`**: extra options passed to the `mount` command when the volume is mounted on a node. Use for driver-specific tuning (`discard` for SSD TRIM, `nfsvers=4.1` for NFS4.1).

---

## volumeBindingMode: the zone topology problem

### Why Immediate fails for zonal storage

AWS EBS, Azure Disk, and GCE Persistent Disk are **zonal** — a volume created in `us-east-1a` can only be attached to EC2 instances in `us-east-1a`.

With `volumeBindingMode: Immediate`:

```
Time 1: PVC created. external-provisioner immediately calls CreateVolume.
         The provisioner doesn't know where the pod will run.
         It picks us-east-1a (or lets AWS decide).
         EBS volume created in us-east-1a. PV created. PVC Bound.

Time 2: Pod is scheduled. Scheduler finds nodes in us-east-1b and us-east-1c
         that have capacity. Schedules to a node in us-east-1c.

Time 3: Attacher tries to attach the EBS volume (us-east-1a) to node (us-east-1c).
         Fails. EC2 API error: "volume and instance must be in the same AZ."
         Pod stays in ContainerCreating indefinitely.
```

This is a subtle, hard-to-debug failure. The PVC shows `Bound`. The volume exists. Everything looks fine until you check the VolumeAttachment error.

### How WaitForFirstConsumer fixes it

```
Time 1: PVC created. volumeBindingMode: WaitForFirstConsumer.
         external-provisioner sees the PVC but does NOT call CreateVolume.
         PVC stays in Pending.

Time 2: Pod is scheduled. Scheduler:
         - Knows the pod has an unbound PVC.
         - Evaluates which nodes satisfy the pod's other constraints.
         - Picks a node in us-east-1b.
         - Sets an annotation on the PVC: "selectedNode: node-xyz"

Time 3: external-provisioner sees the annotation. Calls CreateVolume with
         topology requirement: "AccessibilityRequirements: us-east-1b"
         EBS volume created in us-east-1b. PV created. PVC Bound.

Time 4: Volume attaches to node in us-east-1b. Pod starts.
```

The scheduler's topology awareness is the key mechanism. By waiting for scheduling, the provisioner has the zone information it needs.

### When to use Immediate

`Immediate` is correct for storage that is **not zone-affine**:
- NFS servers (network-attached, accessible from any zone)
- AWS EFS (multi-AZ NFS)
- Ceph RBD (replicated across failure domains)
- ReadWriteMany volumes in general

For these, you want the volume created immediately (the `Pending` PVC with `WaitForFirstConsumer` is a confusing UX if the pod never gets scheduled).

---

## reclaimPolicy: the lifecycle trade-off

### Delete (safe for disposable data, dangerous for stateful apps)

When the PVC is deleted, Kubernetes deletes the PV, and the external-provisioner deletes the backing volume (the EBS volume, the disk). Data is gone.

**Use for**: logs, caches, temporary workloads, anything that can be regenerated.

**Danger**: in a StatefulSet, if you accidentally delete the PVC (not just the pod), the backing volume is gone. StatefulSet does not recreate PVCs by default.

### Retain (safe for stateful apps, requires manual cleanup)

When the PVC is deleted, the PV status moves to `Released`. The backing volume survives. But the PV is no longer bound to any PVC — you can't just re-create the PVC and get it back.

To reuse a Retained volume:
1. Remove the `claimRef` from the PV manually: `kubectl patch pv <name> -p '{"spec":{"claimRef": null}}'`
2. The PV moves to `Available`.
3. Create a new PVC that explicitly references this PV by name, or let the binder match it.

**Use for**: databases, any data that's expensive to recreate or has compliance retention requirements.

**Danger**: orphaned Retained PVs accumulate and cost money. You need a process to audit and clean them up.

---

## The default StorageClass annotation

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

A PVC without a `storageClassName` field uses the cluster's default StorageClass. This is convenient — it means simple workloads don't have to know which StorageClass to use.

**The trap**: if two StorageClasses both have this annotation, PVCs without an explicit class enter `Pending` with the error:

```
error: multiple default StorageClasses found
```

This commonly happens when:
- You install a cloud provider StorageClass (e.g., EKS auto-creates `gp2`).
- You also install the EBS CSI driver, which creates `gp3`.
- Both are annotated as default.

**Fix**: `kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class-`  (the trailing `-` removes the annotation).

**Check for it**: `kubectl get storageclass | grep default` — there should be exactly one line.

---

## Topology-aware provisioning details

When `WaitForFirstConsumer` is active, the scheduler annotates the PVC with:

```yaml
metadata:
  annotations:
    volume.kubernetes.io/selected-node: "ip-10-0-1-5.us-east-1b.compute.internal"
```

The external-provisioner watches for this annotation, reads the node's labels, extracts the zone (`topology.kubernetes.io/zone: us-east-1b`), and passes it to `CreateVolume` as a topology requirement:

```
CreateVolume(
  ...
  accessibility_requirements: {
    requisite: [{segments: {"topology.kubernetes.io/zone": "us-east-1b"}}]
  }
)
```

The CSI driver uses this to pick the right AZ for the underlying storage API call. If the driver or the storage system can't satisfy the topology constraint, the provisioning fails.

---

## The interview answer in 60 seconds

> "With Immediate binding, the volume is provisioned when the PVC is created. At that point the scheduler hasn't run yet, so the provisioner doesn't know which zone the pod will land in. It picks a zone — and if the pod gets scheduled to a different zone, the attach fails. For zonal storage like EBS, this causes pods to stay stuck in ContainerCreating with a misleading error, since the PVC shows Bound.
>
> WaitForFirstConsumer delays provisioning until the scheduler has committed the pod to a node. The scheduler annotates the PVC with the selected node. The provisioner reads that annotation, extracts the zone from the node's topology labels, and creates the volume in the right zone. The attach always succeeds.
>
> This matters for every zone-affine storage system: EBS, Azure Disk, GCE PD. It doesn't matter for multi-AZ storage like EFS or Ceph — for those, Immediate is fine and avoids the confusing Pending PVC state.
>
> The other common gotcha: two StorageClasses both marked as default. PVCs without an explicit class enter Pending immediately. Fix: remove the default annotation from one of them."

---

## Self-test drills

### 1. What's the difference between WaitForFirstConsumer and Immediate, and when does it matter?

**Reference answer**: Immediate provisions at PVC-creation time, before scheduling — can create volumes in the wrong zone. WaitForFirstConsumer waits for the scheduler's node selection and provisions in the matching zone. Matters for zonal storage (EBS, Azure Disk). Full answer above.

### 2. I deleted a PVC and now my data is gone. What could have prevented it?

**Reference answer**: The StorageClass had `reclaimPolicy: Delete`. When the PVC was deleted, the PV was deleted, and the backing volume was destroyed. Prevention: use `reclaimPolicy: Retain` for stateful data. Also: for StatefulSets, understand that PVCs are not deleted with the StatefulSet — only a manual `kubectl delete pvc` triggers this. Add RBAC restrictions on who can delete PVCs in production namespaces.

### 3. My cluster has two StorageClasses. A new PVC with no storageClassName is stuck in Pending. Why?

**Reference answer**: Both StorageClasses have the `storageclass.kubernetes.io/is-default-class: "true"` annotation. K8s doesn't know which one to use for unspecified PVCs, so the PVC stays Pending. Fix: remove the annotation from one of them (`kubectl annotate storageclass <name> storageclass.kubernetes.io/is-default-class-`).

### 4. Can I shrink a PVC?

**Reference answer**: No. Volume expansion is one-directional. Even if `allowVolumeExpansion: true` is set, you can only increase the size. To downsize, you must create a new PVC with the smaller size, copy data from the old PVC to the new one, update the application to use the new PVC, and delete the old one. This is an operational procedure, not a K8s feature.

---

## Further reading

- [K8s docs — StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [K8s docs — Volume Binding Mode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
- [AWS EBS CSI driver parameters](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/parameters.md)
- See also: `csi-architecture.md` (how StorageClass parameters get to the driver), `pvc-pv-lifecycle.md` (the full PVC/PV state machine), `volume-expansion.md` (the resize flow)

---

## The 4 dimensions (senior framing)

- **Tech**: `WaitForFirstConsumer` for zonal storage (EBS, Azure Disk, GCE PD); `Immediate` for multi-AZ storage (EFS, NFS, Ceph); `reclaimPolicy: Retain` for prod data; one and only one default class; `allowVolumeExpansion: true` standard practice; topology annotations flow from scheduler → PVC → external-provisioner → CSI CreateVolume.
- **People**: developers tend to omit `storageClassName` (relying on default) or use whatever the docs example shows. Codify your org's StorageClass choices in Helm chart defaults or a Kyverno mutation policy that injects the correct class based on workload labels. Make it easy to do the right thing by default.
- **CI/CD**: validate StorageClass names in Helm chart CI — fail if a chart references a StorageClass that doesn't exist in the target cluster (different clusters may have different classes). Kyverno policy: warn (not deny) if PVC uses `reclaimPolicy: Delete` and has a label indicating it's a database. Use OPA to block PVCs above a size threshold without an owner annotation.
- **Operations**: audit Retained PVs monthly — orphaned `Released` PVs cost money. Alert on PVCs in Pending >5 minutes. Track StorageClass distribution across the cluster to catch drift (someone adding a new default class during an upgrade is a common incident cause). For EKS upgrades: check if the new version installs a new StorageClass and whether it conflicts with existing defaults.
