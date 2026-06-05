# Pod termination — the simple version (the airplane-deplaning analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **When K8s terminates a pod, several things start happening at the same time. If any one of them is slow or buggy, in-flight requests get dropped.**

That's why "my rolling deploy is dropping requests even though probes are green" is the most common K8s production gotcha.

---

## The airplane just landed

Your plane has just landed at the gate. The captain has decided to deplane. Watch what happens **all at once**:

| In the airplane world | In the K8s world |
|---|---|
| The captain announces "we're starting to deplane" | kubelet decides to terminate the pod (sets `deletionTimestamp`) |
| The **"do not board" sign** at the gate lights up — no new passengers come down the jet bridge | The pod's IP is **removed from the Service's endpoint list** — no new requests should be sent here |
| The flight attendants tell people **"grab your bags, prepare to leave"** | The `preStop` hook runs — gives the app warning before shutdown |
| The captain announces **"begin deplaning"** | The kernel sends **SIGTERM** to the app process |
| Passengers (in-flight requests) **finish what they're doing and leave one by one** | The app finishes its in-flight requests and exits gracefully |

If everything goes well, the plane is empty by the time the cleaning crew shows up. Passengers got off, no one was forced.

If things go badly: there's a **30-minute deadline** (the grace period). If anyone is still on the plane at the deadline, **security drags them off** (SIGKILL).

---

## Why requests get dropped — the 3 things that go wrong

### 1. The "do not board" sign is slow

This is the most surprising one. When the pod is being terminated, K8s updates the Service's endpoint list. But the update has to **propagate to every node** in the cluster — that's not instant. Each node's kube-proxy gets the update on a small delay.

> **So for a few seconds after the pod starts shutting down, other pods may still be sending it requests.**

In airline terms: the captain announced deplaning, but the gate agent at the next gate is still printing boarding passes that say "your flight is at gate 7" — so people keep walking toward a plane that's already closing.

This is why **`preStop: sleep 10`** is so common — it gives the cluster time to stop sending traffic before the app actually starts shutting down.

### 2. The app ignores the announcement

The captain announced deplaning, but a passenger has headphones on and didn't hear. They sit there reading their book. At the 30-minute deadline, security drags them off mid-sentence.

In K8s terms: the app doesn't have a SIGTERM handler. It keeps running its current request to completion (good!) but also keeps accepting new ones (bad!) until kubelet hits the grace period and `SIGKILL`s it mid-request. Anyone whose request was in flight at that exact moment gets cut off.

> **Apps must have a SIGTERM handler that:** stops accepting new requests, finishes in-flight ones, then exits cleanly.

In Node: `process.on('SIGTERM', ...)`. In Go: `signal.Notify(c, syscall.SIGTERM)`. Same idea in every language.

### 3. The deadline is too short

You have 30 seconds. Your slowest request takes 25 seconds. A request comes in at second 6 — it finishes at second 31, **1 second after SIGKILL**. Cut off.

Fix: bump `terminationGracePeriodSeconds` until it comfortably covers your slowest request, plus the preStop sleep, plus a buffer.

> **Grace period = preStop time + app's longest in-flight request + buffer.**

---

## The simplest, most reliable graceful-shutdown pattern

For 90% of stateless web services, this is the pattern that works:

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

What this does:

1. **Pod marked for termination.**
2. **preStop sleeps 10 seconds** — gives kube-proxy time to propagate the endpoint removal across all nodes. No new traffic by second 10.
3. **SIGTERM sent to the app** — app has 50 seconds remaining (60 − 10).
4. **App drains** — finishes in-flight requests, doesn't accept new ones.
5. **App exits cleanly.** No SIGKILL needed.

That's literally it. Three lines of YAML, plus a SIGTERM handler in your app. Solves the "rolling deploy drops requests" problem for almost every service.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Why does my deploy drop requests if my probes are green? | Probes are about "is the pod healthy," not "is the pod being kept in the load balancer." Termination removes it from the load balancer in parallel with shutdown — and the removal is **slow to propagate**. |
| Why use `preStop: sleep 10`? | To give the cluster time to stop sending you traffic before you start shutting down. |
| What happens at the grace period deadline? | `SIGKILL`. The pod is force-killed. Any in-flight request dies. |
| Does the grace period start from the preStop or from SIGTERM? | From `deletionTimestamp`. preStop and SIGTERM both eat into the same budget. |
| What if I set the grace period to 0? | Immediate `SIGKILL`. Use only for truly stateless, idempotent workloads where dropped requests don't matter. |
| Why is `kubectl delete pod --force` dangerous? | It bypasses the grace period entirely. preStop never runs. SIGTERM never sent. The pod is just removed from the API. If it was a StatefulSet member, you can corrupt things. |

---

## Self-test (one question — the killer one)

Out loud:

> **"A rolling deploy is dropping in-flight requests, even though probes are green and grace period is 30 seconds. Walk me through what to check."**

**Reference answer (intuitive version):**

"Probes being green tells you the pod was healthy *before* termination — but the question is what happens *during* termination. When a pod starts terminating, four things kick off in parallel: the Service's endpoint list gets updated, the preStop hook runs, SIGTERM is sent to the app, and a countdown starts toward SIGKILL. If any of these is slow or wrong, you drop requests. So I'd check three things. **One:** is there a `preStop: sleep 10` hook? Without it, the cluster might still be sending the dying pod traffic for a few seconds after it starts shutting down, because kube-proxy is eventually consistent across nodes. **Two:** does the app actually handle SIGTERM? If it ignores SIGTERM and just runs until SIGKILL at the deadline, any in-flight request at that moment dies. **Three:** is the grace period long enough? If the slowest request takes 25 seconds and the grace period is 30 seconds, a request arriving 6 seconds in gets cut off. Fix: add the preStop sleep, add a SIGTERM handler that stops accepting and drains, bump the grace period to comfortably cover the longest request plus the sleep plus a buffer."

If you got that out cleanly, you've internalized it. The deep-dive doc adds probe details, PDB, init containers, and native sidecars.

---

## Next: the deep-dive

When the airplane analogy feels obvious, jump to [`pod-lifecycle.md`](./deep-dive.md). The deep-dive covers:

- Pod phases (Pending → Running → Succeeded/Failed) and why "Terminating" is not a phase
- The full termination sequence diagram with all 4 parallel threads
- SIGTERM handling in app code — concrete examples in Node, Go, and Java
- `preStop` hook variants (`exec`, `httpGet`, `tcpSocket`)
- Grace period budgeting math
- How readiness/liveness/startup probes interact with termination
- **PodDisruptionBudget** — what it actually protects against (only voluntary disruption)
- Init containers — the sequential model
- **Native sidecars** (K8s 1.29+) — and how they solve the "Istio sidecar dies before the app finishes draining" problem
- 4 self-test drills
- 4-dimensions framing

The deep-dive is the reference. This doc is the mental model. Don't dive deeper until the airplane analogy feels obvious.
