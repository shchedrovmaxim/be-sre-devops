# PVC / PV Lifecycle + Reclaim Policies — the simple version (the apartment rental analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **A PVC is a rental application. A PV is the actual apartment. When you leave (delete the PVC), what happens to the apartment depends on your lease terms (reclaimPolicy).**

---

## The apartment rental analogy

You're renting an apartment through a property management company.

| In the rental world | In the K8s world |
|---|---|
| Your rental application ("I need a 1-bed apartment") | **PVC** — PersistentVolumeClaim |
| The specific apartment unit you get | **PV** — PersistentVolume |
| The property management company | The CSI provisioner + K8s controller |
| Move-in (you start using the apartment) | Pod is bound to the PVC |
| The apartment is available to rent | PV status: `Available` |
| You're living there | PV status: `Bound` |
| You moved out but haven't cleaned up | PV status: `Released` |
| Something went wrong with the apartment | PV status: `Failed` |
| Your lease says "unit is demolished after you leave" | `reclaimPolicy: Delete` |
| Your lease says "unit is kept but you must clean it yourself" | `reclaimPolicy: Retain` |

---

## The 4 PV phases

A PersistentVolume goes through 4 states in its life:

```
Available → Bound → Released → (Failed or back to Available)
```

| Phase | What it means |
|---|---|
| **Available** | Volume exists, no PVC has claimed it yet |
| **Bound** | A PVC has claimed it. The pod can use it. |
| **Released** | The PVC was deleted. The volume still exists, but K8s won't bind new PVCs to it yet (it might have old data). |
| **Failed** | Something went wrong during reclamation. Manual intervention needed. |

---

## reclaimPolicy: Delete vs Retain

### Delete (the default for dynamically provisioned volumes)

When the PVC is deleted:
- The PV is deleted.
- The backing volume (EBS volume, disk) is deleted.
- **Data is gone.**

Good for: caches, temp data, anything you can regenerate.
Danger: deleting a PVC for a database = data loss.

### Retain

When the PVC is deleted:
- The PV moves to `Released`.
- The backing volume survives.
- **Data is safe.**

But the PV is now in an awkward state — it's "Released" and K8s won't automatically bind a new PVC to it (the data from the old tenant is still there).

To reuse it, a human must:
1. Manually inspect/clean the data.
2. Remove the old claim reference from the PV.
3. Create a new PVC that references it.

Good for: databases, compliance data, anything you can't lose.

---

## The StatefulSet PVC gotcha

StatefulSets create PVCs automatically. But when you delete a StatefulSet, the PVCs are **NOT deleted**. They stay around.

This is intentional. StatefulSets assume you care about the data. If you scale a StatefulSet from 3 to 0 and back to 3, the same PVCs (and same data) come back.

But it means: if you want a truly fresh start (drop all data), you must delete the PVCs manually:

```bash
kubectl delete pvc -l app=my-statefulset
```

This surprises developers who delete and recreate a StatefulSet expecting a clean slate, and find the old data still there.

---

## What actually blocks PVC deletion

When you `kubectl delete pvc`, K8s doesn't delete it immediately if a pod is still using it. It sets a **finalizer** (`kubernetes.io/pvc-protection`) on the PVC. The finalizer prevents deletion until all pods using the PVC have terminated.

This protects you from accidentally deleting a volume that's still mounted. The PVC will show as `Terminating` until the last pod using it goes away.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's the difference between a PVC and a PV? | PVC = request (what you want). PV = the actual volume (what you got). |
| Who creates PVs? | Dynamically: the CSI provisioner (auto, in response to PVC). Statically: a cluster admin (manually created PVs for pre-existing volumes). |
| What is `Released` status? | The PVC was deleted, but the PV (and the data) still exists. Someone must clean it up before it can be reused. |
| Does deleting a StatefulSet delete its PVCs? | No. PVCs must be deleted manually. This is a feature, not a bug — StatefulSets assume you care about the data. |
| What blocks PVC deletion? | The `pvc-protection` finalizer prevents deletion while a pod is still mounting the PVC. |

---

## Self-test (one question — the killer one)

Out loud:

> **"I deleted a PVC and now my StatefulSet pod can't come back. Walk me through what might have happened with reclaim policies."**

**Reference answer (intuitive version):**

"The most likely cause depends on the StorageClass's reclaimPolicy. If it was Delete, then when I deleted the PVC, the PV and the backing volume (EBS disk) were also deleted. The StatefulSet needs that PVC — its pod spec references a specific PVC name. Since the PVC no longer exists, the pod can't be scheduled. The StatefulSet controller doesn't recreate PVCs — that's not in its job description.

If the reclaimPolicy was Retain, the backing volume still exists, but the PV is now in Released status. The StatefulSet is looking for a PVC with the name it expects, but that PVC doesn't exist anymore. I'd need to recreate the PVC manually with the right name, and bind it to the Released PV by removing the old claimRef from the PV.

The lesson: for StatefulSet data, always use Retain so the backing volume survives accidental PVC deletion, and protect your PVCs with RBAC so that deletion requires explicit intent."

---

## Further reading

- [K8s docs — PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Deep-dive: pvc-pv-lifecycle.md](./deep-dive.md) — covers the full state machine, the finalizer mechanism, orphaned PV recovery, StatefulSet PVC retention policy (v1.23+), and backup/restore patterns with VolumeSnapshots
