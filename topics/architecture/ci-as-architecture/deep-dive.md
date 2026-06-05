# CI as architecture — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why is 'how code reaches prod' an architectural concern, not a tooling detail?"** — with the DORA evidence, the trunk-based argument, the immutable promotion principle, the feature flag architecture, and the GitOps inheritance.

> This topic links to the rejection-gap area. Start with the [simple version](./simple.md) if you haven't, then come back here.

---

## The senior framing — the pipeline is the architecture

The rejection feedback was explicit: "neglecting CI-related aspects." This is not about knowing which CI tools exist. It's about understanding that **how code reaches production is a first-class design decision**, with the same weight as database schema, API contracts, and service boundaries.

Why is it architectural?

1. **It determines feedback speed.** A pipeline that takes 45 minutes to run CI means a developer waits 45 minutes to know if their change is safe. That changes behavior: they batch more changes together to amortize the wait, which increases the blast radius of each deploy. The pipeline speed is a forcing function on team culture.

2. **It determines risk per change.** A weekly big-bang deploy carries the risk of 50 changes combined. A 10-times-daily deploy carries the risk of one change. The pipeline design determines which one you live with.

3. **It determines what your architecture can support.** A service that can't be deployed independently is not a microservice — it's a monolith with extra steps. The pipeline constraint exposes coupling you'd rather not see.

4. **It's expensive to change late.** Switching from bi-weekly deploys to continuous delivery requires changing branch strategy, testing approach, monitoring, rollback procedures, and team habits. The cost compounds with team size. Design it early.

> "Show me your deploy pipeline and I'll know your architecture."
>
> — common senior SRE aphorism

---

## The DORA evidence

DORA (DevOps Research and Assessment) ran multi-year surveys across thousands of organizations. The findings:

| Team tier | Deploy frequency | Lead time | MTTR | Change failure rate |
|---|---|---|---|---|
| Elite | On-demand (multiple/day) | < 1 hour | < 1 hour | 0-15% |
| High | 1-7 times/week | 1 day–1 week | < 1 day | 0-15% |
| Medium | 1-7 times/month | 1 week–1 month | 1 day–1 week | 0-15% |
| Low | Fewer than 1/6 months | 1-6 months | 1-6 months | 46-60% |

The counterintuitive finding: **elite performers deploy more frequently AND have lower failure rates**. More deploys doesn't mean more risk — smaller batches per deploy means lower risk per event, faster recovery when something goes wrong, and continuous feedback that catches problems earlier.

The pipeline design is the direct lever on "deploy frequency" and "lead time." Getting those right gets you to the elite cohort.

---

## Trunk-based development

### The problem with long-lived branches

Feature branches feel safe. The team works in parallel without stepping on each other. But:

- A branch that lives for 2 weeks accumulates divergence from `main`. The merge conflict is often large and hard.
- Integration testing doesn't happen until merge. You discover incompatibilities late, when they're expensive.
- The longer the branch lives, the more pressure to "just ship it" — which means less review, less validation, higher blast radius.

**The irony**: feature branches feel like they reduce risk (isolated development) but increase risk (late integration, big batches).

### Trunk-based in practice

Everyone commits to `main` (or merges short-lived branches of 1-3 days maximum). On every commit:
1. CI runs: build + test + lint + security scan
2. If CI passes, the commit is a candidate for promotion
3. If CI fails, the developer fixes immediately — the trunk is always in a deployable state

The trunk must always be deployable. This is the invariant. It's why short-lived branches and fast CI are load-bearing.

**For unfinished features:** feature flags. Code can be in main, in production, and in users' hands — with the flag off, no user sees the new behavior.

```yaml
# GitHub Actions — trunk-based CI example (simplified)
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: make test
      - name: Lint
        run: make lint
      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          exit-code: 1
          severity: CRITICAL

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/my-app:${{ github.sha }}
            ${{ env.REGISTRY }}/my-app:latest
```

The image is tagged with `github.sha` — the Git commit SHA. This is the immutable identity of the artifact.

---

## Immutable promotion

### The principle

**Build once. Promote the same artifact.**

```
Git commit (SHA: abc123)
    ↓ CI builds image
Image: my-app:abc123
    ↓ deploy to dev
Dev: running my-app:abc123
    ↓ tests pass in dev
Staging: running my-app:abc123
    ↓ tests pass in staging
Prod: running my-app:abc123
```

The same SHA runs in dev, staging, and prod. There's no rebuild between environments.

### Why immutable promotion matters

If you rebuild per environment, subtle differences creep in:
- Different build dependencies resolved at different times (npm lock file drift)
- Different `--build-arg ENV=staging` settings that change behavior
- Build tools update between the dev build and the prod build

With immutable promotion, the staging environment is a proof that the exact prod artifact is safe. Without it, staging proves that a similar-but-different artifact is safe.

### Per-environment configuration without rebuilding

"But the image needs to know which environment it's in!" The answer: **externalize configuration**. The image is environment-agnostic. Configuration is injected at runtime:

```yaml
# Kubernetes Deployment — environment-specific config via ConfigMap and Secret
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log_level
  - name: APP_ENV
    value: production   # the only env-specific value baked in — just a label
```

The image is the same. The ConfigMap and Secret are per-environment. This is the correct split.

### Image tagging strategy

| Tag | Use | Immutable? |
|---|---|---|
| `my-app:abc1234` (SHA) | CI-built artifact for a specific commit | Yes — never changes |
| `my-app:latest` | The most recent build | No — changes on every merge |
| `my-app:v1.2.3` (semver) | Release version | Yes if you treat tags as immutable |
| `my-app:staging` | What's currently in staging | No — a label that moves |

For production use, always promote by SHA or semver. Never use `:latest` in production Kubernetes manifests — you lose immutability guarantees.

---

## Feature flags — the architectural decoupling

### Why feature flags are architectural

Feature flags separate **deploy** (the artifact is in production) from **release** (users see the new behavior). This decoupling is a design decision that affects:

- How you write code (flag evaluation everywhere a new path diverges from old)
- What infrastructure you need (a flag service that's reliable and low-latency)
- What process you follow (flag lifecycle: add → test → gradual rollout → full release → flag cleanup)
- What your on-call story looks like (a flag can turn off a broken feature in seconds vs a full rollback in minutes)

If you don't design for feature flags from the start, adding them later means touching every place the new behavior diverges — expensive refactor.

### Feature flag infrastructure

Simple approach (suitable for small teams):
```python
# Database-backed flags — no external service
def is_enabled(flag_name: str, user_id: str) -> bool:
    flag = db.query("SELECT enabled, rollout_pct FROM flags WHERE name = %s", flag_name)
    if not flag or not flag.enabled:
        return False
    # Deterministic rollout: same user always gets same bucket
    user_bucket = int(hashlib.md5(f"{flag_name}:{user_id}".encode()).hexdigest(), 16) % 100
    return user_bucket < flag.rollout_pct
```

Production approach (high-traffic, high-reliability):
- **LaunchDarkly**: managed, SDKs for all languages, targeting rules, A/B testing, analytics
- **Unleash**: open-source, self-hosted, Kubernetes-native
- **AWS AppConfig**: managed, integrates with IAM, good for configuration + flag hybrid

**The gotcha**: the flag service is now in the critical path of every request. If the flag service goes down and your code doesn't handle it gracefully, you have a second outage. Flag SDKs typically cache the last known state and fall back to defaults — make sure you configure sensible defaults.

### Flag lifecycle — the debt problem

Every flag is a branch in your code. Flags that aren't cleaned up after full rollout become permanent dead weight:
- Code has two paths, only one of which runs
- New engineers don't know if the flag is permanent or temporary
- Tests need to cover both paths even when one is dead

The discipline:
1. **Add flag** with a cleanup date in the code comment: `# TODO: remove flag 'new-checkout' after 2026-07-01`
2. **Roll out** from 0% → 5% → 25% → 100% with monitoring at each step
3. **Clean up** — remove the flag evaluation and the old code path — within a defined window (typically 2-4 weeks after 100% rollout)

---

## GitOps — CI/CD architectural inheritance

GitOps applies the immutable promotion principle to cluster state: **the Git repository is the source of truth for what's deployed**. The cluster state is derived from the repo; no one runs `kubectl apply` by hand.

```
Code PR → CI → image built → image tag bumped in GitOps repo
                                        ↓
                              ArgoCD detects diff
                                        ↓
                           ArgoCD reconciles cluster
```

Two tools: **ArgoCD** (pull-model; ArgoCD watches the Git repo and applies) and **Flux** (similar pull model, more K8s-native). Both implement the same principle.

**Why GitOps is an architectural decision:**
- All cluster changes go through Git → every change has an author, a PR review, a code review
- Rollback is `git revert` followed by ArgoCD reconcile — the history of what ran in prod is the Git history
- Drift detection: ArgoCD alerts if the cluster state diverges from the repo (manual `kubectl apply` happened)
- Multiple clusters: manage prod, staging, and dev clusters from one repo with different overlays

```yaml
# ArgoCD Application — the GitOps source of truth
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-config
    targetRevision: HEAD
    path: k8s/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true      # delete resources not in Git
      selfHeal: true   # re-apply if someone manually changes the cluster
    syncOptions:
      - CreateNamespace=true
```

`selfHeal: true` is the anti-drift setting. If someone runs `kubectl scale deployment my-app --replicas=0` by hand, ArgoCD will restore it to the replica count in Git within seconds.

---

## The full pipeline in a senior-level design answer

When asked to design a CI/CD pipeline in an interview, walk this structure:

**1. Source → CI (every commit to main)**
- Tests run (unit, integration, contract)
- Security scan (Trivy for container image, SAST for code)
- Image built and tagged with commit SHA
- Image pushed to registry
- GitOps repo PR opened: bump image tag in the dev overlay

**2. Dev → Staging promotion**
- Automated: if dev SLO is green after 15 minutes at 100% traffic, open PR to bump staging overlay
- Review optional for staging; required for prod

**3. Staging → Prod promotion**
- Canary deploy (Argo Rollouts or traffic weighting): 5% → 25% → 100%
- At each step: SLO checks, error rate, latency p99
- Auto-rollback if any threshold breached
- Manual approval step optional at the 100% gate

**4. Feature flags**
- New features always behind a flag
- Flag defaults to off in all environments
- Rollout managed separately from deploy

**5. Rollback**
- `git revert` + ArgoCD reconcile = prod rollback in < 5 minutes
- Feature flag disabled = feature rollback in < 30 seconds (no deploy needed)

---

## The interview answer in 60 seconds

> "CI/CD is architectural because the pipeline design determines batch size, feedback speed, risk per change, and ultimately team performance. The DORA research is clear: elite teams deploy frequently with small batches and have lower change failure rates — not despite frequent deploys, but because of them.
>
> The three architectural decisions that matter most:
>
> First, **trunk-based development**: everyone commits to main, short-lived branches only. Integration is continuous and cheap. Feature flags handle unfinished work — code can be in prod with the feature off.
>
> Second, **immutable promotion**: build the artifact once in CI, tag it with the commit SHA, promote the same image through dev → staging → prod. Per-environment config is externalized (ConfigMap, Secret, ESO). If you rebuild per environment, staging doesn't prove anything about prod.
>
> Third, **feature flag infrastructure as a first-class concern**: design it before the first feature that needs it. A flag service that goes down is a second production incident. Build fallback defaults in from the start. Define a flag cleanup policy — unremoved flags are code debt.
>
> GitOps (ArgoCD or Flux) is the inheritance layer: cluster state is derived from Git, rollback is `git revert`, drift is detected and corrected automatically. It's what makes the pipeline auditable and resilient to manual changes.
>
> The 4-dimensions framing: Tech (pipeline stages, image tagging, ArgoCD). People (every change goes through a PR — reviewable, reversible). CI/CD (it's literally the topic — design it as the backbone, not as an afterthought). Operations (rollback in < 5 minutes via Git revert; feature flag rollback in < 30 seconds)."

---

## Self-test drills

### 1. Why is "how code reaches prod" an architectural concern, not a tooling detail?

**Reference answer:**
- Pipeline design determines batch size (more steps per deploy = smaller batches = less risk per change).
- Feedback speed determines developer behavior: slow CI → big batches → high blast radius. Fast CI → small batches → low blast radius.
- Pipeline constraints reveal architectural coupling. A service that can't be deployed independently isn't really a microservice.
- Expensive to change late: moving from biweekly deploys to continuous delivery requires changing branch strategy, testing, monitoring, rollback, and team habits — cheaper at design time.
- DORA evidence: elite performers deploy more frequently and have lower failure rates. The pipeline is the lever.

### 2. What's immutable promotion and why does rebuilding per environment break it?

**Reference answer:**
- Immutable promotion: build the Docker image once in CI, tag with commit SHA. Promote the same SHA from dev → staging → prod. Staging proves the exact prod artifact is safe.
- Rebuilding per environment: each environment gets a fresh build. Subtle differences can creep in (package lock drift, different build tool versions, `--build-arg ENV=staging` behavior). Staging tests prove a similar-but-different artifact is safe — not the prod artifact.
- Per-environment configuration without rebuilding: externalize it. ConfigMap, Kubernetes Secrets, ESO-synced values. The image is environment-agnostic; runtime config is injected.
- Image tagging: always use SHA or semver for production. Never `:latest` — you lose the guarantee that you know what's running.

### 3. What is the deploy/release distinction and why do feature flags implement it?

**Reference answer:**
- Deploy: the code is running in production. Release: users see the new behavior.
- Without the distinction, deploy = release. Every deploy is a binary event: everything changes or nothing does.
- Feature flags separate them. The code is deployed (in prod); the flag is off (no users see it). Gradual rollout: 1% → 5% → 25% → 100% with SLO checks at each step.
- Operational win: turning off a flag is < 30 seconds, no deploy needed. A rollback of a broken feature without flags takes 5-15 minutes. Flags make incident recovery faster.
- The architectural implication: the flag service must be designed before the first feature that uses it. It's in the critical path — if it goes down, the fallback must be a safe default, not a 500 error.

### 4. What's the senior-level problem with "we'll add CI improvements later"?

**Reference answer:**
- Adding a canary deploy to a service that was designed for big-bang deploys requires: stateless service design, readiness probe tuning, PDB configuration, monitoring that distinguishes canary from stable, rollback automation. These aren't bolted on; they're designed in.
- Same for feature flags: adding them to code that wasn't written with flag evaluation in mind means touching every divergence point — expensive refactor.
- Same for trunk-based development: a team that's been on Git-flow for 2 years needs to change habits, resolve the backlog of long-lived branches, rebuild CI checks, and retrain muscle memory. Days of work at minimum.
- The senior pattern: "we'll add it later" is YAGNI misapplied. CI/CD design is not a hypothetical future requirement — it's the mechanism by which every future feature ships. It's load-bearing from day one.

---

## Further reading / watching

- **DORA — *Accelerate* (Forsgren, Humble, Kim)**: the empirical foundation. Chapter 2 (metrics) and Chapter 4 (technical practices) are the relevant sections. Short book.
- **Jez Humble and David Farley — *Continuous Delivery***: the original treatment of pipeline stages, immutable artifacts, and the "build once, promote" principle. Dense but authoritative.
- **Martin Fowler — "Trunk Based Development"** at martinfowler.com: the canonical definition. Short, worth reading.
- **LaunchDarkly blog on feature flag best practices**: the most practical resource on flag architecture, rollout strategies, and flag debt.
- **ArgoCD docs — "Getting Started"**: the fastest way to understand GitOps in practice. The Application CRD and sync policies are worth understanding well.
- **Google SRE Book** — Chapter 8: Release Engineering. The Google-scale version of immutable promotion and release management.

---

## The 4 dimensions (senior framing)

This topic is unusual in that it's directly about CI/CD — so all four dimensions are about CI/CD, viewed from different lenses:

- **Tech**: Trunk-based development (everyone commits to main, short-lived branches). Immutable promotion (build once, SHA-tagged image, promote by bumping the tag in the GitOps repo). Feature flags (flag service in critical path — design fallback defaults; flag lifecycle — add → rollout → clean up). GitOps (ArgoCD/Flux, selfHeal=true for anti-drift). Canary deploys via Argo Rollouts (5% → 25% → 100%, auto-rollback on SLO breach).

- **People**: Every change goes through a PR — reviewable, attributable, reversible. Short-lived branches force frequent integration, which surfaces incompatibilities while they're cheap to fix. Feature flags let product and engineering separate "ship the code" from "turn on the feature" — different decisions, different stakeholders. Flag toggles are a product decision; code promotion is an engineering decision.

- **CI/CD**: This is the topic. The pipeline is the architecture. Design it with: CI stage (< 10 minutes; build + test + scan + push), dev promotion (automated), staging promotion (automated with optional manual gate), prod promotion (canary with auto-rollback). Rollback strategy: `git revert` + ArgoCD reconcile for full rollback; flag disable for feature-level rollback. Policy gates: Trivy CRITICAL scan, OPA/Kyverno admission, SBOM generation.

- **Operations**: Deployment frequency and lead time are team health metrics — track them in your DORA dashboard. SLO checks at each canary step automate the promotion/rollback decision. Alert on GitOps drift (`argocd_app_info{sync_status="OutOfSync"}`). Runbook for "ArgoCD not syncing": check application status in ArgoCD UI, check for webhook delivery failures, check for Kubernetes admission failures (Kyverno policy blocking the manifest). Monitor flag service error rate separately — a flag service outage degrades to the flag default, which may mean all features are off or all are on, depending on your defaults.
