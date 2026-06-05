# cgroups IO controller + EBS throttling

> **Goal**: by the end you can answer the killer question — **"Your pod is using 30% of its IOPS budget but apps feel slow. What might be happening with cgroup IO or EBS throttling?"** — distinguishing kernel-level IO limits from EBS-layer throttling, understanding gp2 burst credits vs gp3 baseline, and knowing which signals to check first.

> Start with the [simple version](./cgroups-io-simple.md) if you haven't — the toll booth analogy makes the layering clear.

---

## The senior framing

IO is the most commonly misdiagnosed performance problem in K8s/EKS, because there are **two independent throttle mechanisms** in the data path and most engineers only know one of them.

The kernel's cgroup IO controller (`io.max` in v2, `blkio` in v1) limits how much IO a cgroup can issue per second. That's toll booth 1. But in practice, K8s doesn't configure this for normal pods — `io.max` is `max` (unlimited) by default. So toll booth 1 is usually wide open.

Toll booth 2 is EBS. AWS enforces IOPS and throughput limits at the storage service layer, below the kernel's view. When a pod's IO volume saturates the EBS ceiling, requests queue at the storage layer. The kernel submits them, they complete slowly, but there's no kernel-level throttle event — the kernel thinks IO is "in flight." `io.stat` looks fine. `cpu.stat` looks fine. The app just slows down.

Knowing this two-layer model is the senior answer.

---

## The cgroup v2 IO controller

### `io.max` — the per-device limit

The file format:

```
<major>:<minor> rbps=<value> wbps=<value> riops=<value> wiops=<value>
```

Example — a container limited to 50 MB/s read/write and 1,000 IOPS read/write on device `8:0`:

```bash
$ cat /sys/fs/cgroup/kubepods.slice/.../io.max
8:0 rbps=52428800 wbps=52428800 riops=1000 wiops=1000
```

Fields:
- `rbps` — read bytes per second
- `wbps` — write bytes per second
- `riops` — read IO operations per second
- `wiops` — write IO operations per second
- Value `max` means no limit for that field

Setting a limit (must be root, cgroup v2):

```bash
echo "8:0 rbps=10485760 wbps=10485760" > /sys/fs/cgroup/mygroup/io.max
# Limits to 10 MB/s read and write on device 8:0
```

### Mapping device major:minor to real disks

The `8:0` notation means major=8, minor=0. Major 8 = sd* (SCSI/SATA). Minor 0 = first device (sda), minor 16 = sdb, etc.

On most modern EC2 instance types, volumes appear as NVMe — major **259**:

```bash
lsblk
# NAME       MAJ:MIN RO  SIZE ...
# nvme0n1    259:0       20G    ← EBS root volume
# nvme1n1    259:16      100G   ← additional EBS volume
```

To find which device a mount point uses:

```bash
df /var/lib/kubelet       # shows mount device e.g. /dev/nvme0n1p1
stat -c %t:%T /dev/nvme0n1   # gives hex major:minor
printf "%d:%d\n" 0x103 0x0   # convert to decimal
```

Or more directly:

```bash
cat /sys/block/nvme0n1/dev   # → 259:0
```

### `io.stat` — current IO statistics per cgroup

```bash
$ cat /sys/fs/cgroup/kubepods.slice/.../io.stat
8:0 rbytes=1048576000 wbytes=2097152000 rios=10240 wios=20480 dbytes=0 dios=0
```

Fields:
- `rbytes`/`wbytes` — total bytes read/written since cgroup creation
- `rios`/`wios` — total IO operations read/written
- `dbytes`/`dios` — discard bytes/operations (TRIM/UNMAP)

This is cumulative. To get current rate, diff it over time — or use `iostat -x 1` at the block device level for real-time view.

### `io.pressure` — PSI for IO

```bash
$ cat /sys/fs/cgroup/kubepods.slice/.../io.pressure
some avg10=0.50 avg60=0.12 avg300=0.02 total=1234567
full avg10=0.02 avg60=0.00 avg300=0.00 total=12345
```

`some` means at least one task was stalled waiting for IO. `full` means all tasks were stalled — the bad state. If `io.pressure some avg10` is climbing, something in this cgroup is IO-bound.

---

## The v1 `blkio` controller — key differences

In cgroup v1, IO limiting was done by the `blkio` controller with a very different interface:

| Concept | v2 file | v1 file |
|---|---|---|
| Read IOPS limit | `io.max` (`riops=N`) | `blkio.throttle.read_iops_device` |
| Write IOPS limit | `io.max` (`wiops=N`) | `blkio.throttle.write_iops_device` |
| Read BPS limit | `io.max` (`rbps=N`) | `blkio.throttle.read_bps_device` |
| Write BPS limit | `io.max` (`wbps=N`) | `blkio.throttle.write_bps_device` |
| IO weight | `io.weight` | `blkio.weight` |
| IO accounting | `io.stat` | `blkio.throttle.io_serviced` + `blkio.throttle.io_service_bytes` |

The v1 `blkio.weight` provided proportional IO scheduling (range 10–1000, default 500) — similar to `cpu.shares` for CPU. v2 keeps this as `io.weight` (1–10000, default 100).

**Key v1 gotcha**: in v1, `blkio.throttle.read_iops_device` required you to specify limits for each device separately AND you had to specify both read and write in separate files. Setting one didn't affect the other. It was verbose and easy to misconfigure.

---

## Per-pod IO limits are NOT enabled by default in K8s

This surprises most engineers. Despite the cgroup IO controller existing, Kubernetes **does not set `io.max`** for pods by default. Every container gets:

```
259:0 rbps=max wbps=max riops=max wiops=max
```

Why? Because the KEP (Kubernetes Enhancement Proposal) for per-pod IO limits (KEP-1287) requires knowing which block device backs each volume, which varies per storage driver, per CSI plugin, per EBS volume type. The kubelet doesn't have a clean way to plumb this through the general scheduling path.

The current path for IO limits:

1. **Directly via cgroup**: write to `io.max` manually (operational hack, not K8s-native)
2. **NRI (Node Resource Interface) plugin**: a containerd/CRI-O plugin hook that can set cgroup values before a container starts. The most practical current approach.
3. **Extended resource + device plugin**: emerging pattern; not widely deployed yet.
4. **Kata Containers**: runs pods in lightweight VMs; the VM's own disk device can be limited by the hypervisor.

In practice, most teams deal with EBS throttling at the volume provisioning level (right-sizing the volume's IOPS) rather than at the cgroup level.

---

## EBS throttling — the real-world IO bottleneck

### gp2 vs gp3 — the numbers that matter

| Volume type | Baseline IOPS | Burst IOPS | Burst duration | Baseline throughput |
|---|---|---|---|---|
| gp2 | 3 IOPS/GB (min 100) | 3,000 | Until burst credits exhausted | 128–250 MB/s depending on size |
| gp3 | 3,000 (always, configurable up to 16,000) | No burst mechanic | — | 125 MB/s baseline (configurable up to 1,000 MB/s) |

**gp2 gotcha — the burst credit drain:**

A 100 GB gp2 volume has a 100 × 3 = **300 IOPS baseline**. It can burst to 3,000 IOPS. Credits accumulate at baseline rate when IO is below baseline. They deplete at (actual IOPS − baseline) rate when IO exceeds baseline.

A busy pod doing 3,000 IOPS on a 100 GB gp2 volume depletes at `3000 - 300 = 2700 IOPS` net drain rate. At that rate, burst credits (5.4 million I/O credits) last about 33 minutes. After that: hard throttle to **300 IOPS** with no warning.

This looks like: "the service was fine all morning, then at 10:47am everything went to hell and the pod appeared healthy but all DB queries were slow." That's burst credit exhaustion.

**gp3 is simpler**: 3,000 IOPS always, no credit mechanic. If you need more, provision up to 16,000 IOPS directly on the volume. Pay per provisioned IOPS above 3,000.

### The instance-level IO ceiling

EBS throttling can also happen at the **EC2 instance level**, not just the volume level. Each EC2 instance type has a maximum EBS aggregate bandwidth and IOPS ceiling across all attached volumes.

Example: `m5.large` = 4,750 Mbps EBS bandwidth, 18,750 IOPS total. If your instance has two EBS volumes each capable of 3,000 IOPS, the instance can still saturate at 18,750 IOPS total across both.

CloudWatch metrics to check:
- `EBSReadOps` / `EBSWriteOps` (per volume, from CloudWatch)
- `EBSReadBytes` / `EBSWriteBytes`
- `VolumeQueueLength` — average number of read/write operations pending. If this is > 1 and rising, the volume is saturated.
- `VolumeThroughputPercentage` — how close to the IOPS ceiling you are (0–100%)

### Diagnosing EBS throttling

```bash
# On the node:
iostat -x 1 /dev/nvme0n1
# Look at: await (average wait time in ms), %util (utilization)
# High await with moderate %util = requests queuing = throttle

# CloudWatch (from your laptop or CI):
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeQueueLength \
  --dimensions Name=VolumeId,Value=vol-0abc123def456789 \
  --start-time 2026-01-01T10:00:00Z \
  --end-time 2026-01-01T11:00:00Z \
  --period 60 \
  --statistics Average
```

If `VolumeQueueLength` is consistently > 0.1 and rising, you have EBS throttle.

---

## The "30% IOPS but slow" diagnosis flow

When someone says "the pod is only using 30% of its IOPS budget but IO feels slow," the investigation order:

1. **Clarify the 30% figure**: is that a cgroup `io.max` limit, or an application-level metric, or the volume's provisioned IOPS? Most likely it's the volume's provisioned IOPS.
2. **Check `io.max`**: is there actually a cgroup IO limit set? For most K8s pods: no. If `io.max` shows `max`, the kernel isn't the bottleneck.
3. **Check `io.pressure`**: is `io.pressure some avg10` non-zero for the pod's cgroup? If so, processes are stalling on IO.
4. **Check `iostat` on the node**: `await` > 10ms on the device suggests queuing. `%util` close to 100% means saturated disk (rare on EBS but happens on instance store).
5. **Check CloudWatch `VolumeQueueLength`**: the primary EBS throttle signal. If it's climbing, you're hitting the EBS ceiling.
6. **Check burst credit balance** (if gp2): CloudWatch `BurstBalance` metric. If it's low or dropping, you're on borrowed time.
7. **Check the instance IOPS ceiling**: if the node is running many IO-heavy pods, the EC2 instance aggregate limit might be the actual ceiling.

---

## The interview answer in 60 seconds

> "I'd start by clarifying what '30% IOPS budget' means — that figure could come from the cgroup `io.max` file, the volume's provisioned IOPS, or an application metric. Most likely it's the volume IOPS.
>
> Then I'd check two layers. First, the kernel layer: does the pod's cgroup actually have an `io.max` limit set? In vanilla Kubernetes, the answer is no — `io.max` is `max` for all pods by default, because K8s doesn't set per-pod IO limits without additional tooling. So the kernel probably isn't throttling anything.
>
> Second, the EBS layer: check CloudWatch `VolumeQueueLength` for the EBS volume. If that's climbing, requests are queuing at the storage service level — the kernel submits them, AWS queues them, they complete slowly, but no kernel-level throttle event fires. The kernel just sees 'in flight IO.'
>
> If it's a gp2 volume, I'd also check `BurstBalance` — burst credits drain at high IOPS load and if they hit zero, you get throttled hard to the baseline (often 300 IOPS for small volumes). gp3 doesn't have this problem; baseline is always 3,000 IOPS.
>
> The fix is usually: upgrade to gp3, provision more IOPS on the volume, or spread the load across multiple volumes."

---

## Self-test drills

### 1. Your pod is using 30% IOPS budget but apps feel slow. What might be happening with cgroup IO or EBS throttling?

**Reference answer:**
- Check if `io.max` is actually set — likely `max` (unlimited) in K8s by default
- Check `io.pressure some avg10` in the pod cgroup — if non-zero, tasks are stalling
- Check CloudWatch `VolumeQueueLength` — the primary EBS throttle signal
- If gp2: check `BurstBalance` — credits may be depleted
- If gp3: check if provisioned IOPS (default 3,000) is sufficient
- Also check EC2 instance-level EBS IOPS ceiling in CloudWatch `EBSReadOps`/`EBSWriteOps`

### 2. What's the difference between `io.max` and `blkio.throttle.*` in v1?

**Reference answer:**
- v2 `io.max`: single file, space-separated, all four limits (rbps/wbps/riops/wiops) per device on one line
- v1 `blkio.throttle.*`: four separate files per device (read_bps_device, write_bps_device, read_iops_device, write_iops_device)
- v1 also had `blkio.weight` for proportional IO (v2 renamed to `io.weight`)
- v2 additionally surfaces IO accounting in `io.stat` and PSI in `io.pressure` — neither existed cleanly in v1

### 3. Why doesn't K8s set per-pod IO limits by default?

**Reference answer:**
- The cgroup IO controller limits by device (major:minor) — the kubelet would need to know which block device backs each pod's volumes
- This varies by storage driver, CSI plugin, and volume type — no clean generic mapping
- Current path: NRI plugins can inject `io.max` before container start, or set limits post-start via cgroup directly
- KEP-1287 tracks the feature; not GA as of mid-2026

### 4. What are the gp3 baseline IOPS and throughput numbers?

**Reference answer:**
- gp3 baseline: **3,000 IOPS** and **125 MB/s** throughput — always, no burst mechanic
- Configurable up to **16,000 IOPS** and **1,000 MB/s** throughput (paid per provisioned unit above baseline)
- gp2 comparison: 3 IOPS/GB baseline (100 GB = 300 IOPS), burst to 3,000 IOPS until credits drain
- Key interview point: gp2 burst credit exhaustion is a common "works fine until it suddenly doesn't" failure mode; gp3 eliminates it

---

## Further reading / watching

- **AWS EBS I/O characteristics**: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-io-characteristics.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-io-characteristics.html)
- **AWS EBS volume types (gp2 vs gp3)**: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html)
- **Kernel docs — IO controller**: [kernel.org/doc/html/latest/admin-guide/cgroup-v2.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html) — search for "IO" section
- **KEP-1287 (per-pod IO limits)**: [github.com/kubernetes/enhancements/issues/1287](https://github.com/kubernetes/enhancements/issues/1287)

---

## The 4 dimensions (senior framing)

- **Tech**: cgroup v2 `io.max` per device (major:minor), `io.stat` for cumulative accounting, `io.pressure` for PSI stall metrics; EBS CloudWatch metrics `VolumeQueueLength` and `BurstBalance`; gp3 = 3,000 IOPS baseline no-burst; gp2 = burst credits that drain; instance-level EBS ceiling.
- **People**: "the pod is slow" reports from app teams often turn into a tour of every layer (app → kernel → storage). Having a clear runbook ("check these 6 things in order") stops the wild goose chase. Educate developers that IOPS limits are two separate things: the cgroup one they can see, and the AWS one that's invisible until CloudWatch.
- **CI/CD**: load tests should run against the same volume type as production. A local or CI test against instance store gives no EBS signal. If your service is IO-heavy, add a CloudWatch `VolumeQueueLength` check to your load test assertion.
- **Operations**: add CloudWatch alarms on `VolumeQueueLength > 1` for all EBS volumes attached to production nodes. For gp2 volumes still in prod, add `BurstBalance < 20%` alarm — that's the "30 minutes until throttle" warning. Consider migrating gp2 → gp3 as a reliability improvement with predictable cost.
