# CSI Architecture — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through what happens when a Pod with a PVC starts, from the storage layer's perspective."** — naming all four volume lifecycle steps, the specific sidecar responsible for each, how kubelet interfaces with the node plugin, and the common failure modes at each step.

> Start with the [simple version](./csi-architecture-simple.md) if you haven't read it. The hotel key card analogy is the spine.

---

## The senior framing — CSI exists because in-tree plugins were unmaintainable

Before CSI (before K8s 1.9), every storage provider (AWS EBS, GCE PD, Azure Disk, Ceph, NFS) was baked into the Kubernetes codebase as an "in-tree" volume plugin. Adding a new storage vendor required a PR to the core K8s repo. Fixing a storage bug required a K8s release.

CSI (Container Storage Interface) moved storage vendor code out of K8s entirely. The standard is a gRPC interface specification. Any storage vendor that implements the three gRPC services (`Identity`, `Controller`, `Node`) can be deployed as a CSI driver without touching K8s source code.

Today: AWS EBS CSI driver, Ceph RBD CSI driver, NFS CSI driver, Longhorn, Rook, OpenEBS — all CSI. The in-tree plugins are all deprecated (most removed in 1.27+).

---

## The architecture: components and their roles

```
Kubernetes control plane
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Controller-side pods (usually Deployment in kube-system):      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  csi-controller pod (one replicated Deployment)          │   │
│  │                                                          │   │
│  │  [external-provisioner]  watches PVCs, calls             │   │
│  │      CreateVolume / DeleteVolume on the CSI controller   │   │
│  │                                                          │   │
│  │  [external-attacher]     watches VolumeAttachments,      │   │
│  │      calls ControllerPublishVolume / Unpublish           │   │
│  │                                                          │   │
│  │  [external-resizer]      watches PVCs with resize,       │   │
│  │      calls ControllerExpandVolume                        │   │
│  │                                                          │   │
│  │  [external-snapshotter]  watches VolumeSnapshots,        │   │
│  │      calls CreateSnapshot / DeleteSnapshot               │   │
│  │                                                          │   │
│  │  [driver container]      implements CSI Controller gRPC  │   │
│  │      service (talks to the actual storage API)           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Node (one DaemonSet pod per node):
┌─────────────────────────────────────────────────────────────────┐
│  csi-node pod                                                   │
│                                                                 │
│  [node-driver-registrar]   registers the driver with kubelet   │
│      via the kubelet plugin registration mechanism              │
│                                                                 │
│  [driver container]        implements CSI Node gRPC service     │
│      (NodeStageVolume, NodePublishVolume, NodeUnpublish...)     │
│                                                                 │
│  kubelet communicates with the node plugin via a Unix socket    │
│  at /var/lib/kubelet/plugins/<driver-name>/csi.sock            │
└─────────────────────────────────────────────────────────────────┘
```

The sidecars (external-provisioner, external-attacher, etc.) are maintained by the Kubernetes SIG-storage community as generic containers. They handle all the K8s-watching logic and translate K8s events into CSI gRPC calls. The driver author only has to implement the gRPC services.

---

## The 4-step volume lifecycle

### Step 1: CreateVolume (provisioning)

**Trigger**: a PVC in `Pending` state with no matching PV.

**Who does it**: `external-provisioner` watches PVCs → calls `CreateVolume` on the CSI Controller service → the driver calls the storage API (e.g., AWS `ec2:CreateVolume`) → returns volume ID.

**K8s objects created**: a PersistentVolume is created by the provisioner. PVC status moves to `Bound`.

```
PVC (Pending) → external-provisioner → CreateVolume RPC → storage API
                                                         ↓
                                            PV created, PVC Bound
```

**Failure mode**: PVC stays `Pending`. Check `kubectl describe pvc <name>` for events. Check csi-controller pod logs for the provisioner container.

### Step 2: ControllerPublishVolume (attaching)

**Trigger**: a pod consuming the PVC is scheduled to a node.

**Who does it**: `external-attacher` creates a `VolumeAttachment` object → calls `ControllerPublishVolume` on the CSI Controller → driver calls the storage API (e.g., AWS `ec2:AttachVolume`).

**K8s objects**: a `VolumeAttachment` object tracks the attachment state. Its `.status.attached` becomes `true` when the cloud confirms.

```yaml
# Example VolumeAttachment (auto-created, not written by hand)
apiVersion: storage.k8s.io/v1
kind: VolumeAttachment
metadata:
  name: csi-xxxx
spec:
  attacher: ebs.csi.aws.com
  nodeName: ip-10-0-1-5.ec2.internal
  source:
    persistentVolumeName: pvc-uuid
status:
  attached: true    # ← set when the cloud confirms the attach
```

**Failure mode**: pod stuck in `ContainerCreating`. `kubectl describe pod` shows `AttachVolume.Attach failed`. Check `kubectl get volumeattachment`. Check cloud console for in-progress or failed attach.

**The stale attachment gotcha**: if a node terminates ungracefully (kernel panic, power cut), the VolumeAttachment may still exist, claiming the volume is attached to the dead node. The attacher won't try to attach it to a new node while the old attachment exists. Fix: manually delete the VolumeAttachment object (`kubectl delete volumeattachment <name>`). The CSI controller will re-attach to the new node.

### Step 3: NodeStageVolume (staging)

**Trigger**: VolumeAttachment is complete. kubelet calls this.

**Who does it**: kubelet calls `NodeStageVolume` on the CSI node plugin (via the Unix socket). The node plugin:
1. Waits for the block device to appear on the node (e.g., `/dev/xvdbf`).
2. Formats the filesystem if empty (`mkfs.ext4 /dev/xvdbf`).
3. Mounts the device to a **global staging path**: `/var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount/`.

The staging path is shared — multiple pods on the same node using the same PV see the same staged mount. (RWX PVs use this; RWO PVs only ever have one pod, but the path still exists.)

**Failure mode**: kubelet logs show `NodeStage failed`. Common causes: wrong filesystem type, device not appearing (AWS EBS can take seconds to appear after attach API call), permission issues.

### Step 4: NodePublishVolume (publishing)

**Trigger**: staging is complete. kubelet calls this.

**Who does it**: kubelet calls `NodePublishVolume` on the CSI node plugin. The plugin bind-mounts from the global staging path to the pod-specific path:

```
/var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount/
    → bind-mounted to →
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pv-name>/mount/
    → which is exposed to the container as →
/data   (or whatever mountPath the container specifies)
```

After this, kubelet starts the container. The container sees a regular filesystem at `/data`.

---

## The teardown sequence (pod deletion)

The lifecycle unwinds in reverse order:

1. `NodeUnpublishVolume`: remove the pod-specific bind-mount.
2. `NodeUnstageVolume`: unmount the global staging path.
3. `ControllerUnpublishVolume`: detach the volume from the node (AWS `ec2:DetachVolume`).
4. (Optionally) `DeleteVolume`: delete the backing volume (if reclaimPolicy is Delete).

**Gotcha**: kubelet must complete NodeUnpublish/NodeUnstage before the external-attacher can detach. If the pod is stuck (frozen container, zombie process), the detach is blocked. The volume is in a "detach pending" state. The VolumeAttachment exists, `.status.detachError` may be set. Fix: resolve the pod termination first; then the volume lifecycle unblocks automatically.

---

## kubelet's role in detail

kubelet discovers registered CSI drivers via the kubelet plugin registration mechanism:

1. The `node-driver-registrar` sidecar calls kubelet's registration API at startup.
2. kubelet records the driver name and the Unix socket path for the driver.
3. When kubelet needs to call NodeStageVolume or NodePublishVolume, it connects to that socket.

This means the CSI node driver must be running and its socket must exist before kubelet tries to stage or publish a volume. If the DaemonSet pod restarts (e.g., OOMKilled), kubelet will temporarily lose the socket and volume operations on that node will fail until the pod recovers.

**After kubelet restarts**: kubelet reconstructs its state from the filesystem. It walks `/var/lib/kubelet/pods/` and reconciles existing bind-mounts. For each volume it finds, it re-calls the CSI driver to verify the state. This is why a kubelet restart doesn't unmount existing pod volumes — the reconciliation keeps them in place.

---

## The interview answer in 60 seconds

> "CSI replaced in-tree volume plugins with a standard gRPC interface. Every CSI driver has two parts: a controller (usually a Deployment) that talks to the storage API, and a node DaemonSet that does filesystem operations on each node.
>
> When a pod with a PVC starts, there are four sequential steps. First, provisioning: the external-provisioner calls CreateVolume on the CSI controller, which calls the storage API (e.g., ec2:CreateVolume). A PV is created and the PVC is Bound. Second, attaching: once the pod is scheduled, the external-attacher creates a VolumeAttachment and calls ControllerPublishVolume — in AWS, that's attaching the EBS volume to the EC2 instance. Third, staging: kubelet calls NodeStageVolume on the node plugin, which formats the disk and mounts it to a global staging path on the node. Fourth, publishing: kubelet calls NodePublishVolume, which bind-mounts the staging path into the pod's specific filesystem path. Then the container starts.
>
> Common failure modes: PVC stays Pending (provisioner error), pod stuck in ContainerCreating with an AttachVolume error (check VolumeAttachment), or kubelet restart leaving a stale socket. The stale-attachment gotcha is the worst: if a node dies, the VolumeAttachment still exists, blocking re-attach on a new node. Fix: delete the VolumeAttachment manually."

---

## Self-test drills

### 1. Walk me through what happens when a Pod with a PVC starts, from the storage layer's perspective.

**Reference answer**: four steps — CreateVolume (provisioner), ControllerPublishVolume (attacher + VolumeAttachment), NodeStageVolume (kubelet → node plugin formats + mounts to global path), NodePublishVolume (kubelet → node plugin bind-mounts to pod path). Full sequence above.

### 2. My pod has been in ContainerCreating for 10 minutes. The PVC shows Bound. What do I check?

**Reference answer**: Check `kubectl describe pod` for events — look for `AttachVolume.Attach failed`. Check `kubectl get volumeattachment` for the volume's attachment state. If the VolumeAttachment shows `.status.attached: false` with an error, the CSI controller couldn't attach (cloud API issue, cloud quota, wrong AZ). Check the csi-controller pod logs (external-attacher container). If VolumeAttachment is attached, the issue is node-side — check kubelet logs on the scheduled node for NodeStage/NodePublish errors.

### 3. A node died unexpectedly. My pod was on it. The replacement pod is stuck in ContainerCreating. What's happening?

**Reference answer**: The VolumeAttachment object still points the volume to the dead node. The external-attacher sees the volume as "attached" and won't re-attach it to the new node. You need to verify the old node is truly dead (or force-delete it), then delete the stale VolumeAttachment object. The external-attacher will then re-attach to the new node.

### 4. What happens to mounted volumes when kubelet restarts?

**Reference answer**: kubelet reconstructs state from `/var/lib/kubelet/pods/`. It finds existing bind-mounts, verifies them by calling the CSI node plugin, and keeps them in place if they're healthy. Pods don't lose their volume mounts on a kubelet restart — the mounts persist in the kernel. kubelet just re-registers them.

---

## Further reading

- [K8s docs — CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [CSI spec GitHub](https://github.com/container-storage-interface/spec/blob/master/spec.md)
- [Kubernetes-csi sidecars](https://kubernetes-csi.github.io/docs/sidecar-containers.html)
- [AWS EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- See also: `storageclass.md` (how StorageClass controls provisioning behavior), `pvc-pv-lifecycle.md` (the PVC/PV state machine)

---

## The 4 dimensions (senior framing)

- **Tech**: 4-step lifecycle (CreateVolume → ControllerPublish → NodeStage → NodePublish); controller Deployment + node DaemonSet split; sidecar containers (external-provisioner, external-attacher, external-resizer, external-snapshotter); VolumeAttachment as the attach state object; kubelet plugin registration via Unix socket; stale attachment fix = delete VolumeAttachment.
- **People**: storage failures at each step have different owners. Provisioning failures (wrong StorageClass, cloud quota) → platform team. Node-side mount failures → platform team + app team if it's a filesystem config issue. Help developers read `kubectl describe pod` and `kubectl describe pvc` — most CSI errors surface there first. Write a runbook for "pod stuck in ContainerCreating with a PVC."
- **CI/CD**: use a StorageClass with `volumeBindingMode: WaitForFirstConsumer` in staging environments that mirror prod topology. Validate PVC size requests in CI (Kyverno policy that blocks PVCs over X Gi without a label justification). In Helm charts, always specify a `storageClassName` explicitly — relying on the default class is a footgun if there are two default classes.
- **Operations**: monitor VolumeAttachment objects — a stuck attachment (exists but not `.status.attached: true`) is a leading indicator. Alert on PVCs in `Pending` state for >5 minutes. Alert on CSI controller pod restarts (every restart can cause a window of attachment failures). For node drain operations: drain waits for pods to terminate, which triggers NodeUnpublish → ControllerUnpublish in sequence. If a pod termination is stuck, the drain is stuck. Check for pods with `terminationGracePeriodSeconds` set very high.
