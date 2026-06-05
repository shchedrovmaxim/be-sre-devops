# CSI Architecture — the simple version (the hotel room key analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./csi-architecture.md) becomes easy.

This doc only explains **one idea**:

> **CSI is how Kubernetes talks to any storage system without knowing the details. It's a standard plug — anything that implements the plug works.**

---

## The hotel room key analogy

You check into a hotel. The hotel doesn't know what lock manufacturer made your door. It doesn't care. The hotel uses a standard key card system (the CSI interface). Whatever lock vendor you buy from, as long as they implement the card reader spec, the front desk can cut you a key.

| In the hotel world | In the K8s world |
|---|---|
| The key card standard | The CSI gRPC interface |
| Front desk (cuts the key) | Kubernetes control plane (requests the volume) |
| Your door (validates the key) | The storage node (mounts the volume) |
| Lock vendor (implements the reader) | CSI driver (EBS, NFS, Ceph, etc.) |
| The bellhop who carries your bag to the room | kubelet (the node-level agent that does the final mount) |

Before CSI: Kubernetes had to know how to talk to AWS EBS, GCE PD, Ceph, NFS, etc. — all baked into core Kubernetes code. Adding a new storage provider meant patching the K8s source. CSI moved that code out.

---

## The two halves of a CSI driver

Every CSI driver has two parts running in different places:

```
Control plane (runs as a Deployment, usually in kube-system):
  csi-controller  ← talks to the storage API (creates/deletes volumes in AWS/GCP/Ceph)
  + sidecar containers that watch K8s and trigger controller calls

Node (runs as a DaemonSet, one pod per node):
  csi-node  ← does the actual mount/unmount on the node's filesystem
  kubelet calls it when attaching a volume to a pod
```

Why split? Because "create a volume in AWS" (controller) is something that can happen anywhere. But "mount the volume on this specific node's filesystem" (node) has to happen on the right node.

---

## What happens when a pod starts with a PVC

Here's the timeline in plain English:

1. **You create a PVC** ("I need a 20Gi volume").
2. **The provisioner sees it** and calls the storage API ("create a 20Gi EBS volume in us-east-1a"). The volume now exists in AWS.
3. **Your pod gets scheduled** to a node.
4. **The attacher tells AWS** "attach that EBS volume to this EC2 instance." The disk now shows up as `/dev/xvdbf` on the node.
5. **The stager formats the disk** (if needed) and mounts it to a staging path.
6. **kubelet does the final bind-mount** into the pod's container filesystem at `/data` (or wherever your `mountPath` is).
7. **Your pod starts.** It sees a normal filesystem at `/data`.

On pod deletion, this unwinds in reverse.

---

## The common failure modes (and how to spot them)

| Symptom | What's stuck |
|---|---|
| Pod stays `Pending` forever | Volume wasn't provisioned (check CSI controller pod logs) |
| Pod stays `Pending`, PVC shows `Bound` but pod says `ContainerCreating` | Volume attach failed (check `kubectl describe node` for attacher errors) |
| Pod is `ContainerCreating` for a long time | Node-side mount is stuck (check kubelet logs on the node) |
| Pod terminated, new pod won't start, old node still "has" the volume | Stale volume attachment — the attacher didn't clean up. Force-delete the `VolumeAttachment` object. |

The trick: every step has its own K8s object. The PVC moves through states. The VolumeAttachment object is created for the attach step. Knowing which object to inspect tells you which step is stuck.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is CSI? | A standard gRPC interface that decouples Kubernetes from specific storage implementations. |
| Why does the driver have two parts? | Controller (cloud API calls, runs anywhere) and Node (filesystem mounts, must run on the right node). |
| What does kubelet do in storage? | After the CSI node plugin stages the volume, kubelet does the final bind-mount into the pod's filesystem. |
| What's a VolumeAttachment? | A K8s object that records "volume X is attached to node Y." Created during the attach step; deleted on detach. |
| What does "attach stuck" mean? | The CSI controller told the cloud to attach the volume to the node, but the cloud hasn't confirmed yet. Check cloud console for in-progress attach. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what happens when a Pod with a PVC starts, from the storage layer's perspective."**

**Reference answer (intuitive version):**

"There are four sequential steps. First, provisioning: if the PVC is new, the CSI controller calls the storage API to create the underlying volume (an EBS volume in AWS, for example). Second, attaching: once the pod is scheduled to a node, the external-attacher sidecar creates a VolumeAttachment object and calls the CSI controller to attach the volume to that specific node — in AWS terms, attaching an EBS volume to an EC2 instance. Third, staging: the CSI node plugin on that node formats the filesystem if needed and mounts the volume to a node-level staging path. Fourth, publishing: kubelet calls the CSI node plugin again to bind-mount from the staging path into the pod's container filesystem at the configured mountPath.

If any step fails, the pod stays in ContainerCreating. The trick to debugging is knowing which step is stuck: check the PVC for provisioning errors, check VolumeAttachment for attach errors, and check kubelet logs for mount errors."

---

## Further reading

- [K8s docs — CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [CSI spec on GitHub](https://github.com/container-storage-interface/spec)
- [Deep-dive: csi-architecture.md](./csi-architecture.md) — covers sidecar containers in detail, the gRPC calls at each step, stale attachment debugging, and kubelet's role in node-side operations
