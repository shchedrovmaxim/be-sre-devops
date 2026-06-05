# etcd basics — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through what you need to know about etcd to be on-call for a self-managed K8s cluster."** — naming Raft quorum math, disk requirements, the 8GB quota gotcha, snapshot procedures, and when managed K8s makes this irrelevant.

> Start with the [simple version](./etcd-basics-simple.md) if you haven't read it. The city hall registry analogy is the spine.

---

## The senior framing — etcd IS the control plane

The K8s apiserver is often called the "brain of the cluster" but that's slightly wrong. The apiserver is a stateless process. Every piece of cluster state — pod specs, node registrations, ConfigMaps, Secrets, RBAC bindings — lives in etcd.

If the apiserver crashes, it restarts and re-reads everything from etcd. The cluster keeps running.

If etcd is corrupted or lost, the apiserver has nothing to read. It can't schedule pods, it can't serve `kubectl`, it can't know what nodes exist. The cluster is blind, even if every workload pod is still running happily.

**This is the "control plane is etcd" mental model**: the apiserver is just a translation layer and enforcer. The state lives in etcd.

---

## Raft consensus — how etcd stays consistent

etcd uses the **Raft consensus algorithm** to ensure that all members agree on every write before committing it.

### The leader-follower model

At any time, one etcd member is the **leader**. All writes go through the leader. The leader:
1. Writes the entry to its own log
2. Sends the entry to all followers (AppendEntries RPC)
3. Waits for a **quorum** of followers to acknowledge
4. Commits the entry to the state machine
5. Returns the ACK to the caller (the K8s apiserver)

Followers replicate the log from the leader. If the leader dies, a new leader election occurs.

### Quorum math

```
quorum = floor(n/2) + 1

n=1: quorum=1, tolerates 0 failures
n=3: quorum=2, tolerates 1 failure
n=5: quorum=3, tolerates 2 failures
n=7: quorum=4, tolerates 3 failures
```

**Why odd numbers**: with 4 members, quorum is 3. If 2 fail, you have 2 remaining — not enough for quorum. So 4 members tolerates 1 failure, same as 3 members. 4 is strictly worse than 3 (more infrastructure, same fault tolerance). Same argument applies to 6 vs 5.

**Read quorum**: etcd can optionally serve reads from followers (linearizable reads still go through the leader; serializable reads can go to any member). By default, all reads are linearizable (leader-routed) for strong consistency.

### Leader election

Election is triggered when a follower doesn't hear from the leader for `election-timeout`. Default is 1000ms (1 second). Under a slow network or slow disk, followers may incorrectly trigger elections, causing instability.

```
--heartbeat-interval=100ms    # default; leader sends heartbeat every 100ms
--election-timeout=1000ms     # default; follower starts election after 1000ms of silence
```

**Rule**: `election-timeout` should be at least 5× `heartbeat-interval`. Don't tune these aggressively unless you understand the trade-offs.

---

## Performance requirements

### The fsync constraint

Every committed write in etcd is **fsynced to disk** before ACK. This means write latency is directly determined by your disk's fsync latency. A typical enterprise SSD has 100-200µs fsync latency. A 7200 RPM HDD has 5-10ms. On HDDs, etcd's effective write throughput collapses and leader election timeouts become common.

**The requirement**: etcd nodes must have **fast local SSD**. NVMe (PCIe) is the gold standard (~50-100µs fsync). SATA SSD is acceptable (~200-400µs). Spinning disk is not acceptable. Network-attached storage (EBS, EFS, NFS) adds latency jitter that causes election timeouts — avoid completely.

**AWS gotcha**: EKS managed nodes run on EC2. If you're managing your own etcd on AWS, use instance storage (NVMe ephemeral) or `io2 Block Express` EBS. Standard `gp3` EBS has acceptable latency for etcd in most cases, but `gp2` can be throttled at low volume sizes.

### Benchmark before you commit

```bash
# fio benchmark for etcd-like workload
fio --rw=write --ioengine=sync --fdatasync=1 --directory=/var/lib/etcd \
    --size=22m --bs=2300 --name=etcd-benchmark
# Look for "sync lat" — should be well under 10ms median
```

etcd also ships its own benchmark tool:

```bash
etcd-bench put --total=10000 --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/server.crt --key=/etc/etcd/server.key
```

---

## The 8GB quota wall

etcd has a default storage limit: **8GB** (`--quota-backend-bytes=8589934592`). When it hits the limit:

- Every write returns: `etcdserver: mvcc: database space exceeded`
- `kubectl apply`, `kubectl create` — all fail
- Cluster is effectively frozen (existing workloads keep running; no new scheduling decisions)

### What causes it

1. **Helm release history**: by default, Helm stores every release revision as a Secret in etcd. 100 releases × 20 revisions = 2000 Secrets. Each release Secret can be hundreds of KB. Fix: `helm upgrade --history-max 5`.
2. **Large ConfigMaps**: a ConfigMap storing a large file (e.g., a bundled nginx config, an ML model) can be multi-MB. etcd is not a blob store — keep individual objects under ~1MB.
3. **Events**: K8s events are stored in etcd with a default TTL of 1 hour. In a busy cluster, events can pile up. Set a lower retention: `--event-ttl=30m` on the apiserver.
4. **CRD data**: large custom resources from operators (Flux, Argo, Vault) can accumulate.

### Diagnosis and fix

```bash
# Check current DB size (needs etcdctl access)
etcdctl endpoint status --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/server.crt --key=/etc/etcd/server.key \
  --write-out=table
# Look for "DB SIZE" column

# Find largest keys (identify what's taking space)
etcdctl get "" --prefix --keys-only \
  | awk -F'/' '{print NF, $0}' | sort -rn | head -50

# Compact the revision history
LATEST=$(etcdctl endpoint status --write-out=json \
  | jq -r '.[0].Status.header.revision')
etcdctl compact "$LATEST"

# Defragment (reclaim disk space)
etcdctl defrag --endpoints=https://127.0.0.1:2379 [same certs]

# After defrag, the alarm should clear automatically; if not:
etcdctl alarm disarm
```

**Proactive monitoring**: alert when etcd DB size exceeds 6GB (75% of 8GB quota). This gives you time to defrag before hitting the wall.

---

## etcdctl command reference

```bash
# Health check
etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key

# Member list (shows who's in the cluster and their leader status)
etcdctl member list \
  --endpoints=https://127.0.0.1:2379 [same certs] \
  --write-out=table

# Snapshot save (backup)
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 [same certs]

# Snapshot status (verify)
etcdctl snapshot status /backup/etcd-20240101-120000.db --write-out=table
# Outputs: hash, revision, total keys, total size

# Get a key (useful for debugging)
etcdctl get /registry/pods/default/my-pod \
  --endpoints=https://127.0.0.1:2379 [same certs]

# Watch a key (real-time, useful for debugging controller behavior)
etcdctl watch /registry/pods/default/ --prefix \
  --endpoints=https://127.0.0.1:2379 [same certs]
```

---

## Backup and restore patterns

### Backup

```bash
# Run on the etcd leader node (or any member — snapshot is consistent)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Automate this with a CronJob (on self-managed) or a Velero backup (which can backup etcd among other things).

### Restore (kubeadm cluster)

Restore is a multi-step process that requires stopping all control plane components:

```bash
# 1. Stop kube-apiserver, kube-scheduler, kube-controller-manager
# (on a kubeadm cluster, move their static pod manifests out of /etc/kubernetes/manifests)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# 2. Stop etcd
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 3. Move the old etcd data dir out of the way
mv /var/lib/etcd /var/lib/etcd.bak

# 4. Restore the snapshot
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --initial-cluster="etcd-1=https://10.0.0.1:2380" \
  --initial-cluster-token=etcd-cluster \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --name=etcd-1

# 5. Bring etcd back up (restore the manifest)
mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 6. Wait for etcd to become healthy, then restore the other components
```

**Important**: restore must happen on ALL etcd members for the cluster to have a consistent state. If you restore on only 1 of 3 members, the cluster will be in a split-brain state.

---

## What managed K8s hides from you

| Concern | Self-managed (kubeadm) | Managed (EKS/GKE/AKS) |
|---|---|---|
| etcd instance management | Your problem | Provider's problem |
| Backup | You set it up | Provider handles (EKS: automatic; GKE: automatic) |
| Disk performance | You choose (NVMe, local SSD) | Provider manages |
| Quota management | You monitor the 8GB limit | Provider manages |
| etcdctl access | Direct access on control plane nodes | No direct access |
| Quorum management | You ensure 3/5 members | Provider manages |

On EKS, GKE, and AKS: etcd is invisible. You can't `etcdctl` to it. Backups are handled. The SLA covers it. You only worry about etcd if you're on self-managed K8s.

The CKA exam expects you to know etcd backup/restore for self-managed clusters. The production SRE interview expects you to know when it matters (self-managed) and when it doesn't (managed).

---

## The interview answer in 60 seconds

> "etcd is the single source of truth for all K8s state. The apiserver is stateless — if it crashes, it comes back and reads from etcd. If etcd is lost, the cluster is blind. On-call for self-managed K8s, the things I'd care about are: backups — automated `etcdctl snapshot save` every few hours, verified, stored off-cluster; the 8GB quota wall — etcd stops accepting writes when it hits the limit, so I'd alert at 75% usage and run `etcdctl defrag` proactively; disk performance — etcd fsyncs every write, so it needs fast local SSD (NVMe), not HDD or network storage; and quorum — three members across three AZs, you can lose one AZ and still have a majority. If two members are gone simultaneously, the cluster freezes on writes until you restore from snapshot. On EKS or GKE, all of this is the cloud provider's responsibility and I don't worry about it."

---

## Self-test drills

### 1. Walk me through what you need to know about etcd to be on-call for a self-managed cluster.

**Reference answer**: backups (snapshot save), 8GB quota (monitor + defrag), disk speed (NVMe local SSD, no network storage), quorum (3 members, can lose 1). Restore procedure requires stopping control plane components, restoring on all members, bringing back up. See 60-second answer above.

### 2. Your etcd writes are all failing with "database space exceeded." Walk me through diagnosing and fixing it.

**Reference answer**: check DB size with `etcdctl endpoint status`; find largest keys to identify the source (Helm history, large ConfigMaps, events); compact the revision history (`etcdctl compact $LATEST`); defrag (`etcdctl defrag`); alarm disarm. Long-term: set `--history-max 5` for Helm, limit event TTL on apiserver, break up large ConfigMaps.

### 3. Why does etcd need fast local SSD? What happens on slow disk?

**Reference answer**: every write is fsynced to disk before ACK. Disk latency is directly on the write path. Slow disk (HDD, network storage) causes: high write latency → followers think the leader is dead → spurious leader elections → apiserver timeouts → slow `kubectl` operations. NVMe (<100µs fsync) is the standard. Never use NFS or EBS without `io2` for etcd.

### 4. Explain Raft quorum. Why 3 or 5 members, not 4 or 6?

**Reference answer**: quorum = floor(n/2)+1. With 4 members, quorum is 3 — tolerate 1 failure. With 3 members, quorum is also 2 — tolerate 1 failure. Same fault tolerance, but 4 members requires more infrastructure. Even numbers add nodes without improving tolerance. Use 3 for most clusters, 5 for higher availability needs.

---

## Further reading

- [etcd documentation](https://etcd.io/docs/)
- [etcd disaster recovery in K8s docs](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Raft consensus paper (Ongaro and Ousterhout)](https://raft.github.io/raft.pdf)
- [etcd performance benchmarking guide](https://etcd.io/docs/v3.5/op-guide/performance/)

---

## The 4 dimensions (senior framing)

- **Tech**: Raft consensus (quorum = floor(n/2)+1); odd members (3 or 5); leader/follower model; fsync on every write → NVMe required; 8GB default quota; `etcdctl` commands (snapshot, member list, endpoint health, compact, defrag); backup/restore procedure for kubeadm clusters; managed K8s abstracts all of this.
- **People**: etcd is the single most critical component in a self-managed cluster — one wrong move in a restore procedure corrupts the entire cluster. Restrict direct etcd access to a small team (ops lead + SRE). Run restore drills quarterly in a non-production environment. Document the restore procedure step-by-step so a panicking on-call engineer can follow it at 3am.
- **CI/CD**: on self-managed clusters, automate backups as a CronJob. Verify backups by periodically restoring to a test cluster and checking cluster object counts. Alert on backup job failures. Before every K8s upgrade, take a manual snapshot — upgrades can corrupt etcd if something goes wrong mid-way.
- **Operations**: monitor: etcd DB size (alert at 75% quota), leader election frequency (should be near zero; frequent elections indicate disk/network problems), peer RTT (should be <1ms in the same AZ), write latency (p99 should be <25ms). Runbook: "etcd quota exceeded" → compact + defrag → alarm disarm. Runbook: "frequent leader elections" → check disk fsync latency with `fio`, check network RTT between members.
