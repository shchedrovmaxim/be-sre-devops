# ArgoCD sync waves + hooks — the deep-dive

> **Goal**: by the end you can answer the killer question — **"I need a DB migration to run before my app rollout. Walk me through using ArgoCD sync waves and hooks for this."** — naming the four hook phases, deletion policies, idempotency requirements, and the complete migration pattern.

> Start with [argocd-sync-waves-hooks-simple.md](./simple.md) if you haven't.

---

## The senior framing — sequencing is a correctness requirement

Mid-level: "we use PreSync hooks for migrations."
Senior: "every hook is idempotent, has a deletion policy, and the sync failure path has been designed. A PreSync hook that fails must leave the cluster in a state that allows rollback — which means the migration must be backward-compatible with the previous app version."

The interview moment is explaining the full failure path, not just the happy path.

---

## Sync wave mechanics

Every Kubernetes resource managed by ArgoCD can carry a wave annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # string, parsed as integer
```

Wave values:
- Default (no annotation): wave 0
- Any integer: negative waves first, then positive
- Resources with the same wave are synced in parallel

ArgoCD evaluates waves using this algorithm:

1. Group all resources by wave number.
2. Apply wave N resources. Wait for all of them to be healthy.
3. If wave N is healthy, apply wave N+1. If not, stop sync (fail).
4. Repeat until all waves are applied.

"Healthy" depends on resource type: Deployment = all pods ready; Job = completed successfully; Service = endpoints populated; etc.

### Practical wave assignment

```
Wave -2: CRDs (must exist before other resources reference them)
Wave -1: Namespaces, ConfigMaps, Secrets (infrastructure that resources depend on)
Wave  0: Core workloads (Deployments, StatefulSets, Services) [default]
Wave  1: Ingress (routes traffic only after backend Services are ready)
Wave  2: HPA (scale based on the Deployment that's now running)
```

For a migration pattern:

```
Wave -1: PersistentVolumeClaim (storage must exist before migration)
Wave  0: PreSync hook Job (migration)
Wave  1: Deployment (new app version)
Wave  2: HPA
```

Wait — but PreSync hooks run *before* the Sync phase, not in a wave. The wave system and hook system operate on different axes. Let's clarify.

---

## Hooks vs waves — two axes

These are complementary, not alternatives:

- **Waves** control the order within the **Sync phase** (when regular resources are applied).
- **Hooks** run outside the normal sync flow — in special phases before or after the Sync phase.

```
Timeline of a full sync:

  [PreSync hooks] → [Sync phase (waves 0, 1, 2...)] → [PostSync hooks]
                                ↓ (only on failure)
                          [SyncFail hooks]
```

A `PreSync` hook runs before any waves. A `PostSync` hook runs after all waves are healthy. You can combine them: a wave 0 resource (the Deployment) with a PostSync hook Job that runs smoke tests after the Deployment is healthy.

---

## Hook phases in detail

### PreSync

Runs before the Sync phase. The rest of the sync waits for all PreSync hooks to complete successfully.

Use for:
- DB migrations
- Batch jobs that must complete before the new version starts (cache warming, data transformation)
- Validation steps (check the cluster is healthy before deploying)

Failure behavior: if any PreSync hook fails, ArgoCD marks the sync as `Failed` and does not proceed to the Sync phase. Existing resources remain at their previous state.

### Sync

Resources annotated with `hook: Sync` are applied during the Sync phase alongside regular resources. Rarely needed — most resources don't need the hook annotation, they're just in the sync by default.

Use for: one-off Jobs that should run in parallel with the Deployment update (rare).

### PostSync

Runs after the Sync phase completes and all resources are healthy.

Use for:
- Smoke tests / integration test jobs
- Cache warming
- Slack/PagerDuty notifications
- Post-deploy cleanup scripts

PostSync hooks run in parallel (unless you use wave annotations within PostSync hooks — yes, hooks can also have waves).

### SyncFail

Runs **only if the sync failed** (any phase failure). Does not run on success.

Use for:
- Alert notifications ("deploy of my-app failed")
- Rollback scripts
- Cleanup (remove partial state left by a failed sync)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-sync-failure
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookFailed
spec:
  template:
    spec:
      containers:
        - name: notify
          image: curlimages/curl
          command:
            - curl
            - -X POST
            - https://hooks.slack.com/...
            - -d
            - '{"text": "Sync failed for my-app!"}'
      restartPolicy: Never
```

---

## Hook deletion policies

Every hook should specify a deletion policy. Without it, the hook Job/Pod remains in the cluster after the sync — cluttering namespaces and confusing operators.

| Policy | When the hook resource is deleted |
|---|---|
| `HookSucceeded` | Immediately after the hook succeeds |
| `HookFailed` | Immediately after the hook fails |
| `BeforeHookCreation` | Before the hook is created on the next sync run (default if none set) |

Best practice:
- Migration jobs: `HookSucceeded` (clean up successful jobs; keep failed jobs for debugging)
- Alert jobs (SyncFail): `HookFailed` (the SyncFail hook itself won't fail cleanly; use `BeforeHookCreation`)
- Smoke test jobs (PostSync): `HookSucceeded`

To keep failed hooks for debugging while still cleaning up eventually:

```yaml
annotations:
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: HookSucceeded
  # failed hooks remain for manual inspection; cleaned up on the next sync attempt via BeforeHookCreation
```

Note: if you don't add `HookFailed`, a failed hook Job stays in the namespace until you manually delete it or the next sync creates a new one (via `BeforeHookCreation`).

---

## The canonical migration pattern — full YAML

```yaml
# migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-app-db-migrate
  namespace: my-app-production
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  ttlSecondsAfterFinished: 300   # K8s will also clean up after 5 min (belt + suspenders)
  backoffLimit: 0                # don't retry on failure — let ArgoCD handle the failure
  template:
    metadata:
      labels:
        app: my-app-db-migrate
    spec:
      restartPolicy: Never
      serviceAccountName: my-app-migrator   # scoped SA with DB access only
      containers:
        - name: migrate
          image: registry.example.com/my-app:{{ .Values.image.tag }}
          command: ["python", "manage.py", "migrate", "--no-input"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: my-app-db-secret
                  key: url
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

Key design choices:
- `backoffLimit: 0`: no K8s-level retries. If the migration fails, fail immediately and let ArgoCD surface the failure. Retrying a partially-applied migration can cause corruption.
- `restartPolicy: Never`: for the same reason.
- `ttlSecondsAfterFinished: 300`: belt-and-suspenders cleanup from K8s's own Job TTL controller.
- Same image as the app: the migration code is in the same image, so it's always compatible with the schema changes the new Deployment will rely on.
- Scoped service account: the migration job doesn't need cluster-wide permissions, just DB access.

---

## Idempotency — the most common mistake

Hooks run on **every sync**. Even if nothing changed in the sync, if you trigger a manual sync, the PreSync hook runs. If you push an unrelated change (fix a typo in a ConfigMap), the PreSync hook runs.

This is correct behavior — ArgoCD doesn't know which changes require a migration. So the migration script must be safe to run when there's nothing to migrate.

For migration tools: Flyway, Alembic, Liquibase, golang-migrate, TypeORM — all maintain a migrations table. Running `migrate` when already up-to-date is a no-op. Built-in idempotency.

For custom scripts:

```bash
#!/bin/bash
set -euo pipefail

# Check if this migration has already been applied
APPLIED=$(psql $DATABASE_URL -tAc "SELECT EXISTS(SELECT 1 FROM migrations WHERE id='2026-06-05-add-user-index')")
if [ "$APPLIED" = "t" ]; then
  echo "Migration already applied, skipping"
  exit 0
fi

# Run the migration
psql $DATABASE_URL < /migrations/2026-06-05-add-user-index.sql

# Record it
psql $DATABASE_URL -c "INSERT INTO migrations (id) VALUES ('2026-06-05-add-user-index')"
```

---

## The backward compatibility requirement — the deeper gotcha

If the PreSync migration runs and the new Deployment is rolling out, there's a window where **both old and new pods are running simultaneously** (during the rolling update). During this window:

- Old pods are running against the new schema.
- New pods are running against the new schema.

The migration must be backward-compatible with the old application version for this window to be safe. Classic patterns:

1. **Expand-contract**: first migration adds columns/tables but doesn't remove old ones. Old app works. New app uses the new columns. Second migration (next deploy) drops the old columns.
2. **Never drop columns in the same deploy as code changes**: add in one deploy, remove after all old instances are gone.
3. **Avoid renaming**: rename = add new + copy data + update references + drop old, across multiple deploys.

This is the senior answer beyond "use PreSync hook" — the migration strategy must account for rolling updates.

---

## The interview answer in 60 seconds

> "For a DB migration before rollout, I'd create a Kubernetes Job with `argocd.argoproj.io/hook: PreSync` and `hook-delete-policy: HookSucceeded`. ArgoCD runs this job before touching any other resource. If the job fails, the sync stops — the existing Deployment keeps running against the old schema, nothing breaks.
>
> If the job succeeds, ArgoCD proceeds with the Sync phase: applies the new Deployment, which rolls out with the new schema ready.
>
> The idempotency requirement: this job runs on every sync. The migration tool (Flyway, Alembic, etc.) must be safe to run when there's nothing to migrate — all standard tools handle this. For custom scripts, add an explicit 'already applied?' check at the top.
>
> The deeper point I'd raise: during the rolling update, old and new pods run simultaneously. The migration must be backward-compatible with the old app version. That means expand-contract: add new columns first, never drop in the same deploy as the code change. That's where the real safety lives."

---

## Self-test drills

### 1. What's the difference between sync waves and sync hooks?

**Reference answer:**
- **Waves** control the order of resources within the Sync phase. Wave 0 resources apply first, then wave 1, etc. Each wave must be healthy before the next starts.
- **Hooks** run in separate phases outside the Sync phase: PreSync (before), PostSync (after success), SyncFail (on failure). Hooks are separate from waves.
- You can combine them: a PreSync hook can have a wave annotation to sequence multiple PreSync hooks.
- The practical framing: waves for infrastructure ordering (CRDs before resources, Services before Ingress). Hooks for operational one-offs (migrations, smoke tests, notifications).

### 2. Why is `backoffLimit: 0` the right choice for a migration Job?

**Reference answer:**
- K8s-level retries on a failed migration can make things worse. If the migration partially applied and then failed, running it again may attempt to re-apply already-applied steps.
- With `backoffLimit: 0`, a failure → Job fails → ArgoCD sees a failed PreSync hook → sync is marked Failed. The failure is surfaced immediately.
- Migration tools with proper idempotency (Flyway's checksum validation, for example) might survive a retry, but it's safer to fail fast and require human review.
- Separate concern: K8s's `ttlSecondsAfterFinished` handles cleanup. Don't conflate "retry" with "cleanup."

### 3. What happens if your PreSync migration job fails halfway through?

**Reference answer:**
- ArgoCD marks the sync as `SyncFailed`. The Sync and PostSync phases don't run.
- The existing Deployment remains at its current version against the pre-migration schema. No inconsistency — nothing changed.
- The failed Job remains in the cluster (because `HookFailed` isn't set) for debugging. You can `kubectl logs` to see what failed.
- Recovery: fix the migration script, commit, push. The next sync triggers the PreSync job again. If the migration is idempotent, it resumes from where it can safely resume.
- The SyncFail hook (if configured) fires an alert so on-call knows immediately.

---

## Further reading

- [ArgoCD resource hooks docs](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)
- [ArgoCD sync waves docs](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Expand-contract migration pattern](https://www.martinfowler.com/bliki/ParallelChange.html) — Martin Fowler's "parallel change" — the backward-compatibility pattern for zero-downtime schema changes

---

## The 4 dimensions (senior framing)

- **Tech**: PreSync hook = Job with `argocd.argoproj.io/hook: PreSync` and `hook-delete-policy: HookSucceeded`. `backoffLimit: 0`. Idempotent migration tool. PostSync for smoke tests. SyncFail for alerts. Waves for resource ordering within Sync phase.
- **People**: explain the expand-contract pattern to app developers. They often write migrations that drop columns in the same PR as the code change. The CI/CD system can't enforce this, but the deployment runbook can — require a migration review for any schema change. A "migration checklist" in the PR template (new column? yes/no; backward compatible? yes/no) surfaces the problem at code review time.
- **CI/CD**: run `argocd app diff` in CI to preview what a sync would do before merging. Add a "dry run" step that triggers a sync with `--dry-run` and shows the diff. This catches waves-and-hooks configuration errors before they hit production.
- **Operations**: monitor PreSync job duration as a deploy-time SLI — migrations that take longer than X minutes should page, because they're blocking the rollout. Set a timeout on the Job (`activeDeadlineSeconds`) so a hung migration doesn't block ArgoCD indefinitely. If a sync is stuck on a PreSync hook, `kubectl delete job my-app-db-migrate -n my-app-production` unblocks it (then investigate separately).
