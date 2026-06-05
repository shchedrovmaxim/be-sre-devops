# cgroups v1 vs v2 — the simple version (the USB hub analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./cgroups-v1-vs-v2.md) becomes easy.

This doc explains **one idea**:

> **cgroups v2's "unified hierarchy" means every process lives in exactly one place in exactly one tree — instead of v1's mess where each controller (memory, CPU, IO) had its own separate tree.**

That's the whole difference. Everything else (PSI, the kubelet cgroup-driver story) is detail on top.

---

## The USB hub analogy

Imagine you're managing a server room and you need to track three things about each employee's workstation: **CPU time used**, **memory used**, and **disk IO used**.

**v1 way (three separate USB hubs):**

You have three completely separate tracking systems — one board for CPU, one for memory, one for IO. The same workstation can appear under "engineering" on the CPU board, but under "contractors" on the memory board, and not appear at all on the IO board. If you want to know a workstation's total resource picture, you have to query three different boards and reconcile them yourself.

**v2 way (one master registry):**

Every workstation is registered in one place. That one entry has columns for CPU, memory, and IO. It's consistent. You look up one workstation, you see everything. No reconciliation needed.

That's it. v2 is the one-master-registry approach. v1 was the three-separate-boards approach.

---

## Why v1's three boards caused real problems

The v1 design had a concrete pain point: **you could set a memory limit on a group of processes in the memory cgroup, but a completely different set of processes in the CPU cgroup.** The kernel would enforce them independently. You couldn't reason about a "container" as a unit — you had to juggle per-controller subtrees.

Docker and K8s worked around this by mounting one controller per hierarchy and keeping them in sync themselves. It worked, but it was fragile. A process that escaped one controller's subtree might still be tracked by another.

With v2, there's one hierarchy. A container is one cgroup. All controllers apply to the same group. Consistency is enforced by the kernel, not by container runtime hacks.

---

## The 2 confusing concepts, demystified

### 1. "Unified hierarchy" — what it actually means

In v1, you could do:

```
/sys/fs/cgroup/memory/mycontainer/  ← memory controller here
/sys/fs/cgroup/cpu/mycontainer/     ← cpu controller here
/sys/fs/cgroup/blkio/mycontainer/   ← IO controller here
```

Three separate trees. The `mycontainer` directories are not connected at the kernel level.

In v2, there's one tree:

```
/sys/fs/cgroup/mycontainer/         ← all controllers here
```

The files `memory.max`, `cpu.max`, `io.max` all live in the same directory. The process can only be in one place in this tree. The kernel guarantees it.

### 2. Pressure Stall Information (PSI) — the v2 superpower

v2 added something v1 never had: **PSI** — a measure of how much time processes spent waiting for a resource (CPU, memory, IO) rather than actually running.

v1 could tell you "you used X memory" but not "processes were stalling because they couldn't get memory." PSI tells you the second thing. It's the difference between a fuel gauge (how much do you have?) and a traffic report (are you stuck in a jam?).

PSI files look like:

```bash
$ cat /sys/fs/cgroup/mycontainer/memory.pressure
some avg10=0.12 avg60=0.04 avg300=0.01 total=4321567
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

- `some` = at least one process was stalled waiting for memory
- `full` = ALL processes were stalled (the really bad state)
- `avg10/60/300` = moving average over 10s, 60s, 5 min

Kubernetes (1.27+) reads PSI to make smarter eviction decisions. Before PSI, the kubelet had to poll `memory.current` and guess. With PSI, the kernel tells it directly: "this group is under pressure."

---

## How to tell which version your host runs

```bash
stat -fc %T /sys/fs/cgroup
```

- Output `cgroup2fs` → **v2**
- Output `tmpfs` → **v1** (with controllers mounted as sub-filesystems)

Modern distros that default to v2:
- Ubuntu 22.04+
- RHEL 9+ / Rocky 9+
- Bottlerocket (AWS default for EKS)
- Amazon Linux 2023 (AL2023)

EKS 1.25+ on AL2023 or Bottlerocket nodes = v2. If you're on EKS 1.24 with Amazon Linux 2, you're probably on v1.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What's the core difference v1 vs v2? | v1: one hierarchy per controller. v2: one unified hierarchy, all controllers. |
| Why does it matter? | v2 is consistent — a process is in exactly one cgroup. v1 required runtime hacks to keep subtrees in sync. |
| How do I check? | `stat -fc %T /sys/fs/cgroup` — `cgroup2fs` = v2, `tmpfs` = v1 |
| What is PSI? | Pressure Stall Info — how long processes waited (stalled) for a resource. v2-only. |
| Why does K8s care about v2? | Better eviction decisions (PSI), simpler driver code, unified cgroup path for a pod. |
| What's `--cgroup-driver=systemd`? | Tells kubelet to use systemd to create cgroup slices instead of doing it directly. Preferred on v2 nodes. |
| Which EKS nodes use v2? | AL2023 and Bottlerocket (EKS 1.25+). AL2 uses v1. |

---

## Self-test (the killer question)

Out loud:

> **"Why did cgroups v2 win, and what does the unified hierarchy actually mean for K8s?"**

**Reference answer (intuitive version):**

"v1 had one hierarchy per controller — the memory controller had its own tree, CPU had its own tree, IO had its own tree. That meant a container could appear at different places in each tree, and keeping them in sync was the container runtime's problem. Docker and early K8s did this with hacks. v2 fixes it at the kernel level: one tree, all controllers on the same node, a process can only be in one cgroup. For K8s specifically, that means the kubelet can reason about a pod as a single cgroup rather than stitching together per-controller paths.

v2 also added PSI — Pressure Stall Information — which tells you how much time processes spent waiting for resources rather than running. K8s 1.27+ uses PSI to make smarter eviction decisions when nodes get tight.

On the practical side: modern nodes (Bottlerocket, AL2023, Ubuntu 22.04+) default to v2. EKS 1.25+ on those AMIs means you're on v2. The kubelet flag `--cgroup-driver=systemd` is the preferred setup on v2 because systemd understands the unified hierarchy natively — it creates the right slice structure rather than the kubelet trying to manage cgroup paths directly."

---

## Further reading / watching

- **Kernel docs — cgroup v2**: [kernel.org/doc/html/latest/admin-guide/cgroup-v2.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html) — the canonical reference. Grep it, don't read cover to cover.
- **Liz Rice — "Containers from Scratch"** (YouTube, 45 min) — builds a container live with raw syscalls. Cgroup v1 but the concepts are identical.
- **K8s docs — cgroup v2 support**: [kubernetes.io/docs/concepts/architecture/cgroups](https://kubernetes.io/docs/concepts/architecture/cgroups/)

---

## Next: the deep-dive

When the USB hub analogy feels obvious, jump to [`cgroups-v1-vs-v2.md`](./cgroups-v1-vs-v2.md). The deep-dive covers:

- The v1 controller-per-hierarchy design and why it created real bugs
- v2's unified hierarchy in detail — what changed at the kernel level
- PSI in depth (the `some` vs `full` distinction, how K8s uses it)
- The kubelet `--cgroup-driver=systemd` story — why systemd, not cgroupfs
- How to identify v1 vs v2 on any host
- The EKS node AMI landscape (AL2 vs AL2023 vs Bottlerocket)
- 3 self-test drills with reference answers
- The 4 dimensions framing
