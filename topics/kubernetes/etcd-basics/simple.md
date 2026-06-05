# etcd basics — the simple version (the single source of truth analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **etcd is the cluster's single source of truth. Everything K8s knows about your cluster — every pod, every node, every secret — lives there. If etcd is gone, the control plane is blind. If it's corrupt, the cluster is corrupt.**

That's why understanding etcd is what separates "I use K8s" from "I run K8s."

---

## The city hall registry

Think of etcd as the city's land registry:

| City hall world | K8s world |
|---|---|
| The master record of every property in the city | etcd — the master record of every K8s object |
| If the registry burns down, no one knows who owns what | If etcd is lost, the cluster is completely blind |
| Three separate offices, each with a copy | Three etcd members forming a cluster |
| A change only becomes official when 2 of the 3 offices agree | **Raft consensus** — a write needs quorum (2 of 3) to commit |
| If 1 office is closed, the other 2 can still do official business | etcd with 3 members can survive 1 failure |
| If 2 offices are closed, no one can issue new deeds (but the records aren't lost) | etcd with 3 members can't accept writes if 2 are down (but reads may still work) |

---

## Raft consensus — the key numbers

| Cluster size | Failures tolerated | Write quorum needed |
|---|---|---|
| 1 member | 0 | 1 |
| 3 members | 1 | 2 |
| 5 members | 2 | 3 |
| 7 members | 3 | 4 |

**The rule**: always use an **odd number** of members. An even number gives you no benefit over the odd number below it.

- 4 members tolerates 1 failure (same as 3) — but requires 3 for quorum (more expensive). Useless.
- 3 or 5 are the standard choices.

**For most clusters**: 3 etcd members, spread across 3 availability zones. You can lose one AZ and the cluster keeps working.

---

## The three things you need to know for on-call

### 1. Snapshots — how you back it up

```bash
# Take a snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key

# Verify a snapshot
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

### 2. The 8GB quota wall

etcd has a default storage limit of **8GB**. When it hits that limit, it stops accepting writes. Every `kubectl apply` fails. The cluster is effectively frozen.

What causes it:
- Large objects stored in etcd (multi-MB ConfigMaps, Helm release history, large CRDs)
- Events and audit records piling up

Fix: run `etcdctl defrag` to compact the database and free space. Also set a retention limit on events.

### 3. Disk speed matters enormously

etcd writes are synchronous. Every write goes to disk (fsync) before being acknowledged. If the disk is slow (spinning HDD, high-latency NFS), etcd leader elections and write latency go through the roof.

**The rule**: etcd must run on **fast local SSD** (NVMe preferred). Never put etcd data on network-attached storage.

---

## What managed K8s hides from you

On EKS, GKE, and AKS, the control plane (including etcd) is managed by the cloud provider:
- You never see etcd directly
- Backup/restore is handled by the provider
- etcd scaling is the provider's problem
- You can't `etcdctl` directly

This is fine for most use cases. But if you're running self-managed K8s (kubeadm, bare metal, or on-prem), etcd ops become your responsibility.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is etcd? | A distributed key-value store that holds all K8s state. |
| Why odd member counts? | Even members offer no additional fault tolerance over the odd number below (e.g., 4 tolerates 1 failure, same as 3). |
| What happens if etcd loses quorum? | Reads may still work (stale data). Writes are rejected. The cluster is frozen. |
| What's the 8GB quota? | etcd's default storage limit. Hit it → all writes fail. Run `etcdctl defrag` to compact. |
| Why does disk speed matter? | Every write fsync's to disk before ACK. Slow disk = slow cluster operations. |
| Do I need to worry about etcd on EKS/GKE? | No — the cloud provider manages it. Only a concern for self-managed clusters. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what you need to know about etcd to be on-call for a self-managed K8s cluster."**

**Reference answer (intuitive version):**

"etcd is the single source of truth for all cluster state. On-call concerns: first, backups — I should have automated snapshots running every few hours via `etcdctl snapshot save`, stored off-cluster. Second, the 8GB quota wall — if etcd fills up, all writes fail; I'd monitor disk usage and run `etcdctl defrag` proactively. Third, disk performance — etcd needs fast local SSD (NVMe); if we see elevated leader election timeouts or slow writes, slow disk is the usual culprit. Fourth, quorum — three members across three AZs; if two are down, the cluster can't take writes; the recovery path is restoring from snapshot. On EKS/GKE/AKS, the provider manages all of this — it only becomes my problem on self-managed clusters."

---

## Further reading

- [etcd docs](https://etcd.io/docs/)
- [etcd disaster recovery](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

---

## Next: the deep-dive

When the city hall analogy feels obvious, jump to [`etcd-basics.md`](./deep-dive.md). The deep-dive covers:

- Raft consensus mechanics in detail
- Read vs write quorum
- Performance tuning (NVMe, `--heartbeat-interval`, `--election-timeout`)
- The 8GB quota: diagnosis and defragmentation
- `etcdctl` command reference (snapshot, member list, endpoint health, defrag)
- Backup/restore patterns
- 4 self-test drills + 4-dimensions framing
