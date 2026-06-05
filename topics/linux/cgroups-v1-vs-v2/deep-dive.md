# cgroups v1 vs v2 — the unified hierarchy

> **Goal**: by the end you can answer the killer question — **"Why did cgroups v2 win, and what does the unified hierarchy actually mean for K8s?"** — covering the v1 multi-hierarchy design flaw, v2's unified model, PSI, how to detect which version a host runs, and the kubelet `--cgroup-driver=systemd` story.

> Start with the [simple version](./simple.md) if you haven't — the USB hub analogy makes the technical detail below stick.

---

## The senior framing

This isn't a trivia question about version numbers. The v1 → v2 transition is about **moving consistency enforcement from the container runtime into the kernel**.

With v1, Docker, containerd, and the kubelet each maintained their own mapping of "process → cgroup subtree per controller." They had to stay in sync. They sometimes didn't. Bugs in Kubernetes node resource accounting, OOM eviction, and IO QoS all trace back to the fundamental v1 design: the kernel offers 12 separately-mountable controller hierarchies, and userspace has to stitch them together.

v2 is the kernel saying: "fine, I'll own the consistency."

---

## cgroups v1 — the multi-hierarchy design

### How v1 works

In v1, every resource controller (memory, cpu, cpuacct, blkio, devices, freezer, net_cls, net_prio, pids, hugetlb, rdma, perf_event) is a **separate subsystem** that can be mounted on its own hierarchy.

A process is tracked in each hierarchy independently. The classic mount layout:

```
/sys/fs/cgroup/
├── memory/        ← memory controller hierarchy
│   ├── mycontainer/
│   │   ├── memory.limit_in_bytes
│   │   └── memory.usage_in_bytes
├── cpu,cpuacct/   ← cpu + cpuacct mounted together
│   ├── mycontainer/
│   │   └── cpu.shares
├── blkio/         ← block IO controller hierarchy
│   ├── mycontainer/
│   │   └── blkio.weight
└── pids/
    └── mycontainer/
        └── pids.max
```

A process living in `/memory/mycontainer/` is tracked for memory. If it's also in `/cpu/mycontainer/` it has CPU accounting. **These are separate trees.** You can put a process at `/memory/team-a/` but `/cpu/team-b/` — the kernel doesn't care. It enforces each controller independently.

### The real-world pain points

**Inconsistency between controllers.** kubelet v1-era code would create a cgroup at `/sys/fs/cgroup/memory/kubepods/pod-<uid>/container-<cid>/` and separately at `/sys/fs/cgroup/cpu/kubepods/pod-<uid>/container-<cid>/`. If the kubelet crashed mid-creation, you could end up with a container whose memory was tracked but whose CPU wasn't.

**No single concept of "this container".** The container runtime had to maintain an external record of which cgroup path corresponded to which container, across controllers. Leaks from one controller that weren't cleaned up from another caused ghost resource accounting.

**`cpuacct` was a separate controller.** If you forgot to mount it alongside `cpu`, you had no CPU usage accounting. Many distros mounted them together (`cpu,cpuacct`) by convention, but it was userspace convention, not kernel enforcement.

**The "named hierarchy" escape hatch.** In v1, you could mount a hierarchy with no controller — just a named tree for organizational purposes. This was widely used but also widely confused.

### Relevant v1 file names

| Resource | v1 file | Notes |
|---|---|---|
| CPU shares | `cpu.shares` | Range 2–262144, default 1024 |
| CPU quota | `cpu.cfs_quota_us` | In microseconds; `-1` = no limit |
| CPU period | `cpu.cfs_period_us` | In microseconds; default 100000 |
| Memory limit | `memory.limit_in_bytes` | Hard limit in bytes |
| Memory soft | `memory.soft_limit_in_bytes` | Soft limit |
| Block IO weight | `blkio.weight` | 10–1000, default 500 |
| PIDs | `pids.max` | Max processes in the group |

---

## cgroups v2 — the unified hierarchy

### The fundamental change

v2 has exactly **one** hierarchy, mounted at `/sys/fs/cgroup/`. All controllers live under that single tree. A process is in exactly one place in that tree. You cannot split a process across controllers in different subtrees — the concept doesn't exist.

```
/sys/fs/cgroup/
├── kubepods.slice/
│   ├── kubepods-burstable.slice/
│   │   └── kubepods-burstable-pod<uid>.slice/
│   │       └── cri-containerd-<cid>.scope/
│   │           ├── memory.max        ← memory limit
│   │           ├── memory.high       ← memory soft limit
│   │           ├── cpu.max           ← cpu quota + period
│   │           ├── cpu.weight        ← cpu shares
│   │           ├── io.max            ← IO limits
│   │           ├── pids.max          ← pid limit
│   │           ├── memory.pressure   ← PSI for memory
│   │           ├── cpu.pressure      ← PSI for cpu
│   │           └── io.pressure       ← PSI for IO
```

One directory. All resource controls and all metrics in the same place.

### Key v2 rule: the "no-internal-process" constraint

In v2, **a cgroup that has child cgroups cannot have processes directly.** Processes live only in leaf cgroups. This sounds restrictive but it enforces clean hierarchy — you organize groups at inner nodes, run workloads at leaves.

v1 had no such constraint — processes could be scattered throughout the tree, which caused accounting confusion.

### v1 → v2 file name changes

| Resource | v1 file | v2 file | Notes |
|---|---|---|---|
| CPU hard cap | `cpu.cfs_quota_us` + `cpu.cfs_period_us` | `cpu.max` | Single file: `<quota> <period>` in µs |
| CPU weight | `cpu.shares` (2–262144, default 1024) | `cpu.weight` (1–10000, default 100) | Renamed and rescaled |
| Memory hard | `memory.limit_in_bytes` | `memory.max` | Cleaner name |
| Memory soft | `memory.soft_limit_in_bytes` | `memory.high` | Different semantics (high triggers reclaim, doesn't OOM) |
| Block IO limit | `blkio.throttle.*` | `io.max` | Unified read/write IOPS + bandwidth |
| Block IO weight | `blkio.weight` | `io.weight` | — |
| CPU accounting | `cpuacct.*` in separate mount | `cpu.stat` | Merged into cpu controller |

---

## Pressure Stall Information (PSI) — the v2 superpower

v1 could tell you resource *usage*. v2 adds resource *stall* — how much time tasks spent blocked waiting for a resource.

PSI gives two measurements per resource:

- **`some`**: the fraction of time during which at least one task in the cgroup was stalled
- **`full`**: the fraction of time during which ALL tasks in the cgroup were stalled (the severe case)

```bash
$ cat /sys/fs/cgroup/kubepods.slice/.../memory.pressure
some avg10=2.30 avg60=0.85 avg300=0.20 total=123456789
full avg10=0.10 avg60=0.02 avg300=0.00 total=12345678
```

Averages are in percent over 10-second, 60-second, and 300-second rolling windows. `total` is cumulative microseconds.

### Why PSI matters for K8s

Before PSI, the kubelet detected memory pressure by polling `memory.current` against `memory.max` and watching `memory.events`. That's reactive — you only know there's a problem after it caused something bad (OOM kill, throttle, latency spike).

PSI is proactive. A rising `memory.pressure some avg10` tells you the cgroup is starting to struggle before things fall off a cliff. K8s 1.27+ (with cgroup v2 and appropriate kubelet configuration) can use PSI to:
- Make better node-level eviction decisions
- Trigger proactive reclaim before OOM pressure builds

### PSI for CPU

CPU PSI works similarly:

```bash
$ cat /sys/fs/cgroup/kubepods.slice/.../cpu.pressure
some avg10=5.12 avg60=3.20 avg300=1.45 total=987654321
```

Note: CPU PSI only has `some` (not `full`) because CPU starvation never blocks *all* tasks — at least one task is always runnable on some CPU.

---

## Detecting v1 vs v2 on any host

```bash
stat -fc %T /sys/fs/cgroup
```

| Output | Meaning |
|---|---|
| `cgroup2fs` | Pure v2 (unified hierarchy only) |
| `tmpfs` | v1 (or mixed — check submounts) |

For a mixed system (some controllers still on v1):
```bash
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2     ← v2 unified
# cgroup on /sys/fs/cgroup/cpu type cgroup   ← v1 controller
```

### Distro defaults

| Distro | Default cgroup version | Notes |
|---|---|---|
| Ubuntu 20.04 | v1 | systemd 245 added v2 support but not default |
| Ubuntu 22.04+ | v2 | Switched default |
| RHEL 8 / CentOS 8 | v1 | |
| RHEL 9 / Rocky 9 | v2 | Switched default |
| Amazon Linux 2 (AL2) | v1 | Still widely used in EKS |
| Amazon Linux 2023 (AL2023) | v2 | Default for EKS 1.25+ new nodes |
| Bottlerocket | v2 | AWS-managed EKS optimized AMI; v2 always |
| Fedora 31+ | v2 | Early adopter |
| Debian 11+ | v2 | |

**EKS practical rule**: EKS 1.25+ with AL2023 or Bottlerocket = v2. AL2 nodes on any EKS version = v1. If you manage a mixed node group, you need to handle both.

---

## The kubelet `--cgroup-driver` story

The kubelet must create cgroup directories for pods and containers. It has two modes:

### `--cgroup-driver=cgroupfs` (legacy)

The kubelet directly writes to `/sys/fs/cgroup/` itself, creating directories and writing files. Works fine but creates a coordination problem: systemd also manages cgroups, and if both systemd and kubelet are creating and deleting cgroup paths, they can conflict. Leaked cgroups and resource accounting drift are the common failure modes.

### `--cgroup-driver=systemd` (preferred, and required for v2)

The kubelet tells systemd to create cgroup slices via the D-Bus API (`org.freedesktop.systemd1`). systemd creates:

```
kubepods.slice
└── kubepods-burstable.slice
    └── kubepods-burstable-pod<uid>.slice
        └── cri-containerd-<cid>.scope
```

systemd owns the cgroup tree and kubelet delegates to it. No conflicts. When a pod is deleted, systemd cleans up the slice properly — no leaked cgroups.

On a v2 host, `cgroupfs` driver technically works but is **not recommended** — systemd's v2 support is tight and circumventing it causes reconciliation drift. The K8s documentation says explicitly: on v2, use `systemd` driver.

You can verify the running kubelet's driver:
```bash
# On a node:
ps aux | grep kubelet | grep cgroup-driver
# or:
cat /var/lib/kubelet/config.yaml | grep cgroupDriver
```

### The containerd side

containerd has the same option in its config:
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Both the kubelet and containerd must agree on the cgroup driver. Mismatch = node notReady or mysterious pod failures.

---

## The "hybrid v1/v2" mode

Between v1 and full v2 adoption, many systems run **hybrid mode**: the unified v2 hierarchy at `/sys/fs/cgroup/`, but with v1 controllers still mounted at their old paths. This allows gradual migration but is a source of confusion.

```bash
ls /sys/fs/cgroup/
# cgroup.controllers  cgroup.max.descendants  cpu.stat  io.stat  memory.stat
# cpu  memory  net_cls  net_prio  blkio  ...  ← v1 bind mounts still present
```

Most modern distros have moved fully to v2 and dropped hybrid. When you see unexpected v1 files on what should be a v2 system, check whether it's a hybrid that hasn't been cleaned up.

---

## The interview answer in 60 seconds

> "cgroups v1 had one hierarchy per controller — memory had its own tree, CPU had its own, blkio had its own. A container's resources were tracked in separate, independently-managed subtrees. The container runtime had to keep them in sync, and bugs from that impedance mismatch caused real K8s issues: leaked cgroups, drifted resource accounting, inconsistent OOM eviction.
>
> v2 fixes this at the kernel level with one unified hierarchy. A process lives in exactly one cgroup path, and all controllers — memory, CPU, IO, pids — apply to the same node. No sync burden for the runtime.
>
> v2 also added PSI — Pressure Stall Information. Instead of just 'how much memory are you using,' PSI tells you 'how much time were your tasks stalled waiting for memory.' K8s 1.27+ uses PSI to drive smarter eviction decisions.
>
> On the operational side, modern nodes — Bottlerocket, AL2023, Ubuntu 22.04+, RHEL 9 — default to v2. EKS 1.25+ on those AMIs runs v2. And when your node runs v2, you want `--cgroup-driver=systemd` on both the kubelet and containerd — it lets systemd own the cgroup lifecycle rather than having the kubelet and systemd fight over the same paths."

---

## Self-test drills

### 1. Why did cgroups v2 win, and what does the unified hierarchy actually mean for K8s?

**Reference answer:**
- v1 had per-controller hierarchies; v2 has one tree for all controllers
- A process is in one place; consistency is kernel-enforced rather than runtime-maintained
- K8s benefits: simpler pod cgroup path, no per-controller sync bugs, PSI-driven eviction
- Practical: Bottlerocket / AL2023 default to v2; EKS 1.25+ uses v2 on those AMIs

### 2. What is PSI and why does K8s care about it?

**Reference answer:**
- Pressure Stall Information — measures time tasks spent stalled waiting for a resource (CPU, memory, IO)
- `some` = at least one task stalled; `full` = all tasks stalled
- v1 only had usage counters; PSI adds stall counters — the difference between a fuel gauge and a traffic jam detector
- K8s 1.27+ uses PSI to make proactive eviction decisions before OOM pressure peaks
- Available per cgroup in `memory.pressure`, `cpu.pressure`, `io.pressure` files

### 3. What's the difference between `--cgroup-driver=cgroupfs` and `--cgroup-driver=systemd`?

**Reference answer:**
- `cgroupfs`: kubelet directly creates/writes cgroup directories. Conflicts with systemd's own cgroup management
- `systemd`: kubelet delegates to systemd via D-Bus. systemd owns cgroup lifecycle; no conflicts; cleaner cleanup on pod delete
- v2 strongly prefers `systemd` driver — the K8s docs explicitly recommend it
- Both kubelet AND containerd must use the same driver; mismatch = node broken

### 4. How do you tell which cgroup version a host is running?

**Reference answer:**
- `stat -fc %T /sys/fs/cgroup` → `cgroup2fs` (v2) or `tmpfs` (v1)
- Or: `mount | grep cgroup` — look for `type cgroup2` vs `type cgroup`
- Distro defaults: Ubuntu 22.04+, RHEL 9+, AL2023, Bottlerocket = v2; AL2 = v1
- In a container: `cat /proc/self/cgroup` — v2 shows single `0::/path` line; v1 shows one line per controller

---

## Further reading / watching

- **Kernel docs — cgroup v2**: [kernel.org/doc/html/latest/admin-guide/cgroup-v2.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html) — the reference. Section on PSI is especially good.
- **K8s docs — cgroup v2**: [kubernetes.io/docs/concepts/architecture/cgroups](https://kubernetes.io/docs/concepts/architecture/cgroups/) — the EKS AMI table is kept up to date here.
- **systemd cgroup v2 migration guide**: [systemd.io/CGROUP_DELEGATION](https://systemd.io/CGROUP_DELEGATION/) — how systemd thinks about cgroup ownership.
- **LWN — "Control groups series" by Neil Brown** — series of articles on both v1 and the v2 design decisions. Search LWN for "cgroup v2".

---

## The 4 dimensions (senior framing)

- **Tech**: v2 unified hierarchy enforced by kernel; PSI for stall-based pressure metrics; `memory.pressure`, `cpu.pressure`, `io.pressure` per cgroup; `stat -fc %T /sys/fs/cgroup` to detect version; prefer `--cgroup-driver=systemd` on v2 nodes.
- **People**: when debugging a node issue, the cgroup driver mismatch (kubelet says cgroupfs, containerd says systemd) is an easy misconfiguration that breaks nodes silently. Document the required config in runbooks; add it to node provisioning IaC.
- **CI/CD**: node AMI selection in Terraform/Pulumi affects which cgroup version you get. If you're switching EKS node groups from AL2 to AL2023, add a test that verifies cgroup driver config is consistent end-to-end. Node join failure due to driver mismatch is hard to diagnose under load.
- **Operations**: add `stat -fc %T /sys/fs/cgroup` to your node health check script. On v2 nodes, add PSI monitoring — `memory.pressure some avg60 > 20%` is a useful early warning of memory pressure before OOM evictions start. Alert on it, not just on OOM events.
