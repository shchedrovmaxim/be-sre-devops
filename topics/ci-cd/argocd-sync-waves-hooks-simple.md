# ArgoCD sync waves + hooks — the simple version (the construction sequence)

> Read this first. Once the analogy clicks, the [deep-dive](./argocd-sync-waves-hooks.md) is easy.

This doc explains **one idea**:

> **Sync waves control the order ArgoCD deploys resources. Hooks let you run one-off jobs at specific phases. Together, they let you run a DB migration before your app starts.**

---

## Building a house — the order matters

You can't put the roof on before the walls. You can't paint before the drywall. Some things must come first.

In Kubernetes, the same applies:
- Run DB migration **before** the new app version starts (or old pods serving requests will query the wrong schema)
- Verify the migration succeeded **before** routing traffic to new pods
- Clean up after a rollout **only if** everything succeeded

ArgoCD sync waves + hooks are the tool for encoding this construction sequence.

---

## Sync waves — numbered ordering

Every resource can have a wave annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # low numbers first
```

ArgoCD deploys resources in wave order: wave -2, then -1, then 0 (default), then 1, then 2, etc. Within a wave, everything is applied in parallel.

Mental model: wave = floor of the building. Foundation (wave -2) → walls (wave 0) → roof (wave 2). Each "floor" is fully stable before the next one starts.

---

## Sync hooks — phase-specific jobs

Beyond wave order, ArgoCD defines sync phases. You can annotate a resource as a **hook** to run it only in a specific phase:

| Phase | When it runs |
|---|---|
| `PreSync` | Before the main sync starts |
| `Sync` | During the main sync (default — most resources) |
| `PostSync` | After the sync completes successfully |
| `SyncFail` | Only if the sync failed (cleanup, alerts) |

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

`hook-delete-policy` controls cleanup:
- `HookSucceeded`: delete the hook resource if it succeeded (keeps cluster clean)
- `HookFailed`: delete if it failed
- `BeforeHookCreation`: delete the old hook before running a new one (useful for idempotent jobs)

---

## The canonical migration pattern

```
PreSync Job: run `rails db:migrate` (or `alembic upgrade head`, or `flyway migrate`)
     ↓ succeeds (ArgoCD waits)
Sync: deploy new Deployment / Service / ConfigMap
     ↓ all healthy
PostSync: run smoke tests, send a Slack notification, warm caches
```

If the PreSync job fails, ArgoCD stops and does not deploy the new app version. The old version keeps running against the old schema. Safe.

---

## The idempotency requirement

Hooks run on **every sync**. If your migration script runs on every deploy (even when there's nothing to migrate), it must be idempotent — running it twice in a row must produce the same result as running it once.

For DB migrations: most migration tools (Flyway, Alembic, Liquibase) are idempotent by design — they track which migrations have already been applied. A no-op run is fine.

For custom scripts: add an explicit check at the top: "has this already been done? If yes, exit 0."

---

## Self-test (the killer question)

Out loud:

> **"I need a DB migration to run before my app rollout. Walk me through using ArgoCD sync waves and hooks for this."**

**Reference answer (intuitive version):**

"I'd create a Kubernetes Job with two annotations: `argocd.argoproj.io/hook: PreSync` and `argocd.argoproj.io/hook-delete-policy: HookSucceeded`. The job runs my migration script. ArgoCD runs it before touching any other resource in the sync. If the job fails — the migration failed — ArgoCD stops the sync there. The old Deployment keeps running against the old schema. Nothing breaks. If the job succeeds, ArgoCD proceeds with the Sync phase: applies the new Deployment, ConfigMap, etc. After everything is healthy, PostSync hooks can run smoke tests or send notifications.

The thing I'd verify: the migration script must be idempotent. On the next deploy that doesn't change the schema, ArgoCD runs the PreSync job again. It must safely do nothing when there's nothing to migrate — most migration tools like Flyway or Alembic handle this automatically."

---

## Further reading

- [ArgoCD sync waves docs](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [ArgoCD resource hooks docs](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)

---

## Next: the deep-dive

When the construction-sequence analogy feels obvious, jump to [`argocd-sync-waves-hooks.md`](./argocd-sync-waves-hooks.md). The deep-dive covers:

- Wave numbering mechanics (integers, negative waves, within-wave ordering)
- All four hook phases (PreSync, Sync, PostSync, SyncFail)
- Hook deletion policies in depth
- The complete migration pattern with YAML
- Idempotency requirements and how to enforce them
- Common pitfalls (hooks that re-run forever, hooks that block rollback)
- 3 self-test drills + 4-dimensions framing
