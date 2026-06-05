# cgroups: CPU — why your pod can be at 30% CPU and still feel slow

> Goal of this doc: by the end, you can answer the trap interview question **"A pod averages 30% CPU and feels slow, but it's well under its 1-CPU limit. Walk me through what could be causing this."** That answer is **CFS quota throttling in sub-second windows** — invisible on most dashboards, and the most common "mystery latency" cause in K8s.

> Read [`cgroups-memory.md`](../cgroups-memory/deep-dive.md) first if you haven't. The kernel/cgroup framing carries over; here we just swap memory for CPU.

---

## The bridge from what you know

You write Kubernetes YAML like this every day:

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 1
```

You probably mentally translate this to "the pod is *guaranteed* 0.5 CPU and *capped* at 1 CPU." That's roughly true at the app level. But these two numbers go to **completely different kernel mechanisms**:

| YAML | cgroup file | Kernel mechanism | When it matters |
|---|---|---|---|
| `requests.cpu` | `cpu.weight` | Scheduler priority (relative share) | **Only under contention** — if the node has spare CPU, requests do nothing |
| `limits.cpu` | `cpu.max` | CFS quota + period (hard cap) | **Always** — kernel slices time, blocks the cgroup when quota runs out |

That mismatch is where the surprises live. The senior answer starts here: requests and limits are not "min and max" of the same thing. They're two different kernel knobs, and the *limit* one is the one that bites.

---

## The core mental model in 4 sentences

1. **Every container's processes live in a cgroup with two CPU knobs**: `cpu.weight` (priority) and `cpu.max` (hard cap).
2. **`cpu.max` is enforced by the Linux CFS scheduler in fixed time windows** — default 100 ms. If the cgroup uses all its allowed CPU-time in that window, it's **throttled** (frozen) until the window ends.
3. **Throttling happens at the period boundary**, so a bursty app can use all its quota in 10 ms and sit frozen for 90 ms — even if its *average* CPU usage is well below the limit.
4. **`cpu.stat` is your forensic file** — `nr_throttled` and `throttled_usec` tell you exactly how often and for how long the cgroup was frozen. This is the file no one looks at and it's where the mystery latency hides.

Memory hits a limit and dies. **CPU hits a limit and gets slow.** Different failure mode, same primitive.

---

## Same primitive, different knob

Everything from [`cgroups-memory.md`](../cgroups-memory/deep-dive.md) about "cgroups are a Linux primitive, not K8s" applies identically here:

- systemd: `systemd-run --user --scope -p CPUQuota=50% bash` caps the shell's CPU at 50% of one core. No container, no K8s.
- Docker standalone: `docker run --cpus=0.5` writes to `cpu.max`. Same kernel, same throttling.
- Podman, Nomad, runc — all the same.

The kernel doesn't know what Kubernetes is. It knows cgroups and CFS.

---

## The 3 files you need to know

Path is the same as for memory — under the container's cgroup directory in `/sys/fs/cgroup/...`.

### 1. `cpu.max` — the hard cap

Two numbers: `<quota> <period>`, both in microseconds.

```bash
$ cat cpu.max
50000 100000
```

Translation: "you can use 50,000 µs of CPU time every 100,000 µs window." That's **50 ms of CPU per 100 ms wall-clock** → **0.5 CPU**.

Default period in K8s: **100,000 µs (100 ms)**. Quota = `limits.cpu × period`.

| `limits.cpu` | `cpu.max` |
|---|---|
| `500m` | `50000 100000` |
| `1` | `100000 100000` |
| `2` | `200000 100000` |
| (none) | `max 100000` (no cap) |

**Critical**: a cgroup with `cpu.max = "200000 100000"` (= 2 CPUs) can use **up to 2 cores in parallel** within a period — multiple threads/processes share the quota. If 4 threads run flat out for 50 ms, they collectively burn 200 ms of CPU time → quota exhausted at the 50 ms mark → all 4 throttled for the next 50 ms. **Quota is CPU-time, not wall-clock-time.**

### 2. `cpu.weight` — the relative priority

```bash
$ cat cpu.weight
50
```

Range: 1–10,000. Default: 100. **Only matters when CPUs are oversubscribed.** If the node has idle capacity, weight does nothing. If two cgroups want the same CPU, the kernel splits proportionally to their weights.

K8s maps `requests.cpu` → `cpu.weight` via a (slightly ugly) formula. The intuition: bigger request → higher weight → more CPU when contended. **Concrete:** if pod A has `requests: 100m` (weight ~4) and pod B has `requests: 900m` (weight ~37), under contention pod B gets ~9× the CPU share.

The (cgroup v1) old name was `cpu.shares`. Same idea, different range (2–262144, default 1024).

### 3. `cpu.stat` — the forensic gauge

This is the file that proves throttling happened.

```bash
$ cat cpu.stat
usage_usec 1234567890
user_usec 1100000000
system_usec 134567890
nr_periods 12345
nr_throttled 678
throttled_usec 67800000
```

The three lines that matter for debugging:

- **`nr_periods`** — how many 100-ms windows have elapsed since the cgroup was created
- **`nr_throttled`** — how many of those windows the cgroup hit its quota and got frozen
- **`throttled_usec`** — total time the cgroup spent frozen, in microseconds

The ratio that matters: **`nr_throttled / nr_periods`**. If 678 / 12345 = 5.5%, **the cgroup was frozen 5.5% of the time** — even if average CPU usage looked fine on the dashboard.

**Prometheus exports this** as `container_cpu_cfs_throttled_periods_total` / `container_cpu_cfs_periods_total`. If you do nothing else after reading this doc, **add a panel for that ratio to your service dashboards.** Most teams don't, and that's why "mystery latency" stays mysterious.

---

## CFS quota+period — the heart of the topic

This is the mechanism that traps every senior candidate who hasn't drilled it.

### The kernel sequence in 6 steps

1. **CFS enforces quota in fixed wall-clock windows** of `cpu.max` period (default 100 ms).
2. **At the start of each period**, the cgroup's "remaining quota" resets to the full quota.
3. **When a process in the cgroup runs**, the kernel decrements remaining quota by the CPU-time used.
4. **When remaining quota hits zero**, the kernel marks the cgroup as throttled — all its processes are taken off the CPU.
5. **Throttled processes wait until the next period boundary** (up to 100 ms — feels like forever to a service handling HTTP).
6. **At the boundary**, the quota refills, throttled processes become runnable again, and `nr_throttled` increments by 1.

### Why "30% CPU and feels slow" happens

Suppose your service:
- Has `limits.cpu: 1` → quota 100 ms, period 100 ms
- Receives a request every 200 ms
- Each request takes 30 ms of CPU on a single thread, spread evenly

Average CPU = 30 ms used / 200 ms wall = **15%**. Plenty of headroom. No throttling.

Now make it multi-threaded — same workload, but 4 worker threads each doing 30 ms of CPU in parallel when a request arrives:
- Wall time to serve one request: ~7.5 ms (4 threads × 30 ms = 120 ms CPU time → exceeds the 100 ms quota at the ~25 ms mark of the period → throttled for the rest of the period (~75 ms))
- Observed p50 latency: ~8 ms. Observed p99: ~83 ms (when a request hit just before throttling kicked in).
- Average CPU on a 60-second-resolution dashboard: still ~15%.

The pod is at 15% average CPU and its p99 latency is 10× p50, on every burst. **That is throttling.** Your dashboards don't see it because they sample every 30 or 60 seconds and throttling lives in 10-ms windows.

### The runtime trap — Go GOMAXPROCS / Java threads / Node workers

This is the deep cut.

Go runtime: at startup, `runtime.GOMAXPROCS(0)` is set to `runtime.NumCPU()` — **the number of host CPUs, not the cgroup quota.** On a 16-core node with `limits.cpu: 1`, Go spawns 16 worker threads and tries to run goroutines on all of them. They collectively burn through 100 ms of CPU in ~6 ms of wall time, then everything stalls for 94 ms.

Fix: use `go.uber.org/automaxprocs` (import for side-effect; reads `cpu.max` and sets `GOMAXPROCS` to match) or set `GOMAXPROCS` env var manually.

Java (before JDK 10): same problem with `Runtime.availableProcessors()`. Many thread pools size themselves to this. Fix: JDK 10+ is container-aware; older versions need `-XX:ActiveProcessorCount=N` manually.

Node.js: less affected (single event loop), but worker threads and the libuv thread pool (`UV_THREADPOOL_SIZE`) follow the same trap.

**Interview gold**: if asked "you've put your app in a container with `limits.cpu: 1` and it's getting throttled at 20% CPU — what would you check?", the answer is: **GOMAXPROCS / thread pool sizing.** The app is parallelizing across host cores but living in a fractional-CPU sandbox.

---

## Throttling vs starvation — the distinction

Like cgroup-OOM vs system-OOM in the memory doc, CPU has two failure modes:

| | Throttling | Starvation |
|---|---|---|
| **What** | Cgroup hit its own `cpu.max` quota | Cgroup wants CPU but the scheduler keeps picking other cgroups |
| **Cause** | Your `limits.cpu` is too low for your workload's burstiness | Node is oversubscribed; `cpu.weight` (requests) is too low relative to neighbors |
| **Visible in** | `cpu.stat: nr_throttled` increments | Lower-than-expected CPU usage despite high load; high run-queue latency |
| **Fix** | Raise the limit (or drop the limit — see next section) | Raise `requests.cpu`; spread workload across more nodes |
| **Healthy?** | Some throttling is fine for batch; bad for latency-sensitive | Generally bad — means the cluster is over-packed |

If you only know one of the two, you'll misdiagnose half of "this pod is slow" tickets.

---

## The "CPU limits are an antipattern" debate (senior framing)

You will be asked about this. The interviewer will have an opinion. Be ready to discuss both sides.

### Why some people say "stop setting CPU limits"

- **CFS throttling penalizes latency** (the 30%-and-slow problem above).
- Modern nodes have plenty of CPU; the goal isn't to "save" CPU, it's to serve fast.
- `requests` already does fair-share allocation under contention — that's enough isolation for most workloads.
- Removing limits often **improves p99 latency dramatically** without making neighbors worse, because the kernel's fair scheduler still arbitrates.
- Tim Hockin (K8s co-founder) has tweeted this position multiple times.

### Why others still set CPU limits

- **Multi-tenant clusters** where you can't trust neighbors — a runaway tenant should not eat the whole node.
- **Predictable, reproducible perf testing** — without limits, dev/staging perf depends on whatever else is running.
- **Cost control** — limits prevent a bug from billing you for 32 cores.
- **Compliance / contractual** SLAs that reference resource ceilings.

### The senior answer

> "It depends on the workload. For latency-sensitive services on trusted multi-tenant clusters, I'd set requests carefully and leave limits off, then monitor `nr_throttled` and per-pod CPU. For batch jobs, untrusted workloads, or environments where I need reproducible perf, I'd set limits. The thing I'd *always* do is monitor `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` per service — that ratio tells you whether your decision is correct."

That answer shows you've thought about it. It doesn't pick a side dogmatically — which is exactly the senior signal.

---

## v1 vs v2 — the one paragraph

**v1**: `cpu.cfs_quota_us` and `cpu.cfs_period_us` as separate files. `cpu.shares` for weight (range 2–262144, default 1024). `cpu.stat` exists but with different fields. **v2**: single `cpu.max` file with both quota and period on one line. `cpu.weight` (range 1–10,000, default 100). Cleaner `cpu.stat`. Same scheduler underneath (CFS in both, except v2 also exposes the newer EEVDF scheduler internals when the kernel uses it on 6.6+). **Assume v2 in interviews** unless explicitly asked.

```bash
# v2 cgroup CPU files
ls /sys/fs/cgroup/system.slice/sshd.service/
# cpu.max  cpu.weight  cpu.stat  cpu.pressure  ...
```

---

## Hands-on — watch throttling happen on Docker Desktop

The flow is identical to the memory doc — start idle, instrument, then trigger. If you did the memory hands-on, this will feel familiar.

### Step 1 — start an idle container with a CPU limit

```bash
# On macOS:
docker run --rm -d --name cputest --cpus=0.5 \
  python:3.12-slim \
  sleep 600
```

The container is capped at 0.5 CPU and just sleeps. The cgroup exists; we can instrument before triggering.

### Step 2 — open a VM shell

```bash
# On macOS, terminal B:
docker run -it --rm --privileged --pid=host justincormack/nsenter1
```

You're now in the Linux VM.

### Step 3 — find the cgroup path

```bash
# On macOS, terminal A:
docker inspect -f '{{.State.Pid}}' cputest
# → e.g. 8842
```

```bash
# In VM, terminal B:
PID=8842   # paste from terminal A
CGROUP=/sys/fs/cgroup$(cat /proc/$PID/cgroup | cut -d: -f3)
echo $CGROUP
cat $CGROUP/cpu.max
# → 50000 100000        ← 50 ms quota per 100 ms period = 0.5 CPU
cat $CGROUP/cpu.weight
# → 100                 ← default; we didn't set requests
cat $CGROUP/cpu.stat
# → ... nr_throttled 0  throttled_usec 0
```

### Step 4 — set up the watch BEFORE triggering CPU burn

Open a third VM shell (terminal C, same nsenter1 command). Set `$CGROUP` again with the same path. Then:

```bash
watch -n 0.5 "cat $CGROUP/cpu.stat | grep -E 'nr_periods|nr_throttled|throttled_usec'"
```

Should show all increasing-but-stable for `nr_periods`, and zero for the throttle ones.

### Step 5 — trigger CPU burn from macOS

```bash
# On macOS, terminal A:
docker exec cputest python -c "
import time
end = time.time() + 30
while time.time() < end:
    pass   # busy loop, eats 100% of one CPU
"
```

**Watch terminal C live.** Within a second or two:

```
nr_periods 145
nr_throttled 73        ← climbing
throttled_usec 3600000 ← climbing
```

The python busy-loop wants 100% of one core, but the cgroup only allows 50%. So roughly **every other period, it's throttled**. `nr_throttled / nr_periods ≈ 0.5`. That's the kernel telling you: "this process spent half its time frozen."

### Step 6 — confirm at the app level

```bash
docker stats cputest --no-stream
# CONTAINER   CPU %    ...
# cputest     50.0%    ...
```

50% CPU. That's the throttling working. Without `--cpus=0.5`, the same Python would show 100%.

### Cleanup

```bash
docker rm -f cputest
```

---

## The interview answer in 60 seconds

> "When you set `limits.cpu: 1` on a pod, kubelet writes `100000 100000` into the cgroup's `cpu.max` — that's 100 ms of quota per 100-ms period. The Linux **CFS scheduler** enforces this in fixed wall-clock windows. If the cgroup's processes collectively burn 100 ms of CPU time in the first 30 ms of a window, **the kernel throttles them for the remaining 70 ms**. They're frozen, waiting for the next period boundary. The pod's *average* CPU might be 30% but **p99 latency spikes hard** because of those 70-ms freezes. The forensic file is `cpu.stat` — `nr_throttled` over `nr_periods` tells you the throttle ratio. Prometheus exports this as `container_cpu_cfs_throttled_periods_total`. A really common cause of throttling at low average CPU is the **GOMAXPROCS trap**: Go (or Java pre-10, or Node workers) sees the host's CPU count, not the cgroup's, and spawns too many threads — they all run at once, burn the quota in milliseconds, and get throttled. Fix is `automaxprocs` or set GOMAXPROCS by hand. Meanwhile `requests.cpu` maps to `cpu.weight`, which is only relevant under contention — it determines fair-share, not a guarantee. There's a real debate about whether to set CPU limits at all on latency-sensitive workloads, because of exactly this throttling penalty."

---

## Self-test

Try each out loud.

### 1. A pod averages 30% CPU and feels slow, but is well under its 1-CPU limit. Walk me through what could be causing this.

**Reference answer:**
- The pod is being **throttled by the CFS quota** in sub-second windows. Average CPU on a 60s dashboard hides 100-ms throttle events.
- Mechanism: `cpu.max` is enforced over a 100-ms period. If multi-threaded code burns the full quota in <100 ms wall time, the cgroup is frozen until the next period.
- Confirm by reading `cpu.stat`: `nr_throttled / nr_periods` ratio. Or query Prometheus: `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])`.
- Likely root cause: **GOMAXPROCS** (or Java thread pool, or Node `UV_THREADPOOL_SIZE`) is sized to host cores, not the cgroup's quota. The runtime is parallelizing harder than the limit allows.
- Fix: `automaxprocs` library (Go), `-XX:ActiveProcessorCount` (old Java), or drop the CPU limit entirely if the workload is latency-sensitive on a trusted node.

### 2. Why are CPU limits considered controversial by some K8s practitioners?

**Reference answer:**
- CFS quota throttling penalizes p99 latency disproportionately — bursty work hits the quota, gets frozen for the rest of the period.
- `requests.cpu` already gives fair-share allocation under contention via `cpu.weight` — that provides enough isolation for most workloads.
- Modern nodes typically have spare CPU; the goal is fast response, not "saving" cores.
- Removing limits often improves p99 dramatically without harming neighbors.
- Counter-arguments: untrusted multi-tenancy, reproducible perf testing, cost control, regulated SLAs.
- Senior take: it depends; what's mandatory is **monitoring the throttle ratio** regardless of which side you pick.

### 3. What's the difference between throttling and starvation?

**Reference answer:**
- **Throttling**: cgroup hit *its own* `cpu.max` quota. Self-imposed limit. Fix by raising/removing the limit or shrinking thread count.
- **Starvation**: cgroup *wants* CPU but the scheduler keeps picking other cgroups. External pressure. Fix by raising `requests.cpu` (= `cpu.weight`), spreading to more nodes, or fixing whichever neighbor is greedy.
- Both manifest as "low CPU usage but slow," but the forensic files differ: throttling shows up in `cpu.stat: nr_throttled`; starvation shows up in `cpu.pressure` (PSI) as high `some` / `full` averages.
- Sloppy version: "throttling = I'm hitting my limit; starvation = I'm being out-prioritized."

---

## The 4 dimensions (senior-level framing)

- **Tech**: set `requests` carefully; question `limits` for latency-sensitive paths; understand `cpu.max` period (100 ms default); know `cpu.stat` and the throttle ratio; fix runtime traps (`automaxprocs`, JDK 10+, `UV_THREADPOOL_SIZE`).
- **People**: developers default to "set limits because the docs say to." Educate the team on the throttling trap. Pair on a dashboard panel showing throttle ratio per service so devs see their own pods get throttled.
- **CI/CD**: load-test with the same CPU limits as prod. Block PRs that ship pods with CPU limits but no perf test. If you ship Go services, mandate `automaxprocs` as a dep via lint.
- **Operations**: alert on **throttle ratio**, not just CPU%. Runbook for "service slow": check `cpu.stat`, then GOMAXPROCS, then the limit itself, before reflexively bumping it. Capacity planning should track per-workload p99 CPU burst, not just averages.
