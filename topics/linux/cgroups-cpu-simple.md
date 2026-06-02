# cgroups CPU — the simple version (the buffet analogy)

> Read this first. Once the intuition clicks, the [deep-dive doc](./cgroups-cpu.md) becomes easy.

This doc only explains **two things**, the two that trip everyone up:

1. **What is the "quota and period" thing?** (and why does the kernel slice time into windows at all?)
2. **Why does adding more threads make throttling *worse*, not better?**

No jargon, no cgroup file paths, no Go runtime. Just intuition.

---

## The family buffet

Imagine an all-you-can-eat buffet with one rule:

> **"A family gets 30 minutes of access per hour."**

The hour is the **period**. The 30 minutes is the **quota**. Both are time. Both reset together. Whatever your family doesn't use in this hour, you don't get to roll over — at the top of the next hour, the timer resets and you get a fresh 30 minutes.

Crucially: **it's the *family's* time, not each person's.** If you bring more people, you don't get more time — you share the same 30 minutes.

That's it. That's CFS quota. Everything else is details.

---

## Why does the kernel slice time at all?

You might ask: why doesn't the kernel just say "your pod can use 0.5 CPUs, total"? Why does it have to be "50 ms of CPU per 100 ms of wall-clock"?

Because **"0.5 CPUs" only has meaning when measured over a time window.** Half a CPU over a millisecond is half-a-CPU's worth of work. Half a CPU over a year is, on average, also half a CPU — but along the way you might have used 0 for hours and then 4 cores in parallel for a minute. The average doesn't constrain anything useful.

So the kernel picks a window: **100 ms by default**. Inside each window, you can use as much CPU as your quota allows. When the window ends, the meter resets.

```
period:  | 100ms  | 100ms  | 100ms  | 100ms  | ...
quota:    50ms     50ms     50ms     50ms        ← each period you get 50ms of CPU
```

You can spend that 50 ms however you like *within* the window:
- 50 ms straight at the start, then freeze for 50 ms
- 25 ms, pause, 25 ms more
- 5 ms every 10 ms — keeps you smooth

The kernel doesn't care. It just stops you when you've used up 50 ms.

---

## The single-chef case (easy intuition)

One thread (one chef in the kitchen). Limit is 0.5 CPU (50 ms quota per 100 ms period).

The chef cooks for 50 ms straight, gets told "you're done for now," sits idle for 50 ms, gets told "go again," cooks for 50 ms, and so on.

```
Wall clock →
period 1:    [chef cooks 50ms ][ chef frozen  50ms ]
period 2:    [chef cooks 50ms ][ chef frozen  50ms ]
period 3:    [chef cooks 50ms ][ chef frozen  50ms ]
```

If you measure the chef's CPU usage on a 1-second dashboard, you see **50% CPU**. Makes sense — the chef worked half the time. No surprises.

This is the case that matches your mental model. **One thread + a CPU limit feels like a fair "use half of one CPU."**

---

## The four-chef case (the surprise)

Now you hire 3 more chefs. Same kitchen, same 0.5-CPU quota. **The quota is shared across the whole family — it doesn't multiply by chef count.**

All 4 chefs want to cook. They all start at the beginning of the period. Each one is using a full CPU. Together they burn **4 CPUs' worth of cook-time** every wall-clock second.

How fast do they burn through 50 ms of quota?

**12.5 ms of wall clock.** Because 4 chefs × 12.5 ms each = 50 ms of total cook-time = the entire quota.

At 12.5 ms into the period, the kernel says: **"Your family is done. Everyone out."** All 4 chefs sit frozen. For the next 87.5 ms.

```
Wall clock →
period 1:    [4 chefs cook 12.5ms][ all 4 chefs frozen 87.5ms       ]
period 2:    [4 chefs cook 12.5ms][ all 4 chefs frozen 87.5ms       ]
period 3:    [4 chefs cook 12.5ms][ all 4 chefs frozen 87.5ms       ]
```

Now look at what the dashboard sees. It samples once per second. Over a 1-second window, the chefs cooked for 10 × 12.5 ms = 125 ms. The host has 4 cores so it has 4 × 1000 ms = 4000 ms of CPU available per second. So dashboard CPU = 125 / 4000 = **3.1%**.

**Three percent on the dashboard, 87% of the time frozen.**

This is the trap. Adding threads doesn't make work go faster — it makes the freeze longer. And the freeze is invisible on a 1-second average.

---

## Why is this the famous "30% CPU but slow" bug?

Because the world is full of multi-threaded code that doesn't know it's in a container.

The default for Go, Java (older), Node worker pools, Python multiprocessing — **they all check "how many CPUs does this *host* have?"** They get an answer like 16 (the machine has 16 cores). They spin up 16 worker threads ready to grind.

Your container has a 1-CPU limit (100 ms quota per 100 ms period). When a request comes in, 16 threads wake up and run. They blast through 100 ms of CPU time in about 6 ms of wall clock — and then everything freezes for 94 ms.

The user sees:
- p50 latency: 8 ms (most requests get answered in the first 6 ms before freezing)
- p99 latency: 100+ ms (some unlucky request hit the freeze)
- Dashboard CPU: maybe 20% (only 6 ms of work per 100 ms period × 16 cores = 96 ms of cook time = 6% of 16 cores)

The senior diagnostician sees: **"You're being throttled. Your runtime spawned threads for the host, not the cgroup."**

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| "What's a quota?" | How much CPU-cook-time you can use per period. |
| "What's a period?" | The reset window. Default 100 ms. |
| "Does the period grow with my limit?" | No. Only the quota does. `limits.cpu: 2` = 200 ms quota in a still-100 ms period. |
| "Do more threads = more total CPU?" | **No.** They share the same quota. More threads just means you burn through it faster. |
| "Can I see throttling on my normal CPU dashboard?" | **No.** It happens in 10-ms windows. Your dashboard samples in 30-second windows. |
| "Where do I look?" | The `cpu.stat` file (or in Prometheus: `container_cpu_cfs_throttled_periods_total`). |
| "What's the fix?" | Pick one: (1) raise the limit, (2) tell your runtime about the cgroup quota (`automaxprocs` in Go), or (3) drop the CPU limit on latency-sensitive workloads and rely on requests for fairness. |

---

## Self-test (just one — the one that matters)

Out loud, as if to an interviewer:

> **"A pod is at 30% average CPU but feels slow. Limit is 1 CPU. What could be happening?"**

**Reference answer (intuitive version):**

"It's almost certainly CPU throttling. The kernel enforces the CPU limit in 100 ms windows — so the pod has 100 ms of CPU time per 100 ms wall clock. If the app is multi-threaded — say 16 worker threads on a 16-core node — they all wake up together when a request comes in, burn through the 100 ms of quota in maybe 6 ms of wall clock, and then everything freezes for 94 ms. The dashboard averages over 30 seconds, so it just sees 'about 30% CPU' — but the latency story is brutal because every burst ends in a 94 ms freeze. The fix is usually telling the runtime how many CPUs it actually has — in Go, that's `automaxprocs`. To confirm, check `nr_throttled / nr_periods` in `cpu.stat`, or the equivalent Prometheus metric."

If you got *that* answer out, you've internalized it. The deep-dive doc just adds the file paths and the runtime trap details.

---

## Next: the deep-dive

When the buffet analogy feels obvious, jump to [`cgroups-cpu.md`](./cgroups-cpu.md). The deep-dive covers:

- The actual cgroup files (`cpu.max`, `cpu.weight`, `cpu.stat`)
- How K8s `requests.cpu` and `limits.cpu` map to those files
- The `cpu.weight` story (relative priority under contention)
- The GOMAXPROCS / Java thread / Node `UV_THREADPOOL_SIZE` runtime trap in detail
- Throttling vs starvation (two different failure modes)
- The "CPU limits are an antipattern" debate
- The hands-on (start container → set up watch → trigger burn → see throttling live)

The deep-dive is the reference. This doc is the mental model. Don't read the deep-dive again until this one feels obvious.
