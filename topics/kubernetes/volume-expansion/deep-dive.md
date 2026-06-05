# Volume Expansion — the deep-dive

> **Goal**: by the end you can answer the killer question — **"I expanded a PVC and it didn't grow — walk me through what might be blocking it."** — covering prerequisites, the two-phase resize (controller + node), how to detect partial-resize state, online vs offline filesystem resize, and common driver gotchas.

> Start with the [simple version](./simple.md) if you haven't read it. The office space analogy is the spine.

---

## The senior framing — volume expansion is simple in theory and annoying in practice

The theory: `kubectl patch pvc → cloud disk grows → filesystem grows → done.` Two-liner.

The practice: there are at least 4 distinct failure modes across two phases, the error messages are misleading, and "why isn't this working" debugging requires knowing which phase is stuck and which component is responsible for each phase.

Senior SREs get called on this regularly because it happens on live production databases and the stakes are high: the pod is out of disk space, writes are failing, and the window to fix it without downtime is short.

---

## Prerequisites — the checklist before you start

Before issuing a resize:

```bash
# 1. Check if the StorageClass allows expansion
kubectl get storageclass <name> -o jsonpath='{.allowVolumeExpansion}'
# Must be "true"

# 2. Check if the CSI driver supports resize
kubectl get csidriver <driver-name> -o jsonpath='{.spec.storageCapacity}'
# More reliable: check the driver's documentation for ControllerExpandVolume support

# 3. Verify the PVC's current state
kubectl get pvc <name> -o wide
# Should be Bound. Expansion on Pending or Released PVCs won't work.

# 4. Check for existing resize conditions
kubectl describe pvc <name>
# Look for Conditions section
```

**Required**: `allowVolumeExpansion: true` on the StorageClass. This is immutable on the StorageClass — you can't add it to an existing class without recreating it (though you can add it to a StorageClass that doesn't have it and existing PVCs will inherit the capability for future resize operations — the allowVolumeExpansion field can be changed on an existing StorageClass).

---

## Phase 1: ControllerExpandVolume

**Who triggers it**: the `external-resizer` sidecar container in the csi-controller Deployment. It watches PVCs for a `status.capacity` that is less than `spec.resources.requests.storage`.

**Sequence**:

```
1. User patches PVC: spec.resources.requests.storage: 20Gi → 50Gi

2. API server validates:
   - PVC is Bound
   - StorageClass has allowVolumeExpansion: true
   - New size > current size (no shrinking)
   - Accepts the change

3. external-resizer detects the spec/status mismatch.
   Adds annotation: volume.kubernetes.io/storage-resizer: <driver-name>

4. external-resizer calls ControllerExpandVolume gRPC on the CSI controller.
   Parameters: volumeId, capacity_range.required_bytes: 50Gi

5. CSI driver calls the storage API (AWS ec2:ModifyVolume, GCP disk resize, etc.)

6. Storage API returns success (the cloud disk is now 50 GiB).

7. external-resizer updates PV.spec.capacity to 50Gi.
   external-resizer updates PVC.status.capacity to 50Gi (pending node resize).
   Adds condition to PVC: FileSystemResizePending: True
```

**AWS-specific timing**: `ec2:ModifyVolume` is asynchronous. The API returns immediately but the modification takes 1-20 minutes. During this time, the `ControllerExpandVolume` call blocks (or polls). If the CSI driver doesn't handle this correctly, it may time out and retry, causing duplicate resize attempts (AWS deduplicates these, but it's noisy in logs).

**Phase 1 failure modes**:

- **API rejection at step 2**: `allowVolumeExpansion: false`, or new size <= current size. Error is immediate from the API server.
- **Cloud API error at step 5**: quota exceeded, wrong volume type (some older EBS volume types don't support online resize), IAM permission missing on the CSI driver's service account. Check external-resizer container logs.
- **Driver timeout**: cloud resize takes too long. The external-resizer retries. Usually self-healing. But if it keeps retrying and the condition never clears, check the cloud console for the volume status.

---

## Phase 2: NodeExpandVolume (filesystem resize)

**Who triggers it**: kubelet. When kubelet mounts a volume whose block device is larger than the filesystem on it (detected by comparing block device size to filesystem size), it calls NodeExpandVolume on the CSI node plugin.

**Sequence**:

```
1. FileSystemResizePending condition is True on the PVC.
   This signals that the cloud disk is now larger, but the filesystem hasn't grown.

2. Two paths depending on whether the volume is currently mounted:

   ONLINE RESIZE (pod is running, volume is mounted):
   kubelet detects the disk grew (via inotify or polling).
   kubelet calls NodeExpandVolume on the CSI node plugin.
   Node plugin runs:
     - For ext4: resize2fs /dev/xvdbf
     - For xfs: xfs_growfs /mount/path
   The filesystem expands in place. Pod immediately sees new capacity.
   FileSystemResizePending condition removed from PVC.

   OFFLINE RESIZE (older drivers, or volume not currently mounted):
   The node-side resize happens when the pod next starts and kubelet calls
   NodeStageVolume/NodePublishVolume. Before mounting to the pod, the CSI node
   plugin runs the filesystem resize tool.
   Pod must restart for the new capacity to be visible.
```

**Which CSI drivers support online resize?**

Most modern drivers (EBS CSI >= 1.6, GCP PD CSI >= 1.1, Azure Disk CSI >= 1.8) support online filesystem resize without pod restart. Older versions required the pod to restart.

**How to tell if online resize is supported**: check the CSI driver's changelog for "support online volume expansion" or "NodeExpandVolume for mounted volumes."

---

## Detecting the partial-resize state

The most common confused state: phase 1 is done, phase 2 is stuck.

```bash
# Check PVC conditions
kubectl describe pvc my-data
# Output includes:
# Conditions:
#   Type:                      FileSystemResizePending
#   Status:                    True
#   LastProbeTime:             <nil>
#   LastTransitionTime:        2024-01-15T10:30:00Z
#   Message: Waiting for user to (re-)start a pod to finish file system resize

# Check PVC capacity fields
kubectl get pvc my-data -o json | jq '{
  requested: .spec.resources.requests.storage,
  status_capacity: .status.capacity.storage
}'
# { "requested": "50Gi", "status_capacity": "50Gi" }
# Both show 50Gi - the PV and cloud disk are resized.
# But FileSystemResizePending means the filesystem inside is still old size.

# From inside the pod, verify:
df -h /data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvdbf       20G   18G  1.9G  91% /data
# ← Still shows 20G even though the disk is 50G
```

**Fix**: for drivers supporting online resize, the filesystem should have already grown. If it hasn't, restart the pod. kubelet will call NodeExpandVolume during the next mount sequence and the filesystem will grow.

For drivers that don't support online resize, the pod must be restarted. Once the pod starts fresh, kubelet calls NodeStageVolume → the CSI node plugin sees the disk size mismatch → runs resize2fs before mounting → pod gets the full 50 GiB.

---

## Common driver-specific gotchas

### EBS gp2 volumes and the optimization state

AWS `gp2` volumes can only be modified once every 6 hours. If you resize and then try again within 6 hours, the `ec2:ModifyVolume` API call returns an error. The external-resizer logs something like `VolumeModificationState is "optimizing"`. Wait 6 hours or use `gp3` volumes (no such limit).

### XFS and resize2fs

XFS uses `xfs_growfs` (not `resize2fs`). The CSI node plugin for most drivers knows which tool to use based on the filesystem type detected on the volume. If the volume was formatted with the wrong filesystem type somehow, the resize will fail with a confusing error message. Check `lsblk -f` on the node.

### Mount options that affect resize

Some mount options prevent online resize (`noexec`, unusual journal settings). This is rare but documented in driver-specific issues. If resize is stuck on a node with unusual mount options, check the CSI node plugin logs for `resize2fs` or `xfs_growfs` failure messages.

### ReadWriteMany (NFS/CephFS) volumes

Resize semantics differ. For NFS-backed volumes, "expanding the PVC" may expand a quota at the server level, not a block device. The `resize2fs` step doesn't apply. The node-side step for NFS-type CSI drivers is a quota update call, not a filesystem resize. The FileSystemResizePending condition behavior varies by driver.

---

## The "shrinking is not supported" rule

The API server enforces this at admission:

```
kubectl patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
# Error: The PersistentVolumeClaim "my-data" is invalid:
#   spec.resources.requests.storage: Forbidden: field can only be updated to a larger size
```

This is not a CSI limitation — it's a K8s API validation rule. Even if the underlying storage system could theoretically shrink a volume (some can), K8s won't allow it through the PVC API.

**To downsize**: provision a new PVC at the smaller size, copy data (e.g., with a copy pod using both PVCs as volumes), update the application to use the new PVC, delete the old PVC.

---

## The interview answer in 60 seconds

> "Volume expansion has two phases. First, the external-resizer calls ControllerExpandVolume on the CSI controller — this grows the actual cloud disk (EBS ModifyVolume, etc.). Second, kubelet calls NodeExpandVolume on the CSI node plugin — this runs resize2fs or xfs_growfs to expand the filesystem inside the disk.
>
> If the PVC patch was rejected immediately: the StorageClass doesn't have allowVolumeExpansion: true. That's the most common cause. You can't fix this for the existing StorageClass without recreating it.
>
> If the patch was accepted but nothing happened: check the external-resizer container logs in the csi-controller pod. There may be a cloud API error (EBS gp2 has a 6-hour cool-down between modifications).
>
> If the PVC shows the new size in status but df -h still shows the old size: phase 1 is done, phase 2 is pending. `kubectl describe pvc` will show FileSystemResizePending: True. For modern CSI drivers, this resolves automatically (online resize). For older drivers, restart the pod — kubelet runs the filesystem resize during the next NodeStage call.
>
> One gotcha: you can never shrink. The API rejects it at the admission layer."

---

## Self-test drills

### 1. I expanded a PVC and it didn't grow — walk me through what might be blocking it.

**Reference answer**: full coverage in the 60-second answer. Three root causes: StorageClass doesn't allow expansion (immediate API rejection); phase 1 stuck (cloud API error, check resizer logs); phase 2 stuck (FileSystemResizePending, restart pod or wait for online resize).

### 2. My pod is still using the old capacity after a PVC resize. `kubectl get pvc` shows the new size. Why?

**Reference answer**: Phase 1 (disk grew) completed. Phase 2 (filesystem resize) is pending or completed but the pod process has a stale view. Run `df -h /mount-path` from inside the pod. If it shows the old size, FileSystemResizePending is still set — restart the pod to trigger NodeExpandVolume. If df shows the new size but the application still says "out of space," the app has a separate quota or is using a filesystem cache.

### 3. I'm getting "VolumeModificationState is optimizing" in my CSI driver logs. What does this mean?

**Reference answer**: This is an AWS EBS gp2 limitation. EBS gp2 volumes can only be modified once every 6 hours. The previous modification hasn't completed "optimizing" yet. Wait for it to finish (check AWS console for the volume state). Switch to gp3 volumes — they have no such cooldown restriction.

### 4. A developer wants to downsize a PVC from 100Gi to 20Gi because they over-provisioned. What do you tell them?

**Reference answer**: Not possible through the PVC API — the API server enforces that storage can only increase, never decrease. The workaround: provision a new PVC at 20Gi, copy the data with a migration job that mounts both PVCs, update the application's PVC reference (update the Deployment/StatefulSet), verify everything is healthy on the new PVC, then delete the old one. This is an operational procedure, not a K8s feature.

---

## Further reading

- [K8s docs — Expanding PVCs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
- [K8s docs — CSI volume expansion](https://kubernetes.io/docs/concepts/storage/volumes/#csi-raw-block-volume-support)
- [AWS EBS CSI driver resize docs](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/resize.md)
- See also: `csi-architecture.md` (the NodeExpandVolume gRPC call context), `storageclass.md` (allowVolumeExpansion field), `pvc-pv-lifecycle.md` (PV state during resize)

---

## The 4 dimensions (senior framing)

- **Tech**: two phases (ControllerExpandVolume → NodeExpandVolume); `allowVolumeExpansion: true` prerequisite; `FileSystemResizePending` condition = phase 1 done; online resize (modern drivers) vs offline resize (pod restart required); gp2 6-hour cooldown trap; shrinking is impossible via the K8s API.
- **People**: disk-full emergencies happen at 3am. Write a runbook: "PVC is full, pod is failing writes, how to expand." The runbook should cover: check StorageClass for allowVolumeExpansion, patch the PVC, verify FileSystemResizePending clears, confirm df -h shows new size. Make this runbook accessible to all on-call engineers, not just storage experts. Set up capacity alerts at 70% and 85% to avoid 3am emergencies.
- **CI/CD**: set `allowVolumeExpansion: true` by default in all StorageClass definitions — it's strictly additive capability. Monitor PVC usage as part of your release process for stateful workloads (add a pre-deploy check that PVC usage is below 70%). Use Prometheus alertmanager to notify the team at 80% fill so there's time to resize before it's an emergency.
- **Operations**: alert on `FileSystemResizePending: True` for more than 15 minutes — it should clear automatically for online resize drivers; if it doesn't, a restart is needed. Alert on PVC capacity above 85%. Runbook for "disk full": emergency procedure is to expand the PVC immediately, not to restart the pod (restarting an out-of-space database before expanding the disk may cause data corruption if writes are actively failing). Expand first, verify the filesystem grew, then optionally restart.
