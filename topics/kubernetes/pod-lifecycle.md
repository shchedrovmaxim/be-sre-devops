# Pod lifecycle, graceful shutdown, probes, PDB, sidecars

> **Goal**: by the end you can answer the killer question — **"A rolling deploy is dropping in-flight requests despite probes being green. Walk me through what to check."** — naming the 4 parallel things that happen at termination, why kube-proxy is eventually consistent, the preStop sleep pattern, and how PDB / probes / sidecars fit in.

> Start with the [simple version](./pod-lifecycle-simple.md) if you haven't read it. The airplane-deplaning analogy is the spine of this whole topic.

---

## Pod phases

A pod is always in exactly one phase, reported as `.status.phase`.

| Phase | What it means | Common causes |
|---|---|---|
| **Pending** | Scheduled or about to be. Either waiting for scheduling, image pull, or PVC binding. | Image pull in progress, no node fits (scheduling failure), waiting for PVC. |
| **Running** | At least one container is running (or starting/restarting). | Normal steady state. |
| **Succeeded** | All containers terminated successfully. | Only happens for Jobs / one-shot pods. |
| **Failed** | All containers terminated, at least one in failure (non-zero exit). | Crash, OOMKilled, exit code != 0. |
| **Unknown** | kubelet can't be reached. | Node NetworkPartition. Becoming `Failed` after node controller eviction. |

**"Terminating" is NOT a phase.** It's a pseudo-state that `kubectl get pod` displays when `.metadata.deletionTimestamp` is set. The phase is still `Running` while the pod is shutting down. Worth knowing because interview questions about "what state is a pod in while terminating?" expect "Running with a deletionTimestamp."

---

## The termination sequence — the full diagram

When kubelet sets `deletionTimestamp` on a pod, **four things kick off in parallel**:

```
              deletionTimestamp set on the pod
                        │
   ┌────────────────────┼────────────────────┬─────────────────────┐
   │                    │                    │                     │
   ▼                    ▼                    ▼                     ▼
┌──────────┐    ┌──────────────┐     ┌──────────────┐     ┌────────────────┐
│Endpoints │    │ Readiness    │     │ preStop hook │     │ SIGTERM sent   │
│controller│    │ probes flip  │     │ runs on each │     │ to PID 1 of    │
│removes   │    │ to NotReady  │     │ container    │     │ each container │
│pod IP    │    │ immediately  │     │ (if defined) │     │ (after preStop │
│from      │    │              │     │              │     │  completes)    │
│Service's │    │              │     │              │     │                │
│endpoints │    │              │     │              │     │                │
└────┬─────┘    └──────┬───────┘     └──────┬───────┘     └────────┬───────┘
     │                 │                    │                      │
     ▼                 ▼                    ▼                      ▼
kube-proxy on    Service stops      Hook has X      App must:
each node        routing to this    seconds (eats   - stop accepting
updates iptables pod's IP           into grace        new requests
(EVENTUALLY                         period)         - drain in-flight
consistent —                                        - close DBs, flush
~seconds!)                                            logs, exit clean
     │                                  │                      │
     └──────────────────────────────────┼──────────────────────┘
                                        │
                                        ▼
                terminationGracePeriodSeconds elapses
                                        │
                                        ▼
                       kubelet sends SIGKILL
                                        │
                                        ▼
                            pod removed from API server
```

The four things to internalize:

1. **The Service endpoint update is the slowest path.** kube-proxy on each node has to receive the update and reprogram iptables / IPVS / eBPF. This takes hundreds of milliseconds to seconds. **During this window, other pods still send requests to the dying pod.**
2. **Readiness is the fast path.** The moment `deletionTimestamp` is set, the pod is treated as NotReady for the purposes of any Service it backs. But this only stops *future* endpoint propagation; it doesn't unwind requests that were already routed to in-flight by `kube-proxy`.
3. **`preStop` is your safety window.** Use it to wait for kube-proxy to catch up. The simplest, most reliable pattern is `sleep 10` (or longer).
4. **SIGTERM is the actual shutdown signal.** Your app must handle it. If it doesn't, kubelet `SIGKILL`s the pod at the grace-period deadline, and anything in-flight at that moment dies.

---

## SIGTERM handling in app code

The contract: receive SIGTERM → stop accepting new work → finish current work → exit cleanly.

### Node.js (Express)

```js
const server = app.listen(3000);

let shuttingDown = false;

process.on('SIGTERM', () => {
  console.log('SIGTERM received, draining...');
  shuttingDown = true;

  // Close server: stops accepting new connections, waits for in-flight to drain
  server.close(() => {
    console.log('All requests drained, exiting');
    process.exit(0);
  });

  // Force exit after grace period buffer
  setTimeout(() => {
    console.error('Forced exit after timeout');
    process.exit(1);
  }, 50_000);    // less than terminationGracePeriodSeconds
});

// Readiness probe: report not ready once we start shutting down
app.get('/ready', (req, res) => {
  if (shuttingDown) return res.status(503).send('draining');
  res.send('ok');
});
```

### Go (net/http)

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

// Wait for SIGTERM
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT)
<-sigs

// Drain with a deadline that's less than terminationGracePeriodSeconds
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Printf("Shutdown error: %v", err)
}
```

`srv.Shutdown` stops accepting new connections, waits for in-flight to complete (or until ctx times out), then returns.

### Java (Spring Boot)

Modern Spring Boot has graceful shutdown built in:

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 50s
```

Spring closes the connection acceptor on SIGTERM, lets in-flight requests finish, then exits. Up to you to also stop async tasks, drain queues, etc.

### What every language has in common

- Trap SIGTERM (don't trap SIGKILL — that's not possible).
- Stop accepting new connections / messages / jobs.
- Wait for in-flight to drain, with a deadline less than the grace period.
- Flip readiness to fail (so any load balancer above K8s also stops sending traffic).
- Exit clean. Non-zero exit is reported as `Failed`.

---

## `preStop` hooks — the three variants

```yaml
lifecycle:
  preStop:
    exec:                                # variant 1: run a command
      command: ["sh", "-c", "sleep 10"]
```

```yaml
lifecycle:
  preStop:
    httpGet:                             # variant 2: HTTP to the app
      path: /shutdown
      port: 8080
```

```yaml
lifecycle:
  preStop:
    tcpSocket:                           # variant 3: just open a TCP socket
      port: 8080
```

**When to use each:**

- **`exec` with `sleep`** — by far the most common. Used to wait for kube-proxy to propagate the endpoint removal. Zero app changes required. **This is the answer 90% of the time.**
- **`exec` calling an admin script** — when you need orderly shutdown logic that's hard to do via SIGTERM (close specific connections, flush a queue first).
- **`httpGet`** — when your app needs to receive a "start shutting down" message it can act on at the HTTP layer. Less common; same can usually be done with SIGTERM.
- **`tcpSocket`** — rarely useful; just checks a port is open.

**Critical**: the preStop hook **eats into `terminationGracePeriodSeconds`**. If grace period is 30s and your preStop sleeps 10s, your app has 20s to drain. **Not** 30s.

---

## Grace period budgeting

```
terminationGracePeriodSeconds = preStop time
                              + app drain time (longest in-flight request + overhead)
                              + buffer for unexpected slowness
```

Worked example for a typical web service:

- Longest in-flight request: 15s (a slow DB query in the 99th percentile)
- preStop: `sleep 10` (kube-proxy convergence)
- Buffer: 5s

`terminationGracePeriodSeconds: 30` ← (10 + 15 + 5)

**Default is 30 seconds.** Most apps need to bump this when they have any long-tail requests.

**When to set very large grace periods**:
- Databases / Kafka / stateful workloads (often 600+ seconds)
- Workers processing long jobs

**When to set 0 (`--force` equivalent)**:
- Pre-stopped pods you've already drained externally
- Test workloads where dropped requests don't matter

---

## Probes during termination

Three probe types:

| Probe | Failure result | When it matters at termination |
|---|---|---|
| `livenessProbe` | kubelet **restarts** the container | Pretty much never relevant during graceful termination — the pod is dying anyway |
| `readinessProbe` | Pod removed from Service endpoint list | Implicit: K8s also flips readiness when deletionTimestamp is set |
| `startupProbe` | Disables liveness during startup | Not relevant at termination |

**The mechanism for "readiness goes red the moment termination begins"**: when `deletionTimestamp` is set, the EndpointSlice controller removes the pod IP from the Service's endpoints. Your `readinessProbe` doesn't have to do anything for this to happen. **But** — and this is the gotcha — many ingress controllers and service meshes maintain their *own* view of pod readiness. If your app keeps reporting "ready" via the readiness endpoint *during* termination, the ingress/mesh might still route to it.

**Senior pattern**: make your readiness handler aware of shutdown state, so external load balancers (Istio, NGINX Ingress, AWS NLB target groups) also drain you cleanly:

```go
http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    if shuttingDown.Load() {
        http.Error(w, "draining", 503)
        return
    }
    w.WriteHeader(200)
})
```

When SIGTERM fires, set `shuttingDown = true`. Readiness returns 503. NLB target group health-check fails 2 in a row. Target group stops sending traffic. *Then* you start draining.

---

## PodDisruptionBudget

PDB is a constraint on how many pods of a workload may be **voluntarily** unavailable at once.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-api-pdb
spec:
  minAvailable: 2          # or: maxUnavailable: 1
  selector:
    matchLabels:
      app: my-api
```

**What PDB does**:
- Blocks the `eviction` API call (used by `kubectl drain`, cluster autoscaler, node upgrade tools) when satisfying the eviction would violate the budget.
- Returns 429 to the eviction request; the caller retries.

**What PDB does NOT do**:
- Doesn't protect against involuntary disruption (node crash, kernel panic, OOM, hardware failure).
- Doesn't pause rolling updates from Deployments — those are governed by `maxSurge` / `maxUnavailable` on the Deployment itself, not the PDB.
- Doesn't add up to "I always have N pods running" — only "I never voluntarily go below N at once."

**Common interview probe**: "I have a Deployment with 3 replicas and a PDB `minAvailable: 2`. The node hosting 2 of them crashes. What happens?"

Answer: 2 pods die instantly. PDB doesn't help — it's involuntary disruption. The Deployment controller will start replacements. During the gap you're at 1 replica.

**Combining PDB + grace period + preStop**: the holy trinity of graceful K8s ops. PDB controls *when* a pod is evicted (during node drains, upgrades). preStop and grace period control *how cleanly* it shuts down. You need all three for zero-downtime node upgrades.

---

## Init containers

A pod can declare init containers that run **before** the main containers, **sequentially**, each to completion:

```yaml
spec:
  initContainers:
  - name: migrate-db
    image: my-migrator:1.2
    command: ["./migrate"]
  - name: wait-for-redis
    image: busybox
    command: ["sh", "-c", "until nc -z redis 6379; do sleep 1; done"]
  containers:
  - name: app
    image: my-api:1.2
```

Each init container runs to completion before the next starts. Main containers don't start until *all* init containers exit 0. If an init container fails, the pod restarts the init sequence.

**Use init containers for**:
- DB migrations (the canonical example)
- Waiting for dependencies to be ready
- Fetching configuration / secrets / TLS certs
- Setting up shared volumes (chmod, populate)

**Limits**: probes don't run on init containers. They have requests/limits like main containers. They contribute to the pod's CPU/memory accounting *while running*.

---

## Native sidecars (K8s 1.29+)

The "Istio sidecar dies first" problem: classic sidecars (Envoy, Vault Agent) are just main containers. When the pod terminates, **all main containers get SIGTERM simultaneously**. The app keeps trying to make outbound calls, but Envoy is already shutting down → outbound calls fail → the app's "drain" actually fails to drain.

K8s 1.29 introduced **native sidecars**: an init container with `restartPolicy: Always` becomes a sidecar that:
- Starts **before** main containers (init-container ordering)
- **Stays running** during the main container's life
- Terminates **after** all main containers exit (reverse-init-ordering)

```yaml
spec:
  initContainers:
  - name: istio-proxy           # this is now a "native sidecar"
    image: istio/proxyv2:1.20
    restartPolicy: Always       # ← makes it a sidecar, not a one-shot init
  containers:
  - name: app
    image: my-api:1.2
```

This single field finally solves a problem the K8s community has worked around for 5 years (Istio's `holdApplicationUntilProxyStarts`, the `pilot-agent wait` trick, etc.). If you're on K8s 1.29+ with Istio 1.20+, you should be using this.

**Termination order with native sidecars:**

1. preStop runs on all containers in parallel
2. SIGTERM sent to **main containers** (sidecar still running)
3. Main containers drain and exit
4. **Then** SIGTERM sent to sidecars
5. Sidecars drain and exit

So your app can still make outbound calls through Envoy *during* its own drain. Exactly what you want.

---

## The full troubleshooting flow: "deploy drops requests"

This is the runbook.

### Step 1 — confirm what's failing

```bash
# Look at error rate during deploy in your observability tool
# If 4xx/5xx spikes correlate with rollout, you have the symptom
```

### Step 2 — check the basics

```bash
kubectl get pod <pod> -o yaml | yq '
  .spec.terminationGracePeriodSeconds,
  .spec.containers[].lifecycle,
  .spec.containers[].readinessProbe
'
```

Does the pod have `terminationGracePeriodSeconds`? Default is 30; might be too short for long-tail requests.
Does it have a `preStop` hook? If not, kube-proxy lag is dropping requests.
Does it have a `readinessProbe`? If not, the only way K8s knows the pod is healthy is "the container is running."

### Step 3 — check the app

Does the app have a SIGTERM handler? Read the code or grep for `SIGTERM` / `signal.Notify`. If not, requests die at the grace-period deadline.

### Step 4 — check the rollout config

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0       # ← prefer 0 for zero-downtime
```

`maxUnavailable: 0` forces K8s to bring up new pods *before* taking old ones down. Combined with `maxSurge`, you always have at least `replicas` pods serving during a rollout.

### Step 5 — for service meshes, check sidecar order

If you're on Istio without native sidecars, the istio-proxy can die before the app finishes draining. Fix: upgrade to K8s 1.29+ + Istio 1.20+ and use the native-sidecar pattern.

### Step 6 — check PDB

```bash
kubectl get pdb
kubectl describe pdb my-api-pdb
```

If "Disruptions Allowed: 0," node drains will block. Not a request-drop issue, but a "deploy is mysteriously stuck" issue. Worth checking.

---

## The interview answer in 60 seconds

> "Probes being green tells me the pod was healthy at the moment of the snapshot — but termination is parallel and slow in places. When a pod terminates, four things start at once: the endpoint controller removes its IP from the Service's endpoints, readiness flips red, preStop runs, and SIGTERM is sent. The endpoint removal is **eventually consistent** — kube-proxy on every node has to receive the update and reprogram iptables. That takes hundreds of milliseconds to seconds. **During that window, other pods still send traffic to the dying pod.** If my app is already shutting down or there's no preStop sleep to delay shutdown, those requests get dropped. So I'd check: is there a `preStop: sleep 10`? Does the app handle SIGTERM by stopping new requests and draining in-flight? Is the grace period long enough to cover the longest in-flight request plus the preStop sleep plus a buffer? On rollouts, `maxUnavailable: 0` so I always have a fresh pod ready before taking an old one down. If I'm on Istio, native sidecars (1.29+) so the proxy doesn't die before the app finishes draining. PDB prevents *voluntary* disruption from violating availability during drains, but doesn't help with node crashes."

---

## Self-test

### 1. A rolling deploy is dropping requests despite probes being green. Walk me through what to check.

**Reference answer:** see the 60-second answer above. Hit the three core gaps: preStop sleep (kube-proxy convergence), SIGTERM handler (drain logic), grace period budget (covers longest request + sleep + buffer). Mention rollout strategy (`maxUnavailable: 0`) and native sidecars for service meshes.

### 2. What does PodDisruptionBudget actually protect against?

**Reference answer:**
- **Voluntary disruption only**: node drains, cluster autoscaler removing nodes, controlled node upgrades.
- It returns 429 to the eviction API when satisfying the request would violate the budget.
- It does NOT protect against involuntary disruption (node crash, OOMKill, hardware failure).
- It does NOT govern Deployment rolling updates — that's `maxSurge`/`maxUnavailable` on the Deployment.
- Common misuse: setting `minAvailable: 100%` blocks all evictions, which prevents node upgrades. Use `maxUnavailable: 1` instead.

### 3. What problem do native sidecars (K8s 1.29+) solve?

**Reference answer:**
- Classic sidecars (Envoy/Istio, Vault Agent) are main containers. At termination, all main containers receive SIGTERM at the same time.
- Result: the sidecar starts shutting down while the app is still trying to drain. The app's outbound calls (now going through a dying proxy) fail. The "graceful" drain isn't graceful.
- Native sidecars fix this: init container + `restartPolicy: Always` = sidecar that starts before main containers and **terminates after them**.
- Termination order becomes: preStop → SIGTERM to main containers → main containers drain and exit → SIGTERM to sidecars → sidecars drain and exit.
- Now the app's outbound calls still work *during* its drain.
- This single field replaces years of workarounds (Istio's `holdApplicationUntilProxyStarts`, `pilot-agent wait`, init scripts that block on Envoy readiness).

### 4. Pod is `Terminating` for a long time. What could be happening?

**Reference answer:**
- preStop hook taking too long (e.g., bug in `exec` command, or hook calling a slow endpoint).
- App not respecting SIGTERM, running until SIGKILL — but kubelet still waits the full grace period.
- Very long `terminationGracePeriodSeconds` configured.
- Finalizers on the pod (controllers that need to clean up before the pod is removed from API).
- Volume detach pending (PVC unmount taking time).
- Last resort: `kubectl delete pod --grace-period=0 --force` — but warn that this bypasses cleanup; can corrupt StatefulSet replicas or leave dangling volumes.

---

## The 4 dimensions (senior framing)

- **Tech**: SIGTERM handlers in app code; `preStop: sleep 10` for kube-proxy convergence; grace period = preStop + drain + buffer; `maxUnavailable: 0` for rolling updates; native sidecars on K8s 1.29+; PDB protects only voluntary disruption.
- **People**: dev teams rarely think about SIGTERM until prod drops requests. Bake graceful shutdown into your Helm chart defaults. Pair-review the first few apps that adopt it so the pattern propagates. Document the "rolling deploy drops requests" runbook so on-call doesn't have to reinvent.
- **CI/CD**: Kyverno policies that **require** `terminationGracePeriodSeconds`, **require** a readiness probe, **require** a SIGTERM-handling annotation (you can't statically check the handler exists, but you can require teams to declare it). Block production rollouts of pods without these.
- **Operations**: alert on rollout-correlated error spikes (Prometheus + Argo Rollouts can do this natively). Track p99 termination time per workload — sudden growth means something changed in the drain path. Runbook for "deploy dropping requests": always start at the preStop + SIGTERM + grace period triad before bumping replicas or rolling back.

---

## What we did NOT cover (parked)

- Custom controllers and finalizers
- StatefulSet ordered termination semantics (different rules from Deployments)
- Lifecycle of CronJob pods (Succeeded ≠ deleted)
- KEDA-driven scaling lifecycle

These belong in follow-up sessions.
