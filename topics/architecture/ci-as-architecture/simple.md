# CI as architecture — the simple version (the stamp-on-the-parcel analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes obvious.

This doc only explains **one idea**:

> **How code reaches production is an architectural decision, not a tooling detail. The shape of your deploy pipeline tells you the shape of your team and your risk tolerance.**

That's it. Everything else (trunk-based development, immutable promotion, feature flags, GitOps) is precision on top.

---

## The stamp-on-the-parcel analogy

You work at a post office. A parcel comes in. It needs to be stamped — that verifies it's legit — and then delivered.

Two approaches:

**Approach 1**: Every worker stamps their own parcel with their own stamp. Each worker uses slightly different ink. When a parcel goes to the wrong address, you spend two days figuring out which stamp was used and whether the ink was right.

**Approach 2**: One machine stamps parcels at intake. Once stamped, the stamp doesn't change. Every delivery person carries the same stamped parcel from that point on. You always know exactly what was verified and when.

| Post office | Software delivery |
|---|---|
| The parcel | The code you want to ship |
| The stamp | The CI build: tests passed, image built, security scanned |
| Approach 1 (per-worker stamp) | Re-building the artifact for each environment: dev build, staging build, prod build |
| Approach 2 (stamp-at-intake) | **Immutable promotion**: build once, promote the same image through dev → staging → prod |

Approach 2 is the correct answer. With Approach 1, you can't be sure the parcel going to prod is the same one that was tested. With immutable promotion, you're certain.

---

## The one insight that makes CI "architectural"

> **"Deploy" and "release" are two different things. Senior engineers separate them.**

**Deploy**: the artifact is running in production.
**Release**: users can see the new behavior.

A feature flag lets you deploy without releasing. The code is in production (deploy), but the feature is off (not released). You can turn it on for 1% of users, validate it, then gradually expand. You can turn it off instantly if something breaks — no rollback required.

This separation is an **architectural decision**. You bake feature flags into the code. You build the flag evaluation infrastructure. You agree on the process for toggling flags. All of this is design work that happens before a single line of feature code is written.

Teams that don't make this decision end up with "deploy = release" — they can't ship code that isn't fully tested, they can't do gradual rollouts, and every deploy is a binary risk event.

---

## The three concepts you need to know

### 1. Trunk-based development

Everyone commits to `main` (the "trunk"). No long-lived feature branches. Small, frequent commits — ideally multiple times per day per developer.

Why it's the senior choice: long-lived branches accumulate divergence. When Branch-A finally merges after 2 weeks, it conflicts with everything that happened in main. Integration is hard and late. Trunk-based development makes integration continuous and cheap.

The concern: "what about unfinished features?" → Feature flags. Code can be in main and in production without being visible to users.

### 2. Immutable promotion

Build the artifact **once** at the CI step. That Docker image (with its specific SHA tag) gets promoted through environments. Dev runs SHA `abc123`. Staging runs SHA `abc123`. Prod runs SHA `abc123`. Same bits, all the way.

The wrong pattern: rebuild the image for each environment with `--build-arg ENV=staging`. Now staging and prod aren't running the same thing. Tests in staging might pass because the staging build is subtly different from the prod build.

### 3. Feature flags

Code is deployed but behavior is toggled separately. The flag evaluation is usually done in a service like LaunchDarkly, Unleash, or a simple database column.

```python
# The code is in production; the feature is off until the flag is enabled
if feature_flags.is_enabled("new-checkout-flow", user_id):
    return new_checkout_flow(request)
else:
    return old_checkout_flow(request)
```

Feature flags decouple "can we ship this?" (is the code ready?) from "should we expose this?" (is the rollout strategy ready?). The former is a tech question. The latter is a business and risk question.

---

## The senior framing in one sentence

> "Show me your deploy pipeline and I'll know your architecture."

A team that deploys once a week has long-lived branches, big batches, and high-risk deploys. A team that deploys 50 times a day has trunk-based development, small changes, feature flags, and automated rollback. The pipeline shape is the architecture.

DORA (the DevOps Research and Assessment group) measured this across thousands of teams: high-performing teams deploy more frequently, have shorter lead times, lower failure rates, and faster recovery. The pipeline is not a detail — it's the thing that determines how the team performs.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's trunk-based development? | Everyone commits to `main`. No long-lived branches. Small batches, frequent. |
| Why trunk-based? | Integration is continuous, not a big-bang merge event. Small batches = smaller blast radius per deploy. |
| What's immutable promotion? | Build the artifact once in CI. Promote the same image (same SHA) from dev → staging → prod. |
| Why immutable promotion? | If staging ran a different build than prod, staging tests don't prove anything about prod. |
| What's the deploy/release distinction? | Deploy = code in production. Release = users can see the feature. Feature flags separate them. |
| What's GitOps? | The Git repository is the source of truth for what's deployed. No `kubectl apply` by hand — changes to the cluster go through a PR. ArgoCD or Flux watches the repo and reconciles. |
| Why is CI "architectural"? | The pipeline shape determines batch size, feedback speed, risk per deploy, and team velocity. Designed late = expensive to change. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Why is 'how code reaches prod' an architectural concern, not a tooling detail?"**

**Reference answer (intuitive version):**

"Because the pipeline determines how fast the team can learn and how much risk each change carries. A team that rebuilds the image per environment can't be sure prod and staging are running the same thing — that's a correctness concern. A team without trunk-based development integrates in big batches — that's a feedback latency concern. A team without feature flags can't separate deploy from release — that's a risk management concern.

These aren't tooling choices you make after the architecture is done. They're constraints that shape the architecture: 'we'll use feature flags' means the feature flag infrastructure needs to be designed, deployed, and reliable before the first feature that uses it. 'We'll do immutable promotion' means the CI pipeline is part of the design spec, not an afterthought.

The DORA research confirmed this empirically: high-performing teams have architectures that enable small-batch, high-frequency deploys. The pipeline is the architecture. Designing the architecture without designing the pipeline is designing half the system."

If that came out clearly, jump to the [deep-dive](./deep-dive.md) for the DORA metrics, trunk-based development in practice, the GitOps pattern, feature flag infrastructure, and the 4-dimensions treatment.

---

## Further reading / watching

- **DORA — *Accelerate* (book by Forsgren, Humble, Kim)**: the data behind what high-performing teams actually do. The four DORA metrics (deployment frequency, lead time, MTTR, change failure rate) are the measurement framework for pipeline quality.
- **Martin Fowler — "Trunk Based Development"**: search the title at martinfowler.com. The canonical definition and practice guide.
- **Jez Humble — *Continuous Delivery* (book)**: the original treatment of immutable artifacts, pipeline stages, and "build once, promote."
- **LaunchDarkly blog**: the best practical resource on feature flag architecture and the deploy/release distinction.

---

## Next: the deep-dive

When the stamp analogy, trunk-based development, and the deploy/release distinction feel obvious, jump to [`ci-as-architecture.md`](./deep-dive.md). The deep-dive covers:

- The DORA metrics and what they measure
- Trunk-based development in practice: short-lived branches, pair commits, feature flag as the escape valve
- Immutable promotion: the pipeline stages, how image tags work, how to handle per-env config without rebuilding
- Feature flags: the infrastructure, the flag lifecycle (add → toggle → clean up), the risk of flag debt
- GitOps: ArgoCD vs Flux, the single-source-of-truth model, the drift detection loop
- The 4-dimensions framing: CI as People + Tech + Operations concern
- Self-test drills
