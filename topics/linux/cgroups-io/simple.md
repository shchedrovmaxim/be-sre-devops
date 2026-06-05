# cgroups IO — the simple version (the highway toll booth analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **IO throttling in the kernel works like a toll booth that limits how many cars (bytes/ops) can pass through per second — and EBS adds its own separate toll booth above the kernel's.**

That's the whole picture. When a pod "feels slow" on IO, you have to figure out which toll booth is the bottleneck.

---

## Two toll booths, one road

Imagine a highway (your disk IO path) with two toll booths in sequence:

**Toll booth 1 (the kernel cgroup):** Set by `io.max` in cgroup v2. Enforced for *every process in the container*. The limit is per device, in IOPS and bandwidth. If you exceed the limit, the kernel queues your requests and drains them slowly.

**Toll booth 2 (EBS):** Set by AWS, entirely independent of the kernel. EBS gp3 volumes have a baseline of 3,000 IOPS and 125 MB/s — guaranteed. Go above that, and AWS throttles you at the storage level. The kernel doesn't know. Your app doesn't know. Requests just start taking 10–20× longer.

Most "the pod is slow on IO" conversations are about **toll booth 2**, not toll booth 1 — because K8s doesn't enable per-pod IO limits by default. The cgroup IO limit is there, but nobody set it. EBS throttling, on the other hand, happens automatically the moment a pod spikes past the volume's budget.

---

## The 2 confusing things, demystified

### 1. `io.max` — what's in the file

The v2 IO limit file lives at:

```
/sys/fs/cgroup/<pod-cgroup>/io.max
```

It looks like:

```
8:0 rbps=52428800 wbps=52428800 riops=1000 wiops=1000
```

Breaking that apart:
- `8:0` — the device. Major:minor number. `8` = SCSI/SATA/NVMe, `0` = first partition.
- `rbps` — read bytes per second. 52428800 = 50 MB/s.
- `wbps` — write bytes per second.
- `riops` — read IOPS.
- `wiops` — write IOPS.

The `max` keyword means no limit:
```
8:0 rbps=max wbps=max riops=max wiops=max
```

**In practice**: K8s does NOT populate `io.max` for normal pods. It's `max` for everything. Per-pod IO limits require extra work (NRI plugins, custom admission webhooks).

### 2. The EBS burst credit trap

EBS gp2 volumes (older type) had a burst credit system — like a battery. At idle, the battery charges. Under IO load, it drains. When it's empty, you're throttled to the baseline: **100 IOPS for small volumes**.

gp3 replaced gp2 with a simpler model: 3,000 IOPS baseline, always. No burst credit. But gp2 volumes still exist widely in production, and when that battery dies mid-incident, diagnosing it requires knowing this story.

The tell: a burst-credit-exhausted gp2 volume goes from fast to suddenly 100 IOPS cold — with zero kernel-level warning.

---

## How to map device major:minor to actual disks

You see `8:0` in `io.max` but don't know which disk that is. Fix:

```bash
ls -la /dev/disk/by-id/
# or:
lsblk
# NAME   MAJ:MIN  ...
# sda      8:0    ← that's your disk
# nvme0n1 259:0   ← NVMe (common on modern EC2)
```

For NVMe (most EC2 instance types): major number is `259`, not `8`.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| Does K8s set per-pod IO limits by default? | No. `io.max` is `max` for all containers unless you add extra tooling. |
| Where does EBS throttling happen? | At the EBS service layer, above the OS. The kernel can't see it happening. |
| What's gp3 baseline? | 3,000 IOPS, 125 MB/s. Always, no burst mechanic. |
| What was gp2's problem? | Burst credits. At 100 GB, you got 300 IOPS baseline. Burst to 3,000 until credits empty. |
| How do I see EBS throttling? | CloudWatch metric `VolumeQueueLength` spikes, or `aws cloudwatch get-metric-statistics` on `VolumeThroughputPercentage`. |
| Where do I look in the kernel? | `/sys/fs/cgroup/<pod>/io.stat` for current IO stats; `/sys/block/<dev>/stat` for device-level queue. |
| What's blkio? | The v1 name for the IO controller. v2 renamed it to `io`. Same concept, different file names. |

---

## Self-test (the killer question)

Out loud:

> **"Your pod is using 30% of its IOPS budget but apps feel slow. What might be happening with cgroup IO or EBS throttling?"**

**Reference answer (intuitive version):**

"First I'd clarify whether there even IS a cgroup IO limit set — in K8s, pods don't get `io.max` limits by default, so the 30% figure might come from somewhere else, like an application-level metric. If there is a kernel-level limit, I'd check `io.stat` in the cgroup directory to see if there's IO waiting or queue depth building up.

But the more likely culprit for 'low apparent usage, slow IO' is EBS throttling. The EBS volume has its own IOPS and bandwidth ceiling that's completely independent of what the kernel sees. A gp3 volume is capped at 3,000 IOPS by default — if the pod is doing bursty IO and another pod on the same node is also hitting the same volume, they share that ceiling. The kernel sees 30% of its cgroup quota used, but at the EBS layer the volume is already saturated.

I'd look at CloudWatch's `VolumeQueueLength` metric for the EBS volume — if it's climbing, requests are queueing at AWS. That's the EBS throttle signal."

---

## Further reading / watching

- **AWS EBS I/O characteristics**: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-io-characteristics.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-io-characteristics.html) — the authoritative gp2/gp3 numbers.
- **Kernel docs — IO controller (v2)**: [kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#io](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html) — scroll to the IO section.

---

## Next: the deep-dive

When the toll booth analogy feels obvious, jump to [`cgroups-io.md`](./deep-dive.md). The deep-dive covers:

- `io.max` file format in full detail, including how to set it and read `io.stat`
- v1 `blkio` controller vs v2 `io` controller — the differences
- How to map device major:minor to real disks (including NVMe on EC2)
- EBS gp2 vs gp3 in depth — burst credits, IOPS ceiling, per-instance caps
- Why K8s doesn't enable per-pod IO limits by default
- The NRI plugin / pod-level IO QoS path
- 4 self-test drills
- The 4 dimensions framing
