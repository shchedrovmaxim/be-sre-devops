# Deployment + ReplicaSet rolling updates — the simple version (the relay race handoff)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one thing**:

> **A rolling update is a controlled handoff. The Deployment owns the race. Each generation of runners is a ReplicaSet. The rollout controller makes sure you never have fewer runners on the track than you've configured.**

---

## The relay race

Your team is running a 10-person relay. You want to swap in a new generation of runners (new image tag) without stopping the race.

| In the relay world | In the K8s world |
|---|---|
| Race manager (coach) | **Deployment** controller |
| Each generation of runners | **ReplicaSet** (RS) |
| Individual runners | **Pods** |
| "Swap in 2 new runners before pulling 2 old ones" | `maxSurge: 2, maxUnavailable: 0` |
| New runners waiting in the warm-up lane | The new RS scaling up before the old RS scales down |
| Old runners walking off the track | Old RS pods getting terminated |
| The coach's clipboard tracking both teams | The Deployment's revision history (`--revision`) |

With `maxUnavailable: 0` (the zero-downtime setting): **the coach always brings a new runner on before pulling an old one**. The race never has fewer than 10 people running. It just gradually all becomes the new team.

With `maxUnavailable: 1`: the coach is OK with 9 runners for a moment. Faster handoff, brief capacity drop.

---

## The two confusing concepts

### 1. Deployment → ReplicaSet → Pods (it's two levels, not one)

Most people think a Deployment directly manages Pods. It doesn't.

```
Deployment
  └── ReplicaSet (v1, old)   ← still owns pods 1, 2
  └── ReplicaSet (v2, new)   ← owns new pods 3, 4, 5
```

A Deployment is a rollout manager. It manages ReplicaSets. Each ReplicaSet owns one "generation" of Pods and keeps exactly N of them running. When you update a Deployment (new image tag, new env var), it creates a **new** RS and scales the two in tandem.

Old RS is NOT deleted — it's scaled to 0 and kept for rollback. `kubectl rollout undo` just scales the old RS back up.

### 2. `maxSurge` and `maxUnavailable` are about the transition, not the steady state

Imagine you have `replicas: 4`:

| Setting | During rollout | What it means |
|---|---|---|
| `maxSurge: 1, maxUnavailable: 0` | Up to 5 pods running, never below 4 | Safe for production: never lose capacity |
| `maxSurge: 0, maxUnavailable: 1` | Max 4 pods, at least 3 running | Smaller footprint, brief capacity dip |
| `maxSurge: 25%, maxUnavailable: 25%` | K8s rounds up surge, rounds down unavailable | Common default; percentages get rounded |

The math is: **max pods = replicas + maxSurge**. **Min pods = replicas − maxUnavailable**. During the rollout, K8s stays inside those bounds.

With `maxUnavailable: 0`: new pods must be `Ready` before old pods are terminated. Your readiness probe is what gates the rollout.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does a Deployment actually own? | ReplicaSets, not Pods directly |
| What does a ReplicaSet own? | A fixed generation of Pods |
| Why are old ReplicaSets kept around? | Rollback — `kubectl rollout undo` scales the old RS back up |
| What is `maxSurge`? | How many extra pods above `replicas` are allowed during rollout |
| What is `maxUnavailable`? | How many pods below `replicas` are allowed during rollout |
| What gates a zero-downtime rollout? | The readiness probe — new pod must be Ready before old pod dies |
| How do you pause a rollout mid-way? | `kubectl rollout pause deployment/<name>` |
| How do you check rollout history? | `kubectl rollout history deployment/<name>` |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what happens, step by step, when I push a new image tag to a Deployment with `maxSurge: 25%, maxUnavailable: 0`."**

**Reference answer (intuitive version):**

"With 4 replicas, `maxSurge: 25%` rounds up to 1 extra pod allowed. `maxUnavailable: 0` means I can never be below 4 ready pods.

Step 1: Deployment controller sees the new image tag and creates a new ReplicaSet (v2). The old RS (v1) still owns all 4 pods.

Step 2: New RS scales up 1 pod (to the surge limit of 5 total). That pod runs, passes its readiness probe, becomes Ready.

Step 3: Now I have 5 ready pods. That's 1 over replicas, so I can terminate 1 old pod. Old RS scales down to 3. Total: 4 ready (1 new + 3 old). Within limits.

Step 4: New RS scales up another pod. Again waits for Ready. Then old RS scales down 1 more. Repeat.

Step 5: Eventually old RS is at 0, new RS is at 4. Rollout complete.

If a new pod fails its readiness probe, K8s stops the rollout — it can't scale down old pods because that would go below 4 ready. The rollout stalls and you get a `kubectl rollout status` timeout. That's the safety net."

---

## Further reading / watching

- **Kubernetes Docs — Deployments**: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- **`kubectl rollout` cheatsheet**: `kubectl rollout status`, `history`, `pause`, `resume`, `undo`
- **Argo Rollouts** (for progressive delivery beyond what Deployment offers): https://argoproj.github.io/rollouts/
- **Kubernetes by Example** — Deployments: https://kubernetesbyexample.com/

---

## Next: the deep-dive

When the relay-race handoff feels obvious, jump to [`deployment-rolling-update.md`](./deep-dive.md). The deep-dive covers:

- The exact rollout algorithm — how the controller decides when to surge and when to drain
- Revision history and rollback mechanics
- `kubectl rollout pause/resume` and what to do when a rollout is stuck
- The `progressDeadlineSeconds` setting
- Argo Rollouts: canary and blue-green strategies that go beyond the built-in rolling update
- 4 self-test drills + 4-dimensions framing
