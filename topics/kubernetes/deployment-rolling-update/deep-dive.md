# Deployment + ReplicaSet — rolling update mechanics (deep-dive)

> **Goal**: by the end you can answer the killer question — **"Walk me through what happens, step by step, when I push a new image tag to a Deployment with `maxSurge: 25%, maxUnavailable: 0`."** — naming the Deployment→RS→Pod ownership chain, the surge-first drain-later algorithm, what gates progress, rollback mechanics, and where Argo Rollouts fits.

> Start with the [simple version](./simple.md) if you haven't read it. The relay-race analogy is the spine of this whole topic.

---

## Mental model: two levels of ownership

The most common misconception: a Deployment directly manages Pods. It doesn't.

```
Deployment (the rollout controller)
  └── ReplicaSet v1  (image: my-api:1.0)  — scaled to 0 after rollout
  └── ReplicaSet v2  (image: my-api:2.0)  — scaled to 4 (replicas)
        ├── Pod v2-abc
        ├── Pod v2-def
        ├── Pod v2-ghi
        └── Pod v2-jkl
```

- **ReplicaSet** has one job: keep exactly N pods running that match its pod template hash. It owns the pods.
- **Deployment** has one job: manage the transition between ReplicaSets. It owns the RS objects.

When you change anything in `spec.template` (image, env var, volume mount, resource limit), the Deployment creates a **new** RS and coordinates the hand-off. The old RS is scaled to 0 but kept (controlled by `spec.revisionHistoryLimit`, default 10).

Why keep the old RS? Rollback. `kubectl rollout undo` just scales the previous RS back up. No new image pull needed; the RS definition is already there.

---

## The rollout algorithm — step by step

Setting: `replicas: 4`, `maxSurge: 25%` (rounds up → 1), `maxUnavailable: 0`.

So the invariants during rollout are:
- **Maximum pods**: 4 + 1 = **5**
- **Minimum ready pods**: 4 − 0 = **4**

The controller's loop:

```
1. Create RS v2 (new image). Start at 0 replicas.
2. Can I surge? current_total (4) < max_total (5)? YES → scale RS v2 up by 1.
3. Wait for the new pod to pass readinessProbe → becomes Ready.
4. Now ready_count (4 old + 1 new = 5 ready). Can I drain? ready_count > min_ready (4)? YES → scale RS v1 down by 1.
5. Back to step 2. Repeat until RS v1 = 0, RS v2 = 4.
```

At every step, the controller checks both invariants before acting. If the new pod fails readiness (crashloop, bad config), the controller can't scale down the old RS — that would violate `minReady`. The rollout **stalls**. No harm is done to the old pods.

### What if `maxUnavailable: 1` instead?

```
Minimum ready pods: 4 − 1 = 3
Maximum pods: 4 + 0 = 4 (with maxSurge: 0)
```

The controller can drain an old pod *immediately*, before the new one is ready. Faster rollout, brief capacity dip. Acceptable for internal tools; risky for high-traffic services.

### Percentages and rounding

`maxSurge` rounds **up** (to be safe — better to have too many than too few). `maxUnavailable` rounds **down** (to be safe — better to be unavailable less than promised). With 3 replicas and `25%`:

- `maxSurge: 25%` → 0.75 → rounds up to **1**
- `maxUnavailable: 25%` → 0.75 → rounds down to **0**

So with 3 replicas and `25/25%`, you effectively get `maxSurge: 1, maxUnavailable: 0` — zero downtime.

---

## The YAML (annotated)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 4
  revisionHistoryLimit: 5          # keep last 5 RSes for rollback; default 10
  progressDeadlineSeconds: 300     # fail the rollout if no progress for 5 min
  selector:
    matchLabels:
      app: my-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # or "25%" — extra pods above replicas
      maxUnavailable: 0            # pods allowed below replicas (0 = zero downtime)
  template:
    metadata:
      labels:
        app: my-api
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: my-api:2.0
        readinessProbe:            # REQUIRED for safe rolling updates
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]
```

Key callouts:
- `readinessProbe` is the gate — without it, K8s assumes a running container is ready, and the surge-drain loop will drain old pods immediately.
- `preStop: sleep 10` is still needed here (kube-proxy convergence). See [pod-lifecycle.md](../pod-lifecycle/deep-dive.md).
- `progressDeadlineSeconds` is the timeout on the whole rollout, not per-pod.

---

## Rollback

```bash
# Check revision history
kubectl rollout history deployment/my-api

# Roll back to previous revision
kubectl rollout undo deployment/my-api

# Roll back to a specific revision
kubectl rollout undo deployment/my-api --to-revision=3

# See what's in revision 3 before rolling back
kubectl rollout history deployment/my-api --revision=3
```

Under the hood: `undo` sets `.spec.template` back to the old RS's pod template. The Deployment then treats this as a new rollout — the old RS (now "new" again) scales up, and the current RS scales down. Same surge-drain algorithm. No special rollback path.

**Gotcha**: `kubectl rollout undo` does NOT undo ConfigMap or Secret changes. If you pushed both a new image AND a new ConfigMap, undo brings back the old image but the pods still use the new ConfigMap. For full rollback, you need to undo the ConfigMap separately.

---

## Pausing and resuming

```bash
# Pause mid-rollout (useful for canary-style validation)
kubectl rollout pause deployment/my-api

# While paused: make additional changes (multiple changes, one rollout)
kubectl set image deployment/my-api app=my-api:2.0
kubectl set env deployment/my-api FEATURE_FLAG=true

# Resume — now both changes roll out together
kubectl rollout resume deployment/my-api
```

Pausing lets you "batch" several spec changes into a single rollout instead of triggering one rollout per change. Also useful for manual canary validation: pause after a partial rollout, validate metrics, then resume (or undo).

`kubectl rollout status` blocks until complete:

```bash
kubectl rollout status deployment/my-api --timeout=5m
```

Returns exit code 0 on success, non-zero on timeout or failure — making it suitable for CI/CD pipeline gates.

---

## When a rollout gets stuck

Signs: `kubectl rollout status` is hanging; `kubectl get pods` shows a mix of old and new pods stuck in pending or not-ready.

Diagnostic:

```bash
kubectl describe deployment my-api       # look at "Conditions" section
kubectl get rs -l app=my-api             # see current/desired/ready for each RS
kubectl describe rs <new-rs>             # look at events — scheduling failures?
kubectl get events --field-selector reason=FailedCreate
```

Common causes:
- New pod fails readiness (bad config, OOM, crashloop)
- New pod can't be scheduled (insufficient CPU/memory, taint mismatch)
- `progressDeadlineSeconds` (default: 600s) elapsed — Deployment marks as `ProgressDeadlineExceeded`, but it does NOT auto-rollback; you must do it manually

---

## Progressive delivery with Argo Rollouts

The built-in Deployment rolling update is binary: old → new, with simple surge/unavailable knobs. For production traffic patterns you often want:

- **Canary**: send 5% of traffic to new version, monitor error rate, gradually increase
- **Blue-green**: spin up a full new environment, cut traffic all at once via Service swap
- **Analysis**: automated metric checks mid-rollout (if p99 latency goes up by > 20%, pause)

**Argo Rollouts** replaces the `Deployment` kind with a `Rollout` kind and adds this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-api
spec:
  replicas: 4
  strategy:
    canary:
      steps:
      - setWeight: 10        # send 10% to new version
      - pause: {}            # wait for human approval (or automated analysis)
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: error-rate-check
        startingStep: 1
```

Argo Rollouts integrates with ingress controllers (NGINX, Istio, AWS ALB) to do traffic splitting at the L7 level — not just by pod count. A 10% canary gets 10% of HTTP requests regardless of whether you have 1 or 100 pods.

When to reach for Argo Rollouts: any service where "suddenly 100% of users hit the bad version" is unacceptable. For low-risk internal services, the Deployment rolling update is fine.

---

## The interview answer in 60 seconds

> "With `replicas: 4` and `maxSurge: 25%, maxUnavailable: 0`, the Deployment controller creates a new ReplicaSet for the new image. `maxSurge: 25%` rounds up to 1 extra pod allowed, so the max pods during rollout is 5. `maxUnavailable: 0` means we must always have at least 4 ready pods.
>
> The controller runs a surge-then-drain loop. Step 1: scale new RS up by 1. Wait for that pod to pass the readiness probe. Step 2: now we have 5 ready, which is above our minimum — scale old RS down by 1. Repeat until old RS hits 0.
>
> If the new pod fails readiness, the controller can't drain old pods — it would violate `minReady: 4`. The rollout stalls. Safe.
>
> The old RS is kept (scaled to 0) for rollback. `kubectl rollout undo` scales the old RS back up using the same algorithm. Under the hood, a rollback is just another rolling update with a different target template.
>
> For more sophisticated patterns — canary, blue-green, automated analysis — you'd use Argo Rollouts instead of the built-in Deployment strategy."

---

## Self-test

### 1. Walk me through the rolling update algorithm with `maxSurge: 25%, maxUnavailable: 0` and 4 replicas.

**Reference answer:** New RS created at 0 replicas. Surge up 1 → wait for Ready → drain old 1. Repeat. At every step: total ≤ 5, ready ≥ 4. Stalls if new pod fails readiness (can't drain old pod). Completes when old RS = 0.

### 2. I ran `kubectl rollout undo` but my pods are still using the new ConfigMap. Why?

**Reference answer:** `rollout undo` reverts `.spec.template` (image, env, etc.) but does NOT revert external objects like ConfigMaps and Secrets. If the config is mounted from a ConfigMap that was also changed, the pods will use the old image but still read from the (new) ConfigMap. You need to separately revert the ConfigMap.

### 3. A Deployment rollout has been stuck for 10 minutes. `kubectl rollout status` is hanging. What do you check?

**Reference answer:** `kubectl describe deployment` for `ProgressDeadlineExceeded` condition. `kubectl get rs` to see which RS has 0 Ready. `kubectl describe rs <new-rs>` and `kubectl get events` to find the root cause — usually: new pods failing readiness (crash loop, bad config, OOM, missing resource), or pods can't be scheduled (taint mismatch, insufficient resources). Note: K8s does NOT auto-rollback on `ProgressDeadlineExceeded`. You must `kubectl rollout undo` manually.

### 4. When should you use Argo Rollouts instead of a plain Deployment?

**Reference answer:** When "100% of users hitting a bad version" is unacceptable — typically for revenue-critical or user-facing services. Argo Rollouts enables canary (traffic-weighted, not just pod-count weighted), blue-green, and automated analysis against Prometheus metrics mid-rollout. Also when you need GitOps-friendly progressive delivery (Argo CD + Argo Rollouts work natively together). For internal tooling or low-risk services, the built-in rolling update is sufficient.

---

## Further reading / watching

- **Kubernetes Docs — Deployments**: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- **Kubernetes Docs — ReplicaSet**: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- **Argo Rollouts**: https://argoproj.github.io/rollouts/ — especially the "Getting Started" and "Canary strategy" pages
- **"How Deployments Work" by Ahmet Alp Balkan** — great YouTube walkthrough of the controller internals
- **`kubectl rollout` reference**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/#updating-resources

---

## The 4 dimensions (senior framing)

- **Tech**: Deployment → RS → Pod ownership chain; surge-first drain-later algorithm; `maxUnavailable: 0` for zero downtime; readiness probe as the gating signal; `revisionHistoryLimit` controls rollback depth; Argo Rollouts for canary/blue-green with metric-based analysis.
- **People**: Developers should understand that "I changed the image" triggers a rollout, and a bad readiness probe stalls it until someone investigates. Make rollout visibility a first-class thing: `kubectl rollout status` in CI, Argo Rollouts UI in staging. Runbook for "rollout stuck" should be in the team wiki, not only in one SRE's head.
- **CI/CD**: Gate on `kubectl rollout status --timeout=N` in the deploy step — non-zero exit = pipeline fails = someone gets paged. Set `progressDeadlineSeconds` to match your expected rollout time. In GitOps: ArgoCD watches the Deployment in git and syncs; Argo Rollouts can be the progressive-delivery layer on top.
- **Operations**: Alert on `DeploymentReplicaSetProgressDeadlineExceeded` (K8s exposes this as a condition). Track rollout duration as a metric over time — sudden increases signal scheduling pressure or readiness regressions. Keep `revisionHistoryLimit` reasonable (5-10); too many old RSes clutters the namespace.
