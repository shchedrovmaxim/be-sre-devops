# Job + CronJob — the simple version (the contractor analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./job-cronjob.md) becomes easy.

This doc only explains **one thing**:

> **A Job runs a task to completion — then stops. A CronJob is a scheduler that creates a new Job on a schedule. Neither is designed to keep running indefinitely the way a Deployment is.**

---

## The building contractor

A Deployment is like a permanent employee — always at work, handling whatever comes in. A Job is like a contractor hired for a specific project.

| In the contractor world | In the K8s world |
|---|---|
| Hire 3 contractors to paint 10 rooms | `completions: 10, parallelism: 3` — 3 pods running at once until 10 complete |
| If a contractor calls in sick, the agency sends a replacement | Pod failure → K8s creates a new pod (up to `backoffLimit` failures allowed) |
| If you need painting done every Monday morning | That's a **CronJob** — creates a Job on a cron schedule |
| Contractor finishes → gets paid → leaves | Job completes → pods stop, but are kept for logs until TTL expires |
| Hiring the same contractor while last week's is still working | CronJob with `concurrencyPolicy: Allow` — two Jobs overlap |

The key mental flip: **Deployment pods failing is a problem. Job pods completing successfully is the goal.**

---

## The two confusing concepts

### 1. Pod failure vs. Job failure — they're different things

A **pod failure** is one pod crashing (exit code != 0). The Job responds by creating a replacement pod. This is expected and handled.

A **Job failure** is when the total number of pod failures exceeds `backoffLimit`. At that point, K8s gives up and marks the Job as Failed.

```
backoffLimit: 4
```

This means: up to 4 pod failures are tolerated. On the 5th failure, the Job is marked Failed and no more pods are created.

Think of it as: 4 failed contractors hired → you give up on the project.

### 2. The CronJob "too many missed schedules" gotcha

CronJob creates Jobs on a schedule. But if the K8s control plane was down (or the CronJob was paused) for long enough, it "missed" scheduled times.

If more than **100 scheduled times were missed**, K8s stops trying to catch up entirely and just marks the CronJob as having an issue. This is a hardcoded limit.

The related knob is `startingDeadlineSeconds`. If set, the CronJob will only look back that many seconds for missed schedules. If you missed schedules within the deadline, it catches up. If you missed more than the deadline allows, it skips.

```yaml
spec:
  startingDeadlineSeconds: 300   # only catch up if missed within the last 5 minutes
```

Without this set, the "100 missed schedules" limit applies to the dawn of the CronJob's existence — not a rolling window. Setting `startingDeadlineSeconds` is almost always the right call.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does `completions` control? | How many pods must succeed for the Job to be done |
| What does `parallelism` control? | How many pods run concurrently |
| What does `backoffLimit` control? | Max pod failures before the Job is marked Failed (default 6) |
| What happens to pods after a Job completes? | They stick around (for log viewing) until `ttlSecondsAfterFinished` expires |
| What is `concurrencyPolicy`? | What CronJob does if the previous Job is still running: Allow / Forbid / Replace |
| What happens if CronJob misses > 100 schedules? | K8s stops trying to catch up entirely |
| What does `startingDeadlineSeconds` control? | How far back to look for missed schedules |
| Does timezone matter for CronJob? | Yes — K8s 1.27+ has a `timeZone` field; earlier versions use UTC |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through what happens when a CronJob's pod fails — and the common gotchas at scale."**

**Reference answer (intuitive version):**

"When a CronJob's schedule fires, it creates a Job. The Job creates pods. If a pod fails (non-zero exit), the Job creates another replacement pod — this is expected, not an emergency. The Job keeps trying until either a pod succeeds, or the number of failures hits `backoffLimit` (default 6). If `backoffLimit` is reached, the Job is marked Failed. The CronJob itself is unaffected — it'll try again on the next schedule.

Gotchas at scale. **First**: `concurrencyPolicy`. If your job takes 10 minutes but runs every 5 minutes, the default `Allow` policy creates overlapping Jobs. You probably want `Forbid` (skip this run) or `Replace` (kill the old job, start fresh). **Second**: pod cleanup. Completed Jobs keep their pods (for log inspection) until you clean them up. Set `ttlSecondsAfterFinished: 3600` or you'll slowly fill the cluster with completed pod objects. **Third**: the missed-schedule cliff — if the CronJob misses more than 100 schedules (control plane down, CronJob suspended), K8s stops catching up. Set `startingDeadlineSeconds` to bound the lookback window. **Fourth**: timezone. Before K8s 1.27, CronJobs always ran in UTC. If your users expect jobs to run at 2am local time, that's a common source of off-by-N-hours surprises. Use the `timeZone` field on K8s 1.27+."

---

## Further reading / watching

- **Kubernetes Docs — Jobs**: https://kubernetes.io/docs/concepts/workloads/controllers/job/
- **Kubernetes Docs — CronJobs**: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- **`ttlSecondsAfterFinished`**: https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/
- The `timeZone` field for CronJob: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#time-zones

---

## Next: the deep-dive

When the contractor analogy clicks, jump to [`job-cronjob.md`](./job-cronjob.md). The deep-dive covers:

- Job completion modes: `completions` + `parallelism` math
- The distinction between pod failure and Job failure
- `backoffPolicy` variants (OnFailure vs. Never)
- CronJob internals — how Jobs are created, what the `LAST SCHEDULE` means
- `concurrencyPolicy: Allow/Forbid/Replace` in detail
- `startingDeadlineSeconds` — the missed-schedule recovery logic
- Pod cleanup with `ttlSecondsAfterFinished`
- Timezone handling across K8s versions
- 4 self-test drills + 4-dimensions framing
