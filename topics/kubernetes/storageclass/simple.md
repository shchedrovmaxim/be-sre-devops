# StorageClass + Dynamic Provisioning — the simple version (the hotel room booking analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **A StorageClass is a template for creating volumes. It tells Kubernetes *how* to provision storage — not *what* to provision. The PVC says "I need 20Gi"; the StorageClass says "here's how to make that."**

---

## The hotel room booking analogy

You're booking a hotel room. You don't specify which exact room — you specify a **room type**: "Standard Room" or "Deluxe Suite."

The hotel has a booking system. When your reservation is confirmed (pod is scheduled), it assigns you a specific room that matches your type.

| In the hotel world | In the K8s world |
|---|---|
| Room type (Standard, Deluxe) | **StorageClass** |
| Booking a room | Creating a **PVC** (requesting storage) |
| The specific room number you get | The **PV** (the actual volume created) |
| "Assign room when guest checks in" | `WaitForFirstConsumer` binding mode |
| "Reserve room the moment you book" | `Immediate` binding mode |
| "Room is cleaned and re-used after checkout" | `reclaimPolicy: Delete` |
| "Room is sealed off; you must clean it yourself" | `reclaimPolicy: Retain` |

---

## What a StorageClass looks like

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # makes this the default
provisioner: ebs.csi.aws.com
parameters:
  type: gp3           # which EBS volume type to create
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete          # delete the EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer   # don't create the volume until a pod needs it
allowVolumeExpansion: true     # lets you resize PVCs later
```

A PVC that uses this class:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

When the PVC is created, K8s knows: "use the `fast-ssd` class to provision this." The actual EBS volume isn't created yet (because `WaitForFirstConsumer`) — it waits until a pod that claims this PVC is scheduled.

---

## The `WaitForFirstConsumer` vs `Immediate` trap

This is the most important gotcha in StorageClass.

### `Immediate` (old default, problematic)

The volume is created the moment the PVC is created, in whatever zone the provisioner picks. If your pod later gets scheduled to a different zone — the zone that doesn't have the volume — it can never start.

```
PVC created → EBS volume created in us-east-1a
Pod scheduled → lands on a node in us-east-1b  ← can't attach! EBS is zonal.
Pod stays in Pending forever.
```

### `WaitForFirstConsumer` (modern default, correct for zonal storage)

The volume is created only after a pod that claims the PVC is scheduled to a node. The volume is created in the **same zone** as that node.

```
PVC created → no volume yet, PVC stays in Pending
Pod scheduled to node in us-east-1b
→ provisioner creates EBS volume in us-east-1b → volume attaches to the node → pod starts
```

**Rule of thumb**: always use `WaitForFirstConsumer` for any zone-affine storage (EBS, Azure Disk, GCE PD). Use `Immediate` only for truly cross-zone storage (EFS, NFS, Ceph with replication).

---

## The two default StorageClass trap

Most clusters ship with a default StorageClass. If a PVC doesn't specify `storageClassName`, it uses the default.

The trap: some Kubernetes distributions install **two** default StorageClasses. Both have the annotation `storageclass.kubernetes.io/is-default-class: "true"`. PVCs without an explicit class become `Pending` with an error like "many default StorageClasses found."

Check for it:

```bash
kubectl get storageclass
# If you see two with (default) in the output — you have a problem
```

Fix: remove the annotation from one of them.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is a StorageClass? | A template that tells the CSI driver how to provision a volume (type, size limits, IOPS, reclaim policy). |
| What is dynamic provisioning? | Instead of pre-creating PVs manually, the provisioner creates them automatically when a PVC is made. |
| What does `WaitForFirstConsumer` do? | Delays volume creation until the pod is scheduled, so the volume is created in the right zone. |
| What is `reclaimPolicy: Delete`? | When the PVC is deleted, the underlying volume (EBS volume) is also deleted. |
| What is `reclaimPolicy: Retain`? | The underlying volume survives PVC deletion. Someone must clean it up manually. |
| What does `allowVolumeExpansion: true` do? | Allows PVCs backed by this class to be resized (capacity can be increased, never decreased). |

---

## Self-test (one question — the killer one)

Out loud:

> **"What's the difference between WaitForFirstConsumer and Immediate volume binding, and when does it matter?"**

**Reference answer (intuitive version):**

"With Immediate binding, the volume is provisioned as soon as the PVC is created, before any pod is scheduled. The provisioner doesn't know which zone the pod will land in, so it picks a zone — which may be the wrong one. The pod then gets scheduled to a different zone and can never attach the volume, because EBS volumes are zonal.

WaitForFirstConsumer delays provisioning until a pod claiming the PVC is scheduled to a node. Now the provisioner knows which zone the node is in, and creates the volume there. The pod can always attach it.

This matters for any zonal storage — EBS, Azure Disk, GCE PD. It doesn't matter for network-attached cross-zone storage like EFS or NFS, where the volume is accessible from any zone. For those, Immediate is fine and actually faster."

---

## Further reading

- [K8s docs — StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Deep-dive: storageclass.md](./deep-dive.md) — covers the full field reference, the default class annotation, topology constraints, reclaim policies in detail, and the `mountOptions` field
