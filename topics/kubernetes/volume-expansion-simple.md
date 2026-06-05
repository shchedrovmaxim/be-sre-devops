# Volume Expansion — the simple version (the growing office space analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./volume-expansion.md) becomes easy.

This doc only explains **one idea**:

> **Expanding a PVC is a two-step operation: first the cloud disk grows, then the filesystem inside it grows. If one step is done but not the other, the pod doesn't see more space.**

---

## The office space analogy

Your company rents office space. You started with 50 desks; now you need 100. You call the building manager.

**Step 1**: the building manager physically removes the wall and adds the new square footage. The room is now bigger. But all the existing cubicles, desks, and file cabinets are still only set up in the original 50-desk area. The extra space is empty and unused.

**Step 2**: facilities comes in, sets up the additional cubicles, and extends the layout to use the full new space. Now your team has 100 desks.

If step 1 is done but not step 2: the room is bigger, but you can only use 50 desks. Your team reports "we still have no space."

Volume expansion in Kubernetes works the same way:

| In the office world | In the K8s world |
|---|---|
| Building manager extends the physical space | CSI controller grows the cloud disk (EBS volume size increases) |
| Facilities sets up the new area | CSI node plugin runs `resize2fs` to expand the filesystem |
| The team actually sits at the new desks | The pod can write to the new capacity |
| Building rules allow expansion ("expandable lease") | `allowVolumeExpansion: true` on the StorageClass |
| The existing cubicle layout limits your use | The filesystem hasn't been resized yet |

---

## What you actually do to expand a PVC

Just edit the PVC's storage request:

```bash
kubectl patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
# was: 20Gi → now requests 50Gi
```

Then Kubernetes takes over. The resize happens in two phases automatically (for most modern CSI drivers):

**Phase 1 (controller resize)**: The external-resizer sidecar calls the CSI controller → the driver calls the cloud API (`ec2:ModifyVolume`) → the underlying disk grows from 20 GiB to 50 GiB. This can take a few minutes.

**Phase 2 (node resize)**: kubelet detects the disk grew. It calls the CSI node plugin → the plugin runs `resize2fs` (or equivalent) on the mounted filesystem → the filesystem expands to use the new disk size. The pod immediately sees 50 GiB available.

---

## The common ways expansion gets stuck

### "The StorageClass doesn't allow expansion"

The most common reason. `allowVolumeExpansion: false` (or not set). The API server rejects your patch immediately with:

```
error: persistentvolumeclaims "my-data" could not be patched:
  spec.resources[storage]: Forbidden: field is immutable
```

Fix: you can't fix this for existing PVCs. The StorageClass must be recreated with `allowVolumeExpansion: true`. For existing PVCs, you'd need to migrate data to a new PVC on a new StorageClass.

### "The controller resize is done but the filesystem didn't grow"

The PVC shows the new size (`50Gi`), the underlying disk is 50 GiB, but `df -h` inside the pod still shows 20 GiB. Phase 1 completed; phase 2 is pending.

Check:

```bash
kubectl describe pvc my-data
# Look for Conditions:
# FileSystemResizePending: True  ← filesystem resize pending on node
```

Fix: the node-side resize happens when the pod restarts (for offline resize) or automatically while running (for online resize). Most modern filesystems (ext4, xfs) and CSI drivers support online resize. If it's stuck, restart the pod.

### "I tried to shrink the PVC"

You can't. The API will reject it. Volume shrinking is not supported by the Kubernetes API or any standard CSI driver. You edited the PVC to a smaller size: the API server rejects the change.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What do I need before expanding a PVC? | `allowVolumeExpansion: true` on the StorageClass, and a CSI driver that supports resize. |
| How do I trigger an expansion? | Edit the PVC's `spec.resources.requests.storage` to a larger value. |
| What are the two phases? | 1. Controller resizes the cloud disk. 2. Node resizes the filesystem. Both need to complete. |
| What does `FileSystemResizePending` mean? | Phase 1 is done but phase 2 hasn't happened yet. Restarting the pod usually triggers phase 2. |
| Can I shrink a PVC? | No. Never. |

---

## Self-test (one question — the killer one)

Out loud:

> **"I expanded a PVC and it didn't grow — walk me through what might be blocking it."**

**Reference answer (intuitive version):**

"I'd check two things. First: does the StorageClass have `allowVolumeExpansion: true`? If not, the API rejects the patch immediately — that's the most common cause.

If the patch was accepted, I'd check PVC status with `kubectl describe pvc`. There's a condition called `FileSystemResizePending`. If that's set to True, it means the controller resize succeeded (the cloud disk grew) but the node-side filesystem resize hasn't happened yet. That's phase 2. For most modern CSI drivers with ext4/xfs, the filesystem resize happens online while the pod is running, but kubelet needs to trigger it. Restarting the pod usually fixes it.

If there's no progress at all after the patch, check the CSI controller pod logs for the external-resizer container — it might be getting an error from the cloud API (quota, wrong volume type, etc.)."

---

## Further reading

- [K8s docs — Expanding PVCs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
- [Deep-dive: volume-expansion.md](./volume-expansion.md) — covers the controller and node resize gRPC calls in detail, online vs offline resize, the partial-resize state detection, and driver-specific gotchas
