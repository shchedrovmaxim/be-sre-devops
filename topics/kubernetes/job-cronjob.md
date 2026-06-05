# Job + CronJob — run-to-completion workloads (deep-dive)

> **Goal**: by the end you can answer the killer question — **"Walk me through what happens when a CronJob's pod fails — and the common gotchas at scale."** — naming pod failure vs. Job failure, `backoffLimit`, `concurrencyPolicy`, the missed-schedule cliff, `startingDeadlineSeconds`, `ttlSecondsAfterFinished`, and timezone handling.

> Start with the [simple version](./job-cronjob-simple.md) if you haven't read it. The contractor analogy is the spine of this topic.

---

## Mental model: run-to-completion, not run-forever

Deployment: "keep N pods running forever, restart on failure."
Job: "run until N successful completions, tolerate up to M pod failures."

This shifts what "failure" means:
- In a Deployment: a pod failing is an **incident**.
- In a Job: a pod failing is **expected and handled** — the Job retries. A Job failing is when retries are exhausted.

---

## Job fundamentals

### `completions` and `parallelism`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-processor
spec:
  completions: 10        # total successful completions needed
  parallelism: 3         # pods running at the same time
  backoffLimit: 4        # max pod failures before Job is marked Failed
  ttlSecondsAfterFinished: 3600  # delete Job + pods 1h after completion
  template:
    spec:
      restartPolicy: Never   # or OnFailure — see below
      containers:
      - name: processor
        image: my-processor:1.0
```

The Job controller keeps track:
- **Successful**: pods that exited 0
- **Failed**: pods that exited non-zero

It creates new pods until `successful == completions`, while keeping running pods ≤ `parallelism`.

Example with `completions: 10, parallelism: 3`:

```
Start: 3 pods running
Pod 1 succeeds → 1 successful. Start pod 4.
Pod 2 fails    → 1 failed (1/backoffLimit). Start pod 5.
Pod 3 succeeds → 2 successful. Start pod 6.
... continue until 10 successes or 4 failures.
```

If `completions` is not set but `parallelism` is: the Job completes when any one pod succeeds (the "work queue" pattern — multiple workers racing to completion).

### `restartPolicy`

Job pods can't have `restartPolicy: Always` (that's for Deployments). Two valid options:

- **`OnFailure`**: when a pod fails, kubelet restarts the container in the same pod. The pod stays; the container restarts. Simpler, but the failed container's logs are lost on restart unless you use a log shipper.
- **`Never`**: when a pod fails, kubelet leaves it terminated and the Job controller creates a new pod. The failed pod stays around for log inspection (until TTL). This is the more observable choice — you can `kubectl logs` the failed pod.

For batch workloads, **`Never` is usually better for debuggability**. For jobs that fail frequently due to transient errors (network blips), `OnFailure` keeps pod count lower.

### `backoffLimit` — pod failure vs. Job failure

```yaml
spec:
  backoffLimit: 6        # default is 6
```

Each time a pod fails (non-zero exit or eviction), the Job records a failure. When failures reach `backoffLimit`, the Job is marked **Failed** and no more pods are created.

Important: failures are cumulative across all pod generations, not just the current batch. If parallelism is 3 and all 3 pods fail simultaneously, that's 3 failures burned in one moment.

The Job failure event:
```bash
kubectl describe job my-job
# Events:
#   Warning  BackoffLimitExceeded  Job has reached the specified backoff limit
```

**Backoff behavior**: between retries, K8s applies exponential backoff — 10s, 20s, 40s, ... capped at 6 minutes. This prevents a crashlooping pod from hammering infrastructure.

### `ttlSecondsAfterFinished` — the cleanup gotcha

Without this field, completed Jobs and their pods **stick around forever**. In a cluster running hundreds of CronJobs daily, this means hundreds of completed pod objects accumulating.

```yaml
spec:
  ttlSecondsAfterFinished: 3600    # delete 1 hour after Job finishes
```

The TTL controller deletes the Job object (and associated pods) after this duration post-completion. Without it, you need manual cleanup or a garbage collector tool.

**Warning**: setting TTL means logs are gone after the TTL. Balance between keeping logs for debugging and keeping the cluster clean. Common pattern: short TTL (1-4h) for routine jobs, no TTL for jobs that might need investigation.

---

## CronJob internals

### How CronJobs create Jobs

A CronJob is a meta-controller. It watches the clock and creates Job objects on schedule. Each execution is a separate Job with a unique name:

```
my-report-cron-1748890800
my-report-cron-1748977200
my-report-cron-1749063600
```

The CronJob controller runs a reconcile loop checking: "is it time to create a Job?" It compares the last scheduled time with the cron schedule and creates Jobs for any missed windows (up to the limit).

### `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`

```yaml
spec:
  successfulJobsHistoryLimit: 3    # keep last 3 successful Jobs (default 3)
  failedJobsHistoryLimit: 1        # keep last 1 failed Job (default 1)
```

These control how many finished Job objects the CronJob keeps around. Old ones are garbage collected. This is separate from `ttlSecondsAfterFinished` (which deletes after time elapsed).

---

## `concurrencyPolicy` — the overlapping-runs problem

What should happen if the previous Job is still running when the next schedule fires?

```yaml
spec:
  concurrencyPolicy: Forbid    # or Allow or Replace
```

| Policy | Behavior |
|---|---|
| `Allow` | Creates a new Job regardless of whether the previous one finished. Both run simultaneously. |
| `Forbid` | Skips the new scheduled run if the previous Job is still running. A skip is logged but not an error. |
| `Replace` | Deletes the currently running Job (and its pods) and creates a new one. |

**Which to choose**:
- `Allow`: for idempotent, independent jobs where overlap is fine (e.g., read-only reports).
- `Forbid`: for jobs that would conflict if run simultaneously (e.g., DB migrations, cleanup scripts). This is the safest default.
- `Replace`: for jobs where "latest run" is what matters and stale runs should be killed (e.g., cache pre-warming where an old run is just wasted compute).

**Practical gotcha with `Forbid`**: if your job consistently takes longer than its schedule interval, it will never run. Every tick is skipped because the previous one is still running. You'll see events like "Cannot determine if job needs to be started." Monitor job duration vs. schedule interval.

---

## `startingDeadlineSeconds` — the missed-schedule recovery logic

This is the most nuanced field in CronJob.

When the controller checks "what Jobs need to be created?", it looks at the schedule and counts missed runs since the last successful schedule time. If > 100 runs were missed, **K8s stops trying to catch up and logs an error**.

`startingDeadlineSeconds` bounds the lookback window:

```yaml
spec:
  startingDeadlineSeconds: 300    # only catch up on schedules missed in the last 5 minutes
```

Without this field: the controller looks back to the very beginning (or last successful schedule), and if > 100 schedules were missed (e.g., cluster was down for a long time), it stops.

With `startingDeadlineSeconds: 300`: the controller only looks back 300 seconds. Misses older than 300 seconds are ignored. If > 100 schedules were missed within the last 300 seconds, it still stops — but this is an extremely edge case (100 schedules in 5 minutes means a sub-3-second cron schedule, which is unusual).

**Why does the 100-schedule cliff matter?** If your cluster control plane goes down for 2 hours and your CronJob runs every minute, that's 120 missed schedules. Without `startingDeadlineSeconds`, the CronJob controller gives up. When the cluster comes back, no Jobs are created for that CronJob. Setting `startingDeadlineSeconds: 300` means it only tries to catch up on the last 5 minutes of misses, creates at most ~5 Jobs, and then continues normally.

**Rule of thumb**: always set `startingDeadlineSeconds`. A value of 1-5 minutes is appropriate for most jobs.

---

## Timezone handling

Before K8s 1.27: CronJob schedules are always in UTC. If you write `0 2 * * *` (run at 2am), that's 2am UTC. For users in different timezones, this is a common source of confusion.

K8s 1.27 (stable in 1.27) added the `timeZone` field:

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "America/New_York"    # or "Europe/London", "Asia/Tokyo", etc.
```

Now the schedule fires at 2am New York time, accounting for DST automatically.

**Gotcha**: the timezone field requires the kube-controller-manager to have timezone data available (standard in most distributions). Verify your K8s version supports it before relying on it.

---

## Complete annotated YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 6 * * *"             # 6am UTC daily
  timeZone: "America/New_York"       # override: 6am Eastern
  concurrencyPolicy: Forbid          # don't overlap; skip if previous is running
  startingDeadlineSeconds: 300       # catch up on missed schedules within 5 min
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      completions: 1
      parallelism: 1
      backoffLimit: 3                # 3 pod failures before Job is Failed
      ttlSecondsAfterFinished: 7200  # keep pods/logs for 2h after completion
      template:
        spec:
          restartPolicy: Never       # use Never for debuggability
          containers:
          - name: reporter
            image: my-reporter:1.0
            env:
            - name: REPORT_DATE
              value: "$(date +%Y-%m-%d)"
            resources:
              requests:
                cpu: 200m
                memory: 256Mi
              limits:
                memory: 512Mi
```

---

## The interview answer in 60 seconds

> "When a CronJob schedule fires, it creates a Job object. The Job creates pods to run the work. If a pod fails (non-zero exit), the Job creates a replacement — that's not a Job failure, just a pod failure. A Job fails when total pod failures exceed `backoffLimit` (default 6). At that point, no more pods are created and the CronJob notes it as a failed run.
>
> The gotchas at scale: **First**, `concurrencyPolicy`. Default is `Allow` — two runs overlap if the previous is still running. For most batch jobs you want `Forbid`. **Second**, pod cleanup: completed pods accumulate forever without `ttlSecondsAfterFinished`. Set it or your cluster fills with ghost pods. **Third**, the missed-schedule cliff: if the control plane was down and more than 100 schedule intervals were missed, K8s stops trying to catch up. Set `startingDeadlineSeconds` to bound the lookback window. **Fourth**, timezone: before K8s 1.27, all cron schedules were UTC. Use the `timeZone` field on 1.27+ to specify the intended timezone explicitly."

---

## Self-test

### 1. What's the difference between a pod failure and a Job failure?

**Reference answer:** A pod failure is one pod exiting non-zero. The Job controller creates a replacement. A Job failure is when the count of pod failures reaches `backoffLimit` — the Job is marked Failed, no more pods created. Pod failures are expected retry events; Job failure is terminal. CronJob behavior: a failed Job is noted in history but the CronJob itself isn't failed — the next schedule still fires.

### 2. Your CronJob runs a report every 5 minutes. The report started taking 7 minutes. What happens?

**Reference answer:** With `concurrencyPolicy: Allow` (default): Jobs pile up — every 5 minutes a new Job starts while the old one is still running. Two concurrent runs, then three, then N. Fix: switch to `concurrencyPolicy: Forbid` (skip the tick if previous is running) or `Replace` (kill the old Job and start fresh). Also investigate why the report is taking longer — that's the real problem.

### 3. Your cluster was down for 3 hours. After recovery, what happens to CronJobs?

**Reference answer:** The CronJob controller checks for missed schedules. If the cron runs every minute and the cluster was down 3 hours, that's 180 missed schedules. Since 180 > 100, the controller stops trying to catch up and logs an error. Without `startingDeadlineSeconds`, the CronJob effectively misses the outage window entirely and resumes on the next regular schedule. With `startingDeadlineSeconds: 300`, only the last 5 minutes of misses are caught up on (5 Jobs created at most), then normal schedule resumes.

### 4. How do you clean up completed Job pods automatically?

**Reference answer:** Two mechanisms. `ttlSecondsAfterFinished` on the Job spec deletes the Job object and its pods after N seconds post-completion — good for routine cleanup. `successfulJobsHistoryLimit` on the CronJob spec controls how many finished Job objects the CronJob retains — old ones are garbage collected. Use both: `ttlSecondsAfterFinished` for the cleanup timing, `successfulJobsHistoryLimit` for keeping some history for debugging.

---

## Further reading / watching

- **Kubernetes Docs — Jobs**: https://kubernetes.io/docs/concepts/workloads/controllers/job/
- **Kubernetes Docs — CronJob**: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- **TTL After Finished Controller**: https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/
- **Indexed Jobs** (advanced pattern — each pod gets a unique completion index): https://kubernetes.io/docs/concepts/workloads/controllers/job/#completion-mode

---

## The 4 dimensions (senior framing)

- **Tech**: `completions` + `parallelism` + `backoffLimit` define the Job contract; `restartPolicy: Never` for debuggability; `ttlSecondsAfterFinished` for cleanup; CronJob's `concurrencyPolicy: Forbid` as the safe default; `startingDeadlineSeconds` to bound missed-schedule recovery; `timeZone` field on K8s 1.27+.
- **People**: Engineers often treat CronJobs like scheduled Deployments. Make sure the team understands `concurrencyPolicy` and `backoffLimit`. A CronJob with `Allow` concurrency and a growing job duration is a silent time bomb. Document this in the team's "CronJob checklist" and enforce via Kyverno.
- **CI/CD**: For database migrations or data pipeline triggers run as Jobs: gate the deploy pipeline on `kubectl wait --for=condition=complete job/<name>` with a timeout. Non-zero exit from `kubectl wait` means the Job failed — pipeline fails too. This is how you gate service deploys on successful migration Jobs.
- **Operations**: Alert on CronJob `LAST SCHEDULE` being older than 2× the schedule interval — indicates missed schedules or the `startingDeadlineSeconds` cliff. Alert on failed Jobs that weren't retried (`BackoffLimitExceeded`). Monitor pod TTL cleanup — clusters without TTL slowly accumulate thousands of completed pod objects (etcd bloat, slow `kubectl get pods` responses).
