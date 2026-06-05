# cgroups memory — the simple version (the circuit-breaker in your electrical panel)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **two things**, the two that trip everyone up:

1. **Why is the pod killed — and how does the kernel pick which process to kill?**
2. **What is cgroup OOM vs system OOM, and why does it matter which one you're looking at?**

No cgroup file paths, no kernel page-allocation internals, no QoS YAML. Just intuition.

---

## The electrical panel in your house

Imagine your house has a **100-amp main breaker** and each room has its own **circuit breaker** in the panel.

- The main breaker protects the whole house. If everything draws too much at once, the main trips and the whole house goes dark.
- Each room's breaker protects only that room. If the kitchen overloads its circuit, **only the kitchen goes dark.** The rest of the house keeps running.

Now imagine each container is a room. Kubernetes told the panel: *"give the API server room a 512-MiB breaker."*

When the API server tries to draw more than 512 MiB of memory:

1. The kernel notices the room's breaker is about to trip.
2. It tries a quick fix first: throw out things that can be easily replaced (file cache — think: turning off the microwave to shed load).
3. If that's not enough, it trips the breaker — kills the most power-hungry appliance in that room (the process with the highest OOM score) with `SIGKILL`.
4. The room's breaker resets. The rest of the house never noticed.

That's cgroup memory management. The room's breaker is `memory.max`. The kill is the cgroup OOM killer. The "rest of the house keeps running" is the isolation cgroups give you.

---

## From `kubectl describe pod` down to the kernel

You've seen this:

```
Last State: Terminated
  Reason:    OOMKilled
  Exit Code: 137
```

Here is the full chain, one step at a time:

| What you see | What's actually happening |
|---|---|
| `Exit Code: 137` | `128 + 9` — the process received signal 9 (`SIGKILL`) |
| `Reason: OOMKilled` | kubelet observed the exit, checked that `memory.events` incremented `oom_kill` |
| The kill itself | The kernel's cgroup OOM killer picked the highest-scoring process in the cgroup |
| Why that process | `oom_score_adj` — a knob kubelet sets per pod when it starts. Higher score = more killable. |
| Why now | The container's cgroup hit `memory.max` and page reclaim didn't free enough |
| Where `memory.max` came from | Kubelet wrote `limits.memory: 512Mi` from the pod spec into the cgroup file |

Nothing here involves Kubernetes specifically — the kernel only sees bytes and cgroup limits. K8s is just the thing that writes the numbers into the right files.

---

## The 4 cgroup files you need at intuition level

On a modern node (cgroup v2), each container has a directory under `/sys/fs/cgroup/kubepods.slice/...`. Inside it:

| File | What it is | Analogy |
|---|---|---|
| `memory.max` | The hard ceiling — trying to cross it triggers the OOM killer | The circuit breaker's rated amperage |
| `memory.high` | A soft warning threshold — crossing it slows allocations but doesn't kill | A "caution" light before the breaker trips |
| `memory.current` | Live memory usage right now | The live amp meter on the panel |
| `memory.events` | An audit log — counts how many times `max` was hit, how many OOM kills happened | The trip counter on the breaker |

**The most important gotcha:** Kubernetes does **not** set `memory.high` by default. It only sets `memory.max`. That means there is no "slow down first, then kill" — it goes straight to kill. This is intentional: K8s prefers a clean, observable failure (`OOMKilled`) over a pod that mysteriously slows down for an unknown reason. A breaker that trips cleanly is better than one that just makes the lights flicker.

**The most useful for debugging:** `memory.events`. When a pod gets `OOMKilled`, this is the file that proves it was a *cgroup-level* OOM and not something else. Watch `oom_kill` count in this file to see the kill happen in real time.

---

## Cgroup OOM vs system OOM — two different rooms

This is the distinction that separates junior and senior answers:

| | Cgroup OOM | System OOM |
|---|---|---|
| **Trigger** | One container's `memory.max` is hit | The entire node runs out of memory |
| **Scope of kill** | Only processes in that container's cgroup | Any process on the host — random pods, kubelet, system daemons |
| **What you see** | `OOMKilled` on a specific pod | Random pods dying, node goes `NotReady`, chaos |
| **`memory.events` updated?** | Yes, on the offending container | No — it wasn't a cgroup event |
| **Healthy?** | Normal, expected — the limit did its job | A bug — usually wrong `kube-reserved` / `system-reserved` settings |

**The interview tell:** if someone describes "a pod died overnight and it wasn't even using much memory," the right response is not "OOMKilled." It's: "That sounds like a system-level OOM, not a cgroup OOM. The cgroup OOM kills the right pod predictably; the system OOM kills randomly. I'd check `dmesg` on the node and the kubelet eviction logs — likely a reservation misconfiguration."

---

## How the kernel picks who dies — QoS and `oom_score_adj`

When the cgroup OOM killer fires, it walks every process in the cgroup and picks the one with the highest **OOM score**. That score is the process's intrinsic memory footprint plus `oom_score_adj` — a bias kubelet sets for each pod at startup.

| QoS class | `oom_score_adj` | What it means |
|---|---|---|
| **BestEffort** (no requests, no limits) | 1000 (maximum) | "Kill me first" — the kernel prefers this |
| **Burstable** (requests ≠ limits) | Computed from request size | Somewhere in the middle — bigger request = lower score |
| **Guaranteed** (requests == limits) | -997 (minimum) | "Kill me last" — the kernel avoids this |

The intuition: **requesting more memory is a kernel-level insurance policy.** A pod that declared `requests.memory: 1Gi` has a lower `oom_score_adj` than one that declared `requests.memory: 128Mi`, even if both are using the same amount right now. The request is a promise that earns protection.

The practical rule: **BestEffort dies first, Guaranteed dies last.** If you have a pod that must not be killed under node pressure, it needs `requests == limits` on both CPU and memory.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What triggers a cgroup OOM kill? | The container hits `memory.max` and page reclaim isn't enough. |
| Why exit code 137? | `128 + 9` — `SIGKILL` is signal 9. |
| What does `memory.high` do? | Slows allocations as a warning. K8s doesn't set it — it goes straight to `memory.max`. |
| How do I see evidence of an OOM? | `memory.events` — look for `oom_kill` counter incrementing. |
| Does killing one pod affect others? | No. The OOM killer is scoped to the cgroup. Other rooms keep their lights on. |
| What kills pods randomly (not cgroup OOM)? | System-level OOM — node ran out of memory entirely. Different problem, different fix. |
| What determines kill order under node pressure? | `oom_score_adj` — set by kubelet based on QoS class. BestEffort dies first. |
| Is my bigger memory request worth it? | Yes — a bigger `requests.memory` lowers `oom_score_adj` and makes the kernel less likely to kill your pod. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what happens at the cgroup level when a pod exceeds its memory limit. Why is it killed, and how does the kernel choose what to kill?"**

**Reference answer (intuitive version):**

"When the pod's container tries to allocate memory beyond its `memory.max` limit — which kubelet set from `limits.memory` in the pod spec — the kernel first tries to reclaim: drop file cache, evict pages. On K8s nodes swap is usually disabled, so if reclaim doesn't free enough, the kernel invokes the cgroup OOM killer. It's *scoped to that cgroup*, so it walks only the processes in that container's cgroup, ranks them by OOM score — intrinsic footprint plus `oom_score_adj` — and sends `SIGKILL` to the highest scorer. The process exits with code 137 (`128 + 9`). Kubelet sees that, reports `OOMKilled`, and the event is recorded in `memory.events` as `oom_kill`.

The kernel's choice of victim is heavily influenced by QoS class: kubelet sets `oom_score_adj` to 1000 for BestEffort pods (die first), -997 for Guaranteed pods (die last), and a computed middle value for Burstable — proportional to how much memory was requested relative to node capacity. So a bigger memory request is literally a kernel-level insurance policy.

Critically, this is a *cgroup* OOM, not a *system* OOM. Only processes in the offending cgroup are at risk. Other pods on the same node keep running. If you see random pods dying without `memory.events` tracking it, that's a system-level OOM — a different problem meaning the node itself ran out of memory, usually because `kube-reserved` is misconfigured."

If you got the cgroup OOM vs system OOM distinction, `oom_score_adj`, and the QoS class order out clearly — you've internalized this. The deep-dive adds the hands-on exercise and the full kernel sequence.

---

## Further reading / watching

The canonical sources:

- **Linux kernel docs** — [cgroups v2 memory controller](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory). Dense, but the `memory.events` section is worth reading directly.
- **Brendan Gregg's blog** — search "Linux memory analysis" for how `oom_score_adj` and the OOM killer work at the system level.

For K8s context specifically:

- **K8s docs — Configure Memory Resources for Containers** (kubernetes.io). The official walkthrough of `requests` vs `limits` and what maps to which cgroup file.
- **K8s docs — Node-pressure Eviction** — covers how kubelet decides to evict pods *before* a system OOM, and how QoS class drives eviction order.

A useful tool:

- **`kubectl top pod`** — shows live memory usage. Compare against `limits` to see how close a pod is to its `memory.max` before it gets killed.

---

## Next: the deep-dive

When the circuit-breaker analogy feels obvious, jump to [`cgroups-memory.md`](./deep-dive.md). The deep-dive covers:

- The complete kernel allocation sequence (malloc → page fault → charge → reclaim → kill)
- The 4 cgroup files with their actual file contents and what each number means
- Why K8s intentionally skips `memory.high` (and the beta `MemoryQoS` feature gate that uses it)
- The cgroup namespace trap (`cat /proc/self/cgroup` shows `/` inside a container — why)
- Hands-on exercise: watch `memory.events` flip from `oom_kill 0` to `oom_kill 1` in real time with Docker
- cgroup v1 vs v2 differences (one paragraph — just enough for the interview)
- The 4-dimensions framing (Tech, People, CI/CD, Operations) applied to memory limits
- 3 self-test drills with reference answers

The deep-dive is the reference. This doc is the mental model. The circuit-breaker-per-room is what you keep in your head during any OOMKilled conversation.
