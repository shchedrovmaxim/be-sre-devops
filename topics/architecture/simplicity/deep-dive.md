# Simplicity, complexity, and the senior architectural mindset

> **Goal**: by the end you can answer **any "design X" interview question** with the 4-dimensions framework (Tech + People + CI/CD + Operations) and consistently propose the simplest viable approach first, naming the cost of alternatives.

> This is **the** rejection-gap topic. The interviewer flagged: "the proposed solution lacked simplicity, neglecting the human dimension (team training, onboarding) and CI-related aspects."

> Start with the [simple version](./simple.md) if you haven't read it — the moat-instead-of-a-fence analogy is the spine of everything here.

---

## The senior failure mode is over-engineering

Junior engineers fail by under-engineering — missing edge cases, no error handling, no monitoring. They learn to add things.

Senior engineers fail by **over-engineering** — adding things that aren't justified, building for hypothetical futures, picking the most general solution. They have to learn to *not* add things.

The discipline is **simplicity as the default, complexity only with justification.** Every component, every abstraction, every service you add has an *ongoing* cost: someone has to learn it, operate it, debug it at 2 AM, document it, upgrade it, pay for it. You don't pay that cost once at build time — you pay it forever.

The rejection feedback you got translates to: **"the candidate optimized for the wrong axis — they thought a more elaborate architecture meant a better one. It didn't."**

---

## KISS — Keep It Stupid Simple

The original (Kelly Johnson, Lockheed): designs should be **so simple a stupid person could repair them in the field with a wrench**. Not "for stupid people" — for *people in stupid situations*. 3 AM, broken laptop, exhausted, paged, in a hotel.

KISS misuse: "It means use the simplest tool." → Wrong. Simple ≠ "small." Simple ≠ "few features." Simple means **few moving parts, few abstractions, few things that can independently fail**, *while still solving the actual problem*.

A monolith can be simpler than microservices. A SQL database can be simpler than DynamoDB. A bash script can be simpler than a Go binary. "Simpler" is measured in **operational and cognitive cost**, not lines of code.

## YAGNI — You Aren't Gonna Need It

The original (Kent Beck, XP): don't build features until you actually need them. Two reasons:

1. Most features speculated about in advance never get used.
2. The features you do need later are usually different from what you predicted.

YAGNI misuse: "It means don't think ahead." → Wrong. It means: *don't build the abstraction* until the abstraction has a concrete second user. Thinking about the future is fine; building for it is not.

The canonical YAGNI violation:

> "We need to wrap this in an interface so we can swap implementations later."

If there is no second implementation right now, there's no interface needed. The interface costs ongoing cognitive load (every reader has to mentally indirect). Add it *when* you have a second implementation, not before.

---

## Ousterhout's "complexity" — what we actually mean

John Ousterhout (Stanford) defines complexity operationally: **anything related to the structure of a system that makes it hard to understand or modify.** Not "hard," not "advanced" — **hard to understand or modify**.

He names three symptoms of complexity:

### 1. Change amplification

A small functional change requires modifications in many places. If adding a new user role requires editing 6 files, the system has change-amplification complexity.

### 2. Cognitive load

A developer needs to hold a lot in their head to make a change safely. If understanding why a function is the way it is requires reading 4 other files and a wiki page, cognitive load is the cost.

### 3. Unknown unknowns

It's not obvious what you need to know to make a change. You make a change that *looks* fine, and 3 weeks later production breaks for a reason that turns out to be related to the change in a way nobody could have predicted. This is the worst kind — the other two you can at least feel.

**Senior framing**: when arguing for simplicity, name the *symptom* the complexity will cause. Don't say "this is too complex" — say "this adds change amplification" or "this adds an unknown unknown around the failure mode."

---

## The 3 sources of complexity

Not all complexity is created equal. Three sources, ranked by how acceptable they are:

| Source | Definition | Example | Acceptable? |
|---|---|---|---|
| **Essential** | Inherent to the problem. You can't make it simpler without solving a different problem. | Distributed consensus is hard. Period. | Yes — necessary cost of solving the actual problem. |
| **Accidental** | Imposed by tooling, history, conventions, or organizational structure. | "We use SOAP because the team did in 2010." | Reducible — work to remove over time. |
| **Designed-in** | Added by the architect because they thought it was elegant / general / future-proof. | "We built a plugin system in case we want to support other DBs." | Almost never acceptable — this is what we're calling over-engineering. |

The senior discipline: **maximize essential, reduce accidental, refuse designed-in.**

When you hear yourself or a teammate say "we'll need this later" or "this gives us flexibility" or "what about scale" — that's the alarm bell for designed-in complexity. The rebuttal is always: **show me the concrete current requirement that needs it.**

---

## The decision framework — "is this complexity justified?"

When evaluating whether to add a component, an abstraction, a new tool, or any net-new complexity, walk these 4 questions:

1. **What concrete current requirement does it solve?** (Not hypothetical. Today.)
2. **What's the simplest viable alternative?** (And does it actually solve the requirement, or is it just smaller?)
3. **Who pays the ongoing cost?** (Operations, on-call, future engineers, security review, compliance.)
4. **What's the exit cost if we're wrong?** (Can we rip this out in a sprint, or is it now load-bearing?)

If you can't answer #1 cleanly with a specific current problem → don't add it.
If #2 produces an alternative that's "good enough" → use it; complexity-add fails.
If #3 names people who didn't ask for this → reconsider.
If #4 is "load-bearing forever" → very high bar for adding.

This is the framework you use *out loud* in an interview. It demonstrates the senior mindset more than any specific technical answer.

---

## The 4 dimensions framework — architecture's actual deliverable

This is the thing the rejection feedback called out by name. **Any senior architectural answer must address 4 dimensions:**

### Tech (the part juniors handle well)
- What components, what data flow, what interfaces
- What database, what message bus, what cache
- How failures cascade or don't
- Scalability ceiling and how we approach it

### People (the dimension the rejection flagged)
- Who runs this? Who's on-call?
- How does a new engineer get up to speed? **In <1 week** is the right bar.
- What's the cognitive load — is it understandable to someone who didn't build it?
- What training, documentation, pairing investment is needed?

### CI/CD (the dimension the rejection flagged)
- How does code reach prod? Trunk-based? Feature-flagged? Canary?
- How do we roll back if we have to?
- How do we promote across environments — immutable artifacts? Re-build per env? (Senior answer is always: build once, promote immutably.)
- What policy gates are in place — Wiz scans, Kyverno admission, SBOMs?

### Operations
- What are the SLOs? SLIs? Error budgets?
- What monitoring proves the system is working — not just up, but *correct*?
- What runbooks exist? Are they tested?
- What's the incident-response flow? Who pages who?
- What happens during a region failure? A bad deploy? A node going down?

**If you only talk about Tech, you sound mid-level.** If you address all four — even briefly — you sound senior. The interviewer doesn't need depth in every dimension; they need to hear that you've *thought about* every dimension.

The mental shortcut: after proposing any architecture, immediately ask yourself:
- "Who runs it?" → People
- "How does it reach prod?" → CI/CD
- "How do we know it's working?" → Operations

If you can't answer any one of those, you haven't finished the design.

---

## ADRs — Architecture Decision Records

A short document recording *why* a particular architectural decision was made. The canonical structure (Michael Nygard, 2011):

```markdown
# ADR-0042: Use PostgreSQL instead of DynamoDB for the user service

## Status
Accepted

## Context
We need a database for user profiles. Reads dominate writes (~50:1).
Profile data is ~1KB per user. We expect <10M users in 3 years.
Team has 8 years of PostgreSQL experience and zero DynamoDB experience.
Current infra has 2 prod PostgreSQL clusters; would be a 3rd.

## Decision
Use PostgreSQL on RDS. Single primary + 2 read replicas in 3 AZs.
JSONB column for profile blob to allow schema flexibility.

## Consequences
+ Team can operate, debug, and tune it on day one
+ Joins available if future requirements call for them
+ Reuses existing backup/restore tooling
- Adds load to existing PostgreSQL operational footprint
- Doesn't scale beyond ~100M users without sharding (acceptable for 3-year horizon)
- DynamoDB's auto-scaling and low-ops model is genuinely simpler at planet scale
```

**When to write one**: any decision that future engineers will look at and ask "why did we do it this way?" If the answer requires more than 2 sentences, write an ADR.

**Where to store them**: `docs/adr/` in the repo. One file per decision. Numbered. Append-only — never edit a decision; supersede it with a new ADR that references the old one.

**Why ADRs matter for the senior mindset**: they force you to write down the **cost** of the alternative you didn't pick. That's the muscle. Most over-engineered systems happen because nobody wrote down "we considered the simple version and rejected it because X." When there's no X, the simple version was almost certainly the right call.

---

## Runbooks as design deliverables

A runbook is the document on-call uses at 2 AM to fix a production problem. A senior architect writes the runbook **as part of designing the system**, not after launch.

If you can't write the runbook, you can't operate the system. If you can't operate the system, you haven't finished designing it.

The structure that works:

```markdown
# Runbook: Service X high latency

## Symptoms
- p99 latency > 1s sustained for >5min
- PagerDuty: "service-x-high-latency"

## First 5 minutes
1. Check the deploy dashboard — was anything deployed in the last hour? If yes, consider rollback.
2. Check the SLO dashboard at <URL> — error budget remaining?
3. Check downstream dependencies (db-x, queue-y) for own alerts.

## Common causes
1. **Slow DB query** — check db-x's `pg_stat_statements`. Look for new queries.
2. **Memory pressure** — check `container_memory_working_set_bytes`. If trending up, recent rollout may have leaked.
3. **External dep slow** — check <upstream service> dashboard.

## Escalation
Page <name> if not resolved in 30 min, then <name> after 60 min.
```

The senior architect's contribution to the runbook is **knowing what to put in "Common causes"** before the system has been on call for 6 months.

---

## "Onboarding in <1 week" as a design constraint

This is the litmus test for "is your system understandable?"

A new engineer joins the team. They need to be productive in the system within a week. Can they:

- Find the relevant code in the repo by intuition (good naming, no clever indirection)?
- Run the system locally without 14 steps?
- Make a small change confidently (good tests, no spooky action at a distance)?
- Page-on-call understand what to do without paging the original author?

If "no" to any of these, the system has accidental complexity that needs to be removed. The system isn't done.

The senior framing: **"the design isn't complete until a new engineer can be on-call for it after a week of onboarding."** Most architectures don't pass this bar. The interview signal is whether you treat it as a constraint at design time, not a wish for later.

---

## 3 design-question drills with full 4-dimension answers

This is the practice format. Take a design question. Answer it. Address all 4 dimensions. Defend the simple answer.

### Drill 1: "Design a notification system that sends emails when a user's subscription expires."

**Tech**: A K8s CronJob runs every hour. Queries the `subscriptions` table for `expires_at < now() + 24h AND notified_at IS NULL`. For each row, posts to an SQS queue. A small worker (`Deployment`, 1-2 replicas) consumes the queue and calls SES `SendEmail`. Updates `notified_at`. **Why this and not Step Functions or EventBridge?** Because the team operates K8s; adding Step Functions adds a new operational surface for a problem that's "iterate over rows and send emails." YAGNI.

**People**: Anyone on the team can read the CronJob YAML and the worker code in 30 minutes. On-call already knows how to debug K8s pods. New engineer onboards in a day, not a week. No new specialized knowledge required.

**CI/CD**: Worker is built once, promoted immutably from dev → staging → prod via GitHub Actions. CronJob deployed via the same Helm chart family as our other workloads. Rollback is `helm rollback`. SES is configured via Terraform alongside other AWS resources.

**Operations**: SLO: 99% of notifications sent within 1 hour of `expires_at` window opening. SLI: count of `notified_at` updates per hour vs count of eligible rows. Alert: if eligible rows > 0 for >2 hours (queue not draining). Runbook: check SQS depth → check worker pod logs → check SES sending limits. Failure mode: SQS retains messages on worker death; idempotent on `notified_at IS NULL` check.

**What I refused to add**: a custom CRD for "notification policies." A Kafka topic instead of SQS. A separate `notifications-service` microservice. Each was tempting, none solved a current problem.

### Drill 2: "Design a system to roll out new ML model versions safely."

**Tech**: Models packaged as container images, tagged by version. Deployed via Argo Rollouts (canary strategy). 5% traffic → wait 1h → check error rate, p99 latency, business KPI → 25% → wait 4h → 100%. New version registered in a `model_versions` table for audit. **Why not a custom in-house deployment pipeline?** Because Argo Rollouts already does this; we just have to configure it. Boring.

**People**: ML team writes the model wrapper (Python). Platform team owns the rollout config. Joint runbook for "what to do if the canary fails." New ML engineer needs to know: how to tag an image, how to open a PR that bumps the version, how to read the canary dashboard. <3 days.

**CI/CD**: Model training pipeline (separate) produces a tagged image. PR auto-generated by Renovate (or ArgoCD Image Updater) bumps the image tag in the Rollout manifest. Argo Rollouts handles the canary mechanics. Failed canary → auto-pause → human decides rollback or fix.

**Operations**: SLOs: model error rate < baseline + 0.5%, p99 latency < baseline + 100ms. Auto-rollback on either breach during canary. Dashboard shows current canary state, traffic split, KPI deltas. Runbook: "canary paused" → diff the model config → check input distribution shift → either continue or revert. PagerDuty alert only on auto-rollback (not on auto-pause, which is normal).

**What I refused to add**: a custom A/B testing framework. Multi-armed-bandit traffic shifting. Bayesian rollout decisions. These solve interesting problems; none of them are the current problem.

### Drill 3: "Design a way for our internal teams to provision PostgreSQL databases on demand."

**Tech**: An internal CLI / web form that creates a Terraform PR. PR is reviewed by platform team (or auto-approved with policy checks via Conftest). Merge → Atlantis runs `terraform apply` → RDS instance created → connection details posted to a Kubernetes Secret in the requester's namespace via External Secrets Operator. **Why not a custom operator (Crossplane / a hand-rolled CRD)?** Because the team already operates Terraform + ESO. Adding an operator means adding a CRD, a controller, RBAC, upgrade path, and another way for provisioning to break. Terraform PRs are auditable, gated, and use the same tooling everything else uses.

**People**: Requester needs to fill a form. Platform team owns the Terraform module. New engineer onboarding: write a request, see the PR, understand the flow. No CRD lore required. Self-service for the requester; auditable for compliance.

**CI/CD**: Conftest in PR checks compliance (encryption at rest, no public endpoint, tags present, instance class within budget). Atlantis applies on merge. Drift detection runs nightly; PR opened automatically if any RDS resource drifts.

**Operations**: SLO: 95% of requests provisioned within 1 hour of merge. Monitor Atlantis success rate. Alert on apply failures. Runbook: failed apply → read Atlantis logs → if AWS-side issue, manual recovery; if Terraform-side, fix the module. RDS itself has its own alerting via CloudWatch + Prometheus exporter.

**What I refused to add**: Crossplane. A custom "DatabaseRequest" CRD. A self-service web app that bypasses Git review. Each adds operational surface for a problem that's "make Terraform PRs easier to write."

---

## The interview answer in 60 seconds — "design X"

For any design question, walk this 4-step structure aloud:

> "Let me start with the **simplest** approach that solves this — [propose simple answer]. Now let me check four dimensions:
>
> **Tech**: here's how the components flow, here's why I picked these over more elaborate alternatives.
>
> **People**: this team operates X already, a new engineer onboards in [N] days, the cognitive load is bounded because [reason].
>
> **CI/CD**: build once, promote immutably, here's the rollback path, here's the policy gate.
>
> **Operations**: SLO is [specific], SLI is [specific], monitoring proves [specific], runbook for the top failure mode covers [specific].
>
> If asked 'what if we need more,' my default is to add complexity *when* that requirement is concrete — not before. I'd rather duplicate twice than build an abstraction prematurely."

That's it. Practice this. Out loud. Until it's the default shape of any answer.

---

## Self-test

### 1. Why might "just a CronJob and a script" be a better answer than "Argo Events + custom CRD"?

**Reference answer:**
- The CronJob solution solves the actual problem (run job at scheduled time) with one resource and zero new abstractions. The Argo Events + CRD solution *also* solves it, but adds three components the team has to operate, learn, and page on.
- Ongoing cost (not build cost) is where complexity hurts: cognitive load for every future reader, debugging surface, on-call pages, upgrade compatibility, RBAC, documentation, training.
- Premature abstraction: the CRD is justified *if* there's a concrete second/third use case today. If there's one cron job, building the abstraction once is wasted — and the abstraction usually doesn't fit the second use case anyway.
- The senior question isn't "what's the best architecture?" but "what's the simplest thing that solves the actual problem, and is the cost of the alternative justified by a concrete current need?"

### 2. Walk me through how you'd address the 4 dimensions when designing any system.

**Reference answer:**
- **Tech**: components, data flow, failure modes, scalability ceiling. The part most engineers naturally cover.
- **People**: who operates it, how a new engineer onboards (target: <1 week), on-call story, cognitive load.
- **CI/CD**: how code reaches prod (trunk-based, immutable promotion, canary), rollback path, policy gates (Kyverno, Wiz, Conftest, SBOMs).
- **Operations**: SLO + SLI + error budget, monitoring that proves correctness (not just uptime), runbooks for the top failure modes, incident response flow.
- The mental shortcut after proposing a design: "Who runs it? How does it reach prod? How do we know it's working?" If any answer is unclear, the design isn't finished.

### 3. When is added complexity actually justified?

**Reference answer:**
- When there's a **concrete, current** requirement the simple solution can't meet — not "we might need it" or "for flexibility."
- When the **cost of the simple alternative is higher** in some named dimension (cost, latency, reliability) and the gap is measurable, not theoretical.
- When the team has the **operational maturity** to run the more complex thing without burning out on-call.
- When the **exit cost** of the complex choice is acceptable if you're wrong — i.e., not "load-bearing forever."
- Test: can you write a 2-sentence ADR for it where the "Consequences" section names real ongoing costs? If yes, justified. If the consequences section is all upside, you're missing something.

### 4. What's the difference between "essential" and "accidental" complexity? Give an example.

**Reference answer:**
- **Essential**: inherent to the problem. Cannot be removed without solving a different problem. Example: distributed consensus is genuinely hard; there's no way to make Raft simple while remaining correct.
- **Accidental**: imposed by tooling, history, conventions, or organizational structure. Reducible over time. Example: "we use SOAP because the team did in 2010" — solving the same problem with REST today would be simpler; we accept it because rewriting is more expensive than tolerating it.
- **Designed-in** (sometimes called a third category): added by the architect because they thought it'd be elegant or future-proof. Almost never justified. Example: "I built a plugin system in case we add more backends" — when there's one backend and no concrete second one in the roadmap, the plugin system is pure cost.
- The senior discipline: maximize essential, reduce accidental, refuse designed-in.

---

## Further reading / watching

### The foundational stuff (do these first)

- **John Ousterhout — "A Philosophy of Software Design"** (Stanford talk on YouTube; ~60 min). The single most influential talk on complexity in software. The book of the same name is also short and worth reading; chapters 1–3 are 80% of the value.
- **Dan McKinley — "Choose Boring Technology"** at [boringtechnology.club](https://boringtechnology.club/). 30-min read. The "innovation tokens" framing is now part of senior-engineering vocabulary.

### The Google SRE perspective

- **Google SRE Book** (free online at [sre.google/books](https://sre.google/books/)). Chapters worth your time:
  - Chapter 1 — Introduction; the SRE mindset
  - Chapter 6 — Monitoring distributed systems
  - Chapter 17 — Testing for reliability
  - Chapter 22 — Addressing cascading failures
  - The full second book ("Site Reliability Workbook") expands with practical case studies

### ADRs and decisions

- **Michael Nygard — "Documenting Architecture Decisions" (2011)**. The canonical post; do a web search for the title. ~10-min read.
- **`npryce/adr-tools`** ([github.com/npryce/adr-tools](https://github.com/npryce/adr-tools)) — the CLI most teams use to manage ADRs.
- **ADR repository at `joelparkerhenderson/architecture-decision-record`** — examples and templates across many companies.

### Senior mindset / leadership reading

- **DORA / Forsgren / Humble / Kim — *Accelerate*** (book). The data behind what high-performing teams actually do. Often summarized as: trunk-based, small batches, deploy often, fast feedback, boring tech.
- **Gene Kim — *The DevOps Handbook*** and ***The Phoenix Project*** (companion novel). The Phoenix Project is fiction but extremely effective at communicating the senior mindset.
- **Will Larson — *An Elegant Puzzle: Systems of Engineering Management*** — for the "scaling teams and architecture together" perspective.

### Specifically for the interview gap

- **Werner Vogels (AWS CTO) — blog at [allthingsdistributed.com](https://www.allthingsdistributed.com/)** — "Everything fails all the time" and related posts. Distills the AWS senior architectural philosophy.
- **Tim Hockin (K8s co-founder) — talks** on YouTube. Search for "Tim Hockin Kubernetes." His talks repeatedly emphasize simplicity and the failure modes of over-engineering.

If you watch one thing this month: **the Ousterhout Stanford talk.** It's the single biggest delta you can get for this rejection-gap area.

---

## The 4 dimensions (meta — applied to architecture itself)

- **Tech**: the simplicity discipline is itself a technical practice — measure complexity by Ousterhout's symptoms (change amplification, cognitive load, unknown unknowns), not by gut feel.
- **People**: the architect's job is to make the system understandable to people who didn't build it. Onboarding-in-<1-week is the metric. Pair-design rather than solo-design when possible.
- **CI/CD**: write the ADR *as part of* the PR that adds the complexity. Reviewers can challenge "why not the simpler version?" If there's no ADR, that's the first review comment.
- **Operations**: write the runbook before launch. If you can't, the design isn't done. Track simplicity over time as a code metric (e.g., service count, abstraction depth) and let regressions trigger discussion.
