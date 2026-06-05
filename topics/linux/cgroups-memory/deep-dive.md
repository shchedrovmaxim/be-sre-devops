# cgroups: memory — what actually happens when a pod gets OOMKilled

> Goal of this doc: by the end, you can answer **"Walk me through what happens at the cgroup level when a pod exceeds its memory limit, and how does the kernel choose what to kill?"** without hesitation.

---

## The bridge from what you know

You've seen this a hundred times:

```bash
$ kubectl describe pod api-7f9...
  Last State:     Terminated
    Reason:       OOMKilled
    Exit Code:    137
```

You know `137 = 128 + 9 = SIGKILL`. You know "OOMKilled" means it ran out of memory. That's the **app-level** view.

But in an interview, "the pod ran out of memory" is not a senior answer. The senior answer goes one layer down: **the kernel killed a process because a cgroup hit its memory limit.** Then it explains *which* cgroup, *which* limit, and *how* the kernel picked the victim.

That's what this doc is. We trace backward: from "exit code 137" → kernel → cgroup → the four files you actually need to know.

---

## The core mental model in 4 sentences

1. **Every container's processes live inside a cgroup.** Kubernetes creates one per pod and one per container, inside the pod's cgroup.
2. **The cgroup has files in `/sys/fs/cgroup/...` that are knobs and gauges.** Write a number to a knob (`memory.max`) to set a limit; read a gauge (`memory.current`) to see usage.
3. **When the cgroup's usage tries to exceed `memory.max`, the kernel invokes the OOM killer scoped to that cgroup.** It picks a process inside that cgroup and sends it `SIGKILL`.
4. **`SIGKILL` is exit signal 9, and the process exits with code 137.** Kubelet sees the exit code, reports `OOMKilled`, and (depending on `restartPolicy`) restarts the container.

That's the whole pipeline. Now the details.

---

## cgroups are a Linux primitive — Kubernetes just uses them

Important framing: **cgroups have nothing to do with Kubernetes.** They're a Linux kernel feature (since 2007), and every modern Linux system uses them constantly — with or without K8s, with or without containers.

Examples of cgroup users you already touch every day:

| User | What it uses cgroups for |
|---|---|
| **systemd** | Every service unit on every modern Linux distro lives in its own cgroup at `/sys/fs/cgroup/system.slice/<unit>.service/`. Try `systemctl status sshd` — that "CGroup:" line is real. |
| **Docker standalone** (no K8s) | Same `memory.max`, same OOM kill mechanics. Docker writes to the cgroup files; the kernel does the rest. |
| **Podman, LXC, Nomad, runc directly** | All container runtimes use cgroups. The container spec (OCI) literally has a "cgroups" section. |
| **Plain `systemd-run`** | `systemd-run --scope -p MemoryMax=100M python script.py` — boom, your script is now in a 100 MiB-limited cgroup. No container, no K8s, no Docker. |
| **`cgcreate` / `cgexec`** | Old-school CLI from `libcgroup-tools`. Make a cgroup, put a PID in it, set a limit. The original interface. |

Try this on any Linux box right now (no Docker required):

```bash
systemd-run --user --scope -p MemoryMax=50M bash
# You now have a shell whose entire memory is capped at 50 MiB.
# Inside it, run: cat /proc/self/cgroup    # see the cgroup path
# Then try: python -c "x=' '*200_000_000"  # → killed
```

**Why this matters for interviews**: if someone asks "how do limits work in Kubernetes?", the senior answer starts at the kernel, not at the YAML. The same sentences explain Docker, systemd, ECS, Nomad — *anything* on Linux. K8s is just one more thing writing values into `memory.max` on your behalf.

So everything in this doc — `memory.max`, the OOM killer sequence, `memory.events`, `oom_score_adj` — applies anywhere Linux runs processes. The only thing Kubernetes-specific is **which knobs kubelet decides to set, and the QoS-class policy that drives them.**

---

## The 4 files you need to know

We're talking **cgroup v2** — the modern unified hierarchy. v1 differences are in a section near the bottom. On any modern node, the path looks like:

```
/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<UID>.slice/cri-containerd-<CID>.scope/
```

Inside that directory:

### 1. `memory.max` — the hard ceiling

The number of bytes this cgroup is **allowed** to use. If usage tries to exceed it, the kernel either:
- reclaims pages aggressively (drops file cache, swaps if available), or
- invokes the OOM killer scoped to this cgroup.

**This is what K8s `resources.limits.memory` maps to.** Set `limits.memory: 512Mi` on a container → kubelet writes `536870912` into that container's cgroup's `memory.max`.

```bash
$ cat memory.max
536870912
```

If you set no limit, it reads `max` (the literal string) — meaning unbounded.

### 2. `memory.high` — the soft throttle

When usage crosses `memory.high`, the kernel **throttles** the cgroup: page allocations get slowed down so the cgroup feels backpressure and (hopefully) frees memory before it hits `memory.max`. The cgroup is *not* killed — it just becomes slow.

**Kubernetes does not use `memory.high` by default.** This is interview gold. The reason: K8s wants deterministic, predictable behavior — kill on limit, don't silently degrade performance. `memory.high` makes pods feel slow without a clear failure signal, which is a worse outcome for an orchestrator than a clean kill + restart.

```bash
$ cat memory.high
max
```

(There's a kubelet feature gate, `MemoryQoS`, that *does* use `memory.high`. It's beta and off by default in most managed clusters. Mention it if asked — but the senior answer is "K8s prefers hard limits for predictability.")

### 3. `memory.current` — the live gauge

How many bytes this cgroup is currently using. Reading this is how you debug "is the pod about to OOM?" without waiting for the kill.

```bash
$ cat memory.current
489234432
```

This is what Prometheus's `container_memory_working_set_bytes` ultimately comes from (via cAdvisor reading these files).

### 4. `memory.events` — the audit trail

This is the file that **proves** an OOM happened. It's a counter of events:

```bash
$ cat memory.events
low 0
high 0
max 142
oom 3
oom_kill 3
```

- `max` — number of times allocation hit `memory.max` and was blocked or reclaimed
- `oom` — number of times the cgroup triggered an OOM event
- `oom_kill` — number of processes the cgroup OOM killer actually killed

**This file is your forensic evidence.** When a pod is reported OOMKilled, this is the file that proves it was a *cgroup-level* OOM (not a system-wide one). Senior debugging move: `kubectl exec` into a sibling pod, find the dying pod's cgroup path, `cat memory.events` repeatedly while traffic flows. Watch `oom_kill` increment in real time.

---

## What actually happens when the limit is hit — the kernel sequence

Suppose the cgroup is at 511 MiB / 512 MiB and your app calls `malloc()` for another 2 MiB. Here is the sequence:

1. **Allocation request** — the app calls `malloc()` → libc → kernel via `brk()` or `mmap()`.
2. **Page fault** — the kernel needs to back the virtual memory with physical pages.
3. **Charge to cgroup** — before handing out the page, the kernel "charges" it against the cgroup's memory counter.
4. **Reclaim attempt** — if charging would cross `memory.max`, the kernel first tries to *reclaim* memory inside this cgroup: drop file cache, evict anonymous pages to swap (if swap is enabled — usually disabled on K8s nodes).
5. **OOM kill (the cgroup OOM killer)** — if reclaim doesn't free enough, the kernel walks every process inside the cgroup, computes an OOM score per process, picks the highest scorer, sends it `SIGKILL`.
6. **Process dies with exit code 137** — `SIGKILL` is signal 9 → exit code `128 + 9 = 137`.
7. **Kernel logs to `dmesg`** — you'll see lines like:
   ```
   [12345.678] Memory cgroup out of memory: Killed process 4321 (python) total-vm:524288kB, anon-rss:498000kB ...
   ```
8. **Kubelet observes the exit code** — reports `Reason: OOMKilled, ExitCode: 137`, increments `restartCount`, decides whether to restart based on `restartPolicy`.

**Senior-level nuance**: the OOM killer is *scoped to the cgroup*. It does not look at processes in other cgroups. This means **one pod hitting its limit does not affect other pods.** That isolation *is* the point of cgroups. (Contrast with a system OOM — covered next.)

---

## Cgroup OOM vs system OOM — the distinction

There are **two** OOM killers and they have different scopes:

| | Cgroup OOM | System OOM |
|---|---|---|
| **Trigger** | A cgroup hits `memory.max` | The whole node runs out of memory |
| **Scope of victim** | Only processes in that cgroup | Any process on the system |
| **Reported as** | Pod `OOMKilled` | Node-level chaos: random pods die, kubelet may evict, NotReady |
| **`memory.events` updated?** | Yes, on the offending cgroup | No (it's not a cgroup event) |
| **Healthy?** | This is normal, expected behavior | This is a bug — your `kube-reserved` / `system-reserved` is wrong |

**Why this distinction matters for interviews**: if someone tells you "a random pod died at 3 AM and it wasn't even high on memory," you don't say "OOMKilled." You say: **"Sounds like a system-level OOM, not a cgroup OOM. Let me check `dmesg` on the node and the kubelet's eviction logs — likely a reservation misconfiguration."**

---

## How K8s decides who dies first under node pressure

Cgroup OOM is one mechanism. **Eviction** is the other — kubelet kills pods *before* the node hits a system OOM, based on QoS class.

### QoS classes (review)

| Class | Definition | Eviction order |
|---|---|---|
| **Guaranteed** | `requests == limits` for both CPU and memory | Last to be evicted |
| **Burstable** | At least one container has a request *or* limit set, but not requests==limits | Middle |
| **BestEffort** | No requests, no limits anywhere | First to be evicted |

### Under node memory pressure, kubelet evicts in this order:

1. **BestEffort pods** — sorted by memory usage (highest first)
2. **Burstable pods exceeding their request** — sorted by "how much over request" (most over first)
3. **Burstable pods under their request** — same logic
4. **Guaranteed pods** — only if absolutely nothing else can be evicted

**`oom_score_adj`** is the kernel-level expression of this hierarchy. Kubelet sets it per pod when launching:

| QoS | `oom_score_adj` |
|---|---|
| Guaranteed | -997 (very low — kernel avoids killing) |
| BestEffort | 1000 (max — kernel kills first) |
| Burstable | computed: `1000 - (1000 * memory_request / node_memory_capacity)` |

`oom_score_adj` is added to a process's intrinsic OOM score. The kernel kills the highest score. So a BestEffort pod scores ~1000, a Guaranteed pod scores ~-997, and Burstable falls in between proportional to how big its request is relative to the node.

**Translation**: bigger request → lower oom_score_adj → kernel less likely to kill. Asking for more memory in your `requests` is literally a kernel-level insurance policy.

---

## v1 vs v2 — the one paragraph you need

**v1** (legacy): each controller (memory, cpu, blkio, ...) had its own separate hierarchy. A process could be in different cgroups for different resources. Resource accounting was inconsistent, and the memory controller had quirks (`memory.usage_in_bytes`, separate `memory.kmem` accounting, etc).

**v2** (unified): one hierarchy. A process is in one cgroup, and that cgroup has all the controllers attached. Cleaner files (`memory.max` vs v1's `memory.limit_in_bytes`), better accounting (no separate kmem), pressure stall information (PSI) baked in. Every modern distro (RHEL 9, Ubuntu 22.04+, Bottlerocket, Amazon Linux 2023) defaults to v2. EKS 1.25+ on AL2023 / Bottlerocket is v2. **You can safely assume v2 in interviews unless explicitly asked about legacy systems.**

To check on a host:

```bash
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs   # → v2
tmpfs       # → v1 (legacy)
```

---

## Hands-on — see it happen on Docker Desktop

Docker Desktop on macOS runs a tiny Linux VM. We'll exec into it and watch a cgroup OOM happen live.

### Step 1 — start an idle container with a memory limit

We deliberately do **not** allocate yet. Start it idle so the cgroup exists, we can instrument it, and only *then* trigger the OOM. (Earlier draft of this doc made the container start allocating immediately; it died and got removed before you could even read the cgroup files. Don't do that.)

```bash
docker run --rm -d --name oomtest --memory=100m \
  python:3.12-slim \
  sleep 600
```

The container will sit idle for 10 minutes. Plenty of time to set up.

### Step 2 — get a shell inside Docker Desktop's VM

macOS doesn't expose `/sys/fs/cgroup` directly — the cgroups exist inside Docker's Linux VM. The classic trick:

```bash
# Open a shell in the Docker VM
docker run -it --rm --privileged --pid=host justincormack/nsenter1
```

You're now root on the VM. (If you're on a Linux host directly — EC2, Multipass — you skip this step; just `ssh` in.)

### Step 3 — find the cgroup path (the namespace trap)

You might be tempted to ask the container itself:

```bash
# On macOS:
docker exec oomtest cat /proc/self/cgroup
# → 0::/
```

**That `/` is a trap.** It's not the real path. The container lives in a **cgroup namespace** (a Linux namespace added in kernel 4.6, on by default in modern Docker), which re-roots the cgroup tree so the container sees its own cgroup as `/`. Same isolation principle as the PID namespace — the container is lied to so it can't see the host's full structure.

To get the real path you need two pieces: the container's **host-side PID** (which only `docker inspect` on macOS knows), and the cgroup file (which only the VM has). So you'll use two terminals:

```bash
# Terminal A — on macOS — get the host PID:
PID=$(docker inspect -f '{{.State.Pid}}' oomtest)
echo $PID
# → 4321  (some number — copy it for terminal B)
```

```bash
# Terminal B — inside the VM (via nsenter1):
PID=4321   # paste the number from terminal A
cat /proc/$PID/cgroup
# → 0::/system.slice/docker-<full-id>.scope
```

`/proc/<PID>/cgroup` read from the **host's namespace** shows the real path because the host isn't inside the cgroup namespace. Now save the path for the next steps:

```bash
# Still in terminal B:
CGROUP=/sys/fs/cgroup$(cat /proc/$PID/cgroup | cut -d: -f3)
echo $CGROUP
ls $CGROUP
cat $CGROUP/memory.max
# → 104857600  (= 100 MiB)
```

**Interview-grade follow-up**: if asked *"why does `cat /proc/self/cgroup` inside a container show `/`?"* — that's the cgroup namespace at work. Tie it back to the broader principle: containers are namespaces + cgroups + caps; namespaces hide the host, cgroups constrain resources, and `/proc/self/cgroup` is one place where the hiding shows up.

### Step 4 — set up the watches BEFORE triggering OOM

You want to be watching when the kill happens. Open **two more** VM shells (each is just another `docker run -it --rm --privileged --pid=host justincormack/nsenter1` from macOS) so you have a setup like:

| Terminal | Where | What it's doing |
|---|---|---|
| A | macOS | the `docker` CLI (will trigger the OOM in Step 5) |
| B | VM | already used in Step 3; can stay open as scratch |
| C | VM | `watch` on `memory.current` |
| D | VM | `watch` on `memory.events` |

In **terminal C**:

```bash
CGROUP=/sys/fs/cgroup/system.slice/docker-<full-id>.scope   # paste the path from Step 3
watch -n 0.5 "cat $CGROUP/memory.current"
```

Should show a small steady number (a few MB — sleeping Python).

In **terminal D**:

```bash
CGROUP=/sys/fs/cgroup/system.slice/docker-<full-id>.scope   # same path
watch -n 0.5 "cat $CGROUP/memory.events"
```

Should show all zeros: `low 0  high 0  max 0  oom 0  oom_kill 0`.

### Step 5 — trigger the OOM from macOS

In **terminal A**:

```bash
docker exec oomtest python -c "
import time
data = []
while True:
    data.append(b'x' * 1024 * 1024)
    time.sleep(0.5)
"
```

**Watch terminals C and D live.** Terminal C's `memory.current` will climb from ~5 MB up toward 100 MB. The moment it tries to cross, terminal D will flip `oom_kill 0 → 1` (and `oom 0 → 1`, `max 0 → 1`). The `docker exec` command in terminal A will return with a non-zero exit code.

This is the moment. You watched it happen.

### Step 6 — confirm in `dmesg` and Docker

```bash
# In any VM terminal:
dmesg | tail -20 | grep -i "out of memory"
# → Memory cgroup out of memory: Killed process <PID> (python) ...

# Back on macOS (terminal A):
docker inspect oomtest --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}'
# Note: since we ran `sleep 600` as PID 1 and the python OOM'd as a child of `docker exec`,
# the container itself may still be running. The OOM killed the python process, not PID 1.
# That's actually a useful insight: cgroup OOM kills *a* process in the cgroup, not the whole container.
```

**You just traced the entire pipeline.** This is the answer in your hand.

**Bonus observation**: because we triggered the allocation via `docker exec` (which runs as a *child* process inside the container, not as PID 1), the OOM killer picked the python child as its victim — not the original `sleep 600` PID 1. The container kept running. In real K8s the app is PID 1, so killing it means the container exits and kubelet reports `OOMKilled`. Same kernel mechanism, different observed outcome based on process tree shape.

### Cleanup

```bash
docker rm -f oomtest
```

---

## The interview answer in 60 seconds

> "When a pod exceeds its memory limit, it's actually the **cgroup** hitting `memory.max`. Kubelet writes the pod's `limits.memory` into the container's cgroup's `memory.max` file under `/sys/fs/cgroup/`. The kernel charges every allocation against that cgroup. When an allocation would cross the limit, the kernel first tries to reclaim — drop file cache, evict pages. If that fails, it invokes the **cgroup-scoped OOM killer**: it walks processes inside that cgroup, picks the highest OOM score, and sends `SIGKILL`. The process exits 137. Kubelet observes that, reports `OOMKilled`, and `memory.events` increments `oom_kill`. The kernel's choice of victim is influenced by `oom_score_adj`, which kubelet sets per pod based on QoS class — Guaranteed gets -997, BestEffort gets 1000, Burstable is computed from the memory request. So your request is literally a kernel-level insurance policy. This is **cgroup OOM** — isolated to the offending pod. If you see random pods dying without `memory.events` showing it, you're looking at a **system OOM**, which means your node reservations are wrong."

---

## Self-test

Try each out loud as if you're the candidate in an interview.

### 1. Walk me through what happens at the cgroup level when a pod exceeds its memory limit.

**Reference answer:**
- App calls `malloc` → kernel allocates pages → charges them to the container's cgroup memory counter.
- If charge would exceed `memory.max`, kernel attempts reclaim (file cache eviction, anon swap-out if swap on — usually off in K8s).
- If reclaim insufficient, kernel invokes cgroup OOM killer: walks processes in that cgroup, ranks by OOM score (intrinsic score + `oom_score_adj`), `SIGKILL`s the highest.
- Process exits 137. `memory.events` increments `oom`/`oom_kill`. `dmesg` logs "Memory cgroup out of memory."
- Kubelet observes exit code, reports `OOMKilled`, restarts per `restartPolicy`.
- Bonus mark: mention this is *isolated* to the cgroup; other pods unaffected.

### 2. Why does Kubernetes set `memory.max` but not `memory.high`?

**Reference answer:**
K8s wants **deterministic, observable failures**. `memory.high` throttles silently — the pod just gets slow, no clear failure signal, hard to alert on. `memory.max` is a hard kill — clean signal (`OOMKilled`), restart, alert. For an orchestrator that needs to make scheduling decisions, "this pod failed at this time" is far more useful than "this pod is mysteriously slow." There's a beta feature gate `MemoryQoS` that does set `memory.high`, but it's off by default.

### 3. Two Burstable pods on the same node, node hits memory pressure — who gets evicted first and why?

**Reference answer:**
Kubelet evicts based on **usage relative to memory request**, not absolute usage. The pod that's most over its request gets evicted first. Mechanism: `oom_score_adj` is computed as `1000 - (1000 * memory_request / node_capacity)`, so a pod that requested *less* has a higher `oom_score_adj` and is more attractive to the OOM killer. Translation: if you want survival under pressure, **request what you actually need.** A pod that requested 200 MiB but is using 800 MiB is far more evictable than one that requested 1 GiB and is using 600 MiB.

---

## The 4 dimensions (senior-level framing)

When asked about memory limits in a design interview, address all four:

- **Tech**: cgroup `memory.max` is hard; pick QoS class deliberately; understand `oom_score_adj`; know v2 is the modern default.
- **People**: developers often set `limits` without `requests` (or vice versa) — educate, or set defaults at the namespace level (`LimitRange`). Make OOM signals visible in dashboards so devs see their own pods dying.
- **CI/CD**: enforce that every container has both `requests` and `limits` via Kyverno / OPA Gatekeeper policy. Block PRs that ship pods with no memory request. Test memory limits in load tests *before* prod.
- **Operations**: alert on `OOMKilled` rate, not just node memory. Surface `memory.events` deltas. Capacity planning: track p99 memory usage per workload to set right-sized requests. Runbook: "pod OOMKilled" → check `memory.events`, recent code changes, GC settings (Java/Node), trend over time before reflexively bumping the limit.
