# SLI / SLO / SLA / Error budgets — the deep-dive

> **Goal**: by the end you can answer the killer question — **"How would you set up an SLO for a service with both a latency and availability requirement? What's the error budget policy?"** — naming two SLIs, two budgets, a pre-agreed policy, and burn-rate alerts. You can also pick *meaningful* SLIs (not "uptime") and resist the most common SLO mistakes.

> Start with the [simple version](./slos-simple.md) if you haven't read it. The monthly-data-plan analogy is the spine of everything here.

---

## The senior framing — SLOs are about negotiating risk

Mid-level engineers think SLOs are about **measuring uptime**.
Senior engineers know SLOs are about **negotiating between dev velocity and reliability**, in data, before there's a problem.

The eternal tension on any team:

- **Dev wants** to ship features fast. Risky deploys, bold changes, fast iteration.
- **SRE wants** the service to be reliable. Slow deploys, careful changes, paged people sleep at night.

Without an SLO, this is a political fight that gets resolved by whoever is louder or more senior. With an SLO, it becomes a data-driven decision: **how much error budget do we have? If lots, ship. If little, slow down.**

The error budget is what *operationalizes* "how reliable is reliable enough." It puts a number on the trade-off so the team can act on it without arguing.

**Bonus senior insight**: **100% reliability is not the goal.** Each "nine" of reliability costs ~10× the previous one (99.9% → 99.99% is a 10× engineering investment). Your customers can't perceive the difference between 99.9% and 99.99%, but they *can* perceive the difference between "this team ships features weekly" and "this team ships features monthly." Spend your money where the difference matters.

---

## SLI — Service Level Indicator

A specific, machine-measured signal about whether the service is doing its job. The standard formulation is a ratio:

```
SLI = good events / valid events
```

- **good events** — successful, well-behaved instances of the thing being measured
- **valid events** — all instances of that thing, *excluding* things that aren't legitimate measurements

Examples:

| Service type | SLI | "good events" | "valid events" |
|---|---|---|---|
| Web API | Availability | HTTP requests with status < 500 | All HTTP requests (excluding 4xx caused by client) |
| Web API | Latency | Requests served in < 200 ms | All requests |
| Batch worker | Throughput | Messages processed within 60s of receipt | All messages received |
| Data pipeline | Freshness | Records arriving within 5 min of source timestamp | All records |
| Storage | Durability | Bytes read back successfully | All bytes written |

### Why "valid events" matters

A bad SLI: "100 − percentage of 5xx responses." This counts the wrong things. If your service returns 5xx because a *client* sent malformed data, that's not a service-quality problem — it's a client problem. Counting it as bad means your SLI is conflating two failure modes.

Good SLI design carefully defines what counts as valid:

- 5xx responses to *server-side* errors → bad event
- 4xx responses (client errors) → not a service-quality problem; exclude from "valid events" or count as good
- Requests dropped at the load balancer before reaching your service → still bad (the user experienced it)
- Health-check requests from internal monitoring → exclude entirely; not a user request

The single most common SLI bug: forgetting to exclude health checks from the denominator. They make your SLI look great because they always succeed.

### Critical User Journey (CUJ) — how to pick SLIs

Don't just measure "the API is up." Measure the **journeys the user cares about**. For an e-commerce site:

- **CUJ 1**: Browse → add to cart → checkout. The SLI is "successful checkouts / attempted checkouts." Doesn't matter if the search API is fast; if checkout is broken, the user is gone.
- **CUJ 2**: Search → view product. SLI: "search results returned in < 500 ms / search requests."
- **CUJ 3**: Log in. SLI: "successful logins / login attempts."

Each CUJ gets its own SLI (and probably its own SLO). The user's experience of "the site works" is the intersection of these.

**The product mindset**: an SLO is a promise about the user's experience, not about your infrastructure. "EKS is up" is not an SLO. "Users can complete checkout 99.9% of the time" is.

---

## SLO — Service Level Objective

The target value of an SLI, evaluated over a time window.

```
SLO = SLI ≥ threshold  over  rolling window
```

Example: **99.9% of requests succeed, over a rolling 30-day window.**

Three components:

| Component | Example | Notes |
|---|---|---|
| SLI | "HTTP requests with status < 500 / all valid requests" | The measured ratio |
| Target | 99.9% | The "how reliable" number |
| Window | rolling 30 days | The period over which the SLO is evaluated |

### Choosing the target

Pick the target based on **what users actually need**, not what feels round. Process:

1. **Look at current performance**. If you're already at 99.95% over the last quarter, setting an SLO at 99.99% means immediate failure with no slack.
2. **Look at what users tolerate**. For an internal admin tool, 99% is probably fine. For payments, 99.99% might be required.
3. **Pick a number you can actually defend in 6 months**. An SLO is a promise. Don't promise more than you can keep.

### Choosing the window

Rolling 28 or 30 days is the standard. Why rolling?

- **Calendar-month** windows have ugly edge effects: a bad week at the end of February has different impact than a bad week at the start of March.
- **Rolling** smooths this out. At any moment, you're evaluating against the last 30 days.

Shorter windows (7 days) give faster feedback but are noisy. Longer windows (90 days) hide problems but are stable. **Choose based on how fast you want to respond to regressions.** Most teams use 28 or 30 days.

---

## SLA — Service Level Agreement

The legal / contractual version of an SLO. Includes **consequences** (refunds, penalties) when violated.

Two important rules:

1. **SLA target should be weaker than the internal SLO.** If your SLO is 99.95%, your SLA might promise customers 99.9%. The gap is your safety margin — you can blow the SLO without breaching the SLA.
2. **An SLA without an internal SLO is dangerous.** If you only have an SLA and nothing more aggressive internally, you have no warning signal before you breach the contract.

Most internal services don't have SLAs (no contract, no consequences). They have SLOs only. That's fine.

---

## Error budget — the inverse of the SLO

If your SLO is 99.9%, your error budget is **0.1%** of valid events in the window.

Concrete: 1 million requests in 30 days at SLO 99.9% → **error budget = 1,000 failed requests** over 30 days.

That budget is **spent every time a request fails**. The team can see:
- Budget remaining (current and projected)
- Burn rate (how fast it's being spent right now)
- Time to exhaustion at current rate

### Why thinking in budget (not error rate) matters

If you say "alert when error rate > 0.5%", you'll get alerts for brief spikes that don't matter. The error rate measures *instantaneous* badness; the budget measures *accumulated* badness against a target.

A 5-minute spike to 2% error rate, if everything is fine the rest of the day, **doesn't matter** — the day's budget impact is small. A persistent 0.2% error rate that's been going for a week **matters** — you're trending toward budget exhaustion. The budget view catches both correctly; the rate view doesn't.

---

## Error budget policy — what happens when you blow it

The policy is **pre-agreed**, **written down**, and **signed off by both dev and SRE** before there's a problem. The whole point is to avoid the political fight in the moment.

A sample tiered policy:

| Budget remaining | Posture |
|---|---|
| > 75% | Normal velocity. Risky deploys OK. Run chaos experiments. |
| 25-75% | Normal velocity, but reliability work prioritized in next sprint. |
| < 25% | New feature work pauses for SRE pair-review. Only reliability, security, and revenue-critical work. |
| Exhausted (< 0%) | **Deploy freeze.** Only reliability, security, and revenue-critical work. Daily review until budget recovers. |
| Exhausted for > 30 days | Re-negotiate the SLO. Either the target is unrealistic, or fundamental work is needed (architecture review). |

Variations exist — some teams use a single threshold ("burned 50% in the first week → freeze"). The key is: **it's pre-agreed.** When the budget gets blown at 3am, no one is arguing about what to do.

### What the policy is NOT

- Not a punishment.
- Not "you screwed up, no fun for you."
- Not a permanent state.

It's a **forcing function**. It turns "we should do reliability work eventually" into "we must do reliability work this week because the policy says so." The team has explicit permission (and pressure) to deprioritize features without being seen as not delivering.

---

## Burn rate alerts — multi-window multi-burn-rate

The Google SRE book's modern approach. The intuition:

- A **1× burn rate** spends the budget in exactly the window length (30 days). Acceptable.
- A **10× burn rate** spends the budget in 3 days. Bad.
- A **100× burn rate** spends the budget in 7 hours. Catastrophic.

You want alerts that fire at increasing severity as the burn rate goes up. The trick is **two windows** for each rate, to avoid false alarms from short spikes.

A standard pattern (from the SRE workbook):

| Severity | Burn rate threshold | Long window | Short window | Fires when |
|---|---|---|---|---|
| Page (urgent) | 14.4× | 1 hour | 5 min | Burning fast enough to exhaust monthly budget in 2 days, sustained for at least 5 min |
| Page (high) | 6× | 6 hours | 30 min | Burning at 5-day-exhaustion pace, sustained for 30 min |
| Ticket (slow) | 3× | 24 hours | 2 hours | Burning at 10-day-exhaustion pace, sustained for 2 hours |
| Ticket (chronic) | 1× | 3 days | 6 hours | Slow leak — burning exactly at SLO limit for 6 hours sustained |

The "long window" sets the burn-rate threshold (need at least N% bad over the long window to consider it a real burn). The "short window" gates how quickly the alert fires (need it sustained for the short window before paging).

**Why this is better than rate-based alerts**:
- Doesn't false-alarm on 30-second spikes (the short window must be sustained).
- Doesn't miss slow-burn regressions (the chronic ticket catches them).
- Severity scales with how fast you'd blow the budget — pages match urgency.

In Prometheus terms, this is built with recording rules computing the burn rate over each window, and alerting rules that fire when both windows exceed the threshold. Tools like **Sloth** generate all this YAML from a simple SLO spec.

---

## Worked examples

### Example 1 — Web service with both availability and latency

```yaml
# Conceptual SLO spec (pseudocode)
slo: checkout-api
window: 30d

availability:
  sli: count(http_requests{status!~"5.."}) / count(http_requests)
  target: 99.9%
  budget: 0.1% = 43.2 min downtime per 30 days

latency:
  sli: count(http_requests{duration<200ms}) / count(http_requests)
  target: 99%
  budget: 1% of requests over 200ms allowed

policy:
  - if either budget < 25%: SRE pair-review required for non-trivial PRs
  - if either budget exhausted: deploy freeze on checkout-api, daily review

alerts:
  - severity: page,   burn_rate >= 14.4x, long: 1h,  short: 5m
  - severity: page,   burn_rate >= 6x,   long: 6h,  short: 30m
  - severity: ticket, burn_rate >= 3x,   long: 24h, short: 2h
  - severity: ticket, burn_rate >= 1x,   long: 3d,  short: 6h
```

Two budgets, evaluated independently. A latency regression can blow the latency budget without affecting availability. The policy applies if *either* is exhausted.

### Example 2 — Batch worker (queue freshness SLO)

A worker that processes Kafka messages. "Up or down" isn't the right SLI — the worker can be up and falling behind.

```yaml
slo: order-processor
window: 7d   # batch workloads often want shorter windows

freshness:
  sli: count(messages_processed{lag<60s}) / count(messages_processed)
  target: 99%

policy:
  - if budget exhausted: pause non-critical features in upstream services
    that feed this queue (i.e., reduce load), prioritize worker scaling

alerts:
  - severity: page,   burn_rate >= 14.4x, long: 1h, short: 5m
```

For batch, latency is replaced by freshness. The error budget operationalizes "how stale is too stale."

### Example 3 — Data pipeline (freshness + completeness)

A pipeline that ingests daily reports.

```yaml
slo: analytics-pipeline
window: 28d

freshness:
  sli: count(reports_arrived{within_1h}) / count(scheduled_reports)
  target: 99%

completeness:
  sli: count(records_passing_validation) / count(records_received)
  target: 99.5%

policy:
  - chronic SLO violations trigger an upstream-source review
    (often the source is the actual problem, not the pipeline)
```

Two SLIs because "did it run on time" and "was the data right" are different failure modes.

---

## Common SLO mistakes

### 1. The "shadow SLA" trap

Engineers without SLOs implicitly promise 100% reliability because no one has agreed on a target. When 99.9% happens, the team is blamed even though no formal target was missed. **Fix**: write down an SLO explicitly. The number on paper becomes the bar.

### 2. The dashboard-without-policy trap

A team has beautiful SLO dashboards but no error-budget policy. When the budget is blown, no one knows what to do. Velocity continues as usual. Reliability work never happens. **Fix**: don't ship an SLO dashboard without a policy that says what the team will do when the budget burns.

### 3. The "uptime" SLI trap

Measuring "is the load balancer responding to ping?" instead of "are users successfully checking out?" Tells you nothing about user experience. **Fix**: pick SLIs that match Critical User Journeys, not infrastructure availability.

### 4. The "every microservice gets an SLO" trap

You don't need an SLO per microservice. You need an SLO per **user-facing journey**. The internal microservices contribute to that, but they don't each need their own SLA. **Fix**: SLOs at the journey level; SLIs on internal services are diagnostic, not contractual.

### 5. The aggressive target trap

Setting 99.999% because it sounds impressive. Then the team breaks it every week. Then the SLO loses credibility. Then no one cares. **Fix**: pick a target you can defend. 99.9% is a fine starting point; you can always tighten later.

### 6. The "exclude failures from the math" trap

When the SLO is blown, the temptation is to exclude the cause ("that was an AWS region outage, not our fault"). Sometimes legit (the AWS outage really wasn't your fault). Often a way to game the metric. **Fix**: define exclusion criteria upfront in the SLO doc. New exclusions need explicit approval, not retroactive editing.

### 7. The "we'll figure out alerts later" trap

Writing the SLO and then alerting on the raw error rate. Doesn't catch slow burns; false-alarms on short spikes. **Fix**: implement multi-window multi-burn-rate alerts from the start. Tools like Sloth make this YAML-trivial.

---

## The interview answer in 60 seconds

> "I'd start by identifying the Critical User Journey, then define SLIs for it. For a service with both latency and availability requirements, that's **two SLIs** — for example, count of successful requests over total, and count of requests served under 200ms over total. Each gets its own SLO and its own error budget. Typical targets: 99.9% availability, 99% latency-under-threshold, both over a 30-day rolling window.
>
> I'd write down an **error budget policy** pre-emptively. Tiered: if budget drops below 25%, new features need SRE review; if exhausted, deploy freeze except for reliability and security. Both teams sign off before the policy is needed.
>
> For alerts, I'd use **multi-window multi-burn-rate** — pages at 14.4× burn over 1h/5min for fast outages, and tickets at 3× burn over 24h/2h for slow regressions. Same on both SLOs. Page-only-on-user-impact discipline; everything else is a ticket.
>
> The senior point I'd emphasize: **the SLO isn't a measurement tool, it's a decision tool.** It exists to operationalize 'how reliable is reliable enough' so the team isn't politically arguing about ship-vs-don't-ship. The budget tells them."

---

## Self-test

### 1. How would you set up an SLO for a service with both latency and availability requirements? What's the error budget policy?

**Reference answer:**
- Two SLIs (latency, availability), each as a good/valid ratio.
- Each gets its own SLO over a rolling 30-day window — e.g., 99% latency-under-200ms, 99.9% availability.
- Two error budgets, managed independently.
- Pre-agreed tiered policy: minor burn → SRE review on PRs; exhausted → deploy freeze except for reliability/security/revenue-critical work.
- Multi-window multi-burn-rate alerts on each budget: page at 14.4× burn (1h/5min), ticket at 3× burn (24h/2h), etc.
- Bonus: emphasize that the SLO is a *decision tool* (operationalize the dev-vs-SRE tension), not just a measurement.

### 2. What makes a good SLI?

**Reference answer:**
- Specific to a Critical User Journey, not "is the box up."
- A ratio: good events / valid events.
- "Valid events" carefully defined: exclude health checks, exclude client-side 4xx errors (or count them as good).
- Measurable from existing monitoring (Prometheus, OTel) — not requiring new instrumentation.
- Aligned to user perception — the user experiences the SLI, not the SRE team.
- Bonus: pick fewer SLIs that matter than many that don't. Cardinality and review cost are real.

### 3. Why might setting an SLO at 99.99% be a mistake?

**Reference answer:**
- 99.99% costs ~10× more engineering investment than 99.9%, and users typically can't perceive the difference.
- It leaves only 4.3 minutes/month of downtime budget — every deploy that goes slightly wrong blows the SLO.
- An SLO that's constantly blown loses credibility and stops driving behavior.
- The right target is what users actually need (start by surveying or A/B testing) and what you can defend in 6 months.
- Better to start at 99.9% and tighten later if you have headroom than start at 99.99% and constantly miss.

### 4. What's the difference between alerting on burn rate vs alerting on error rate?

**Reference answer:**
- Error rate alerts on *instantaneous* badness. A brief 2% error spike triggers a page, even if the day's budget is unaffected.
- Burn rate alerts on the *rate at which you're spending the error budget*. A sustained 0.5% error rate over hours is a higher burn rate than a 2% spike for 30 seconds.
- Burn rate matches the actual concern ("are we going to blow our budget?") instead of a proxy.
- Multi-window multi-burn-rate uses two time windows (long + short) at each threshold — the long window decides "is this real?" and the short window decides "fire now or later?"
- Result: fewer false alarms (spikes don't page), catches slow burns (chronic low-rate regressions also alert), and severity scales naturally with how bad the situation is.

---

## Further reading / watching

### The canonical sources

- **Google SRE Book** (free online at [sre.google/books](https://sre.google/books/)).
  - Chapter 3: Embracing Risk
  - Chapter 4: Service Level Objectives
  - Chapter 6: Monitoring Distributed Systems
- **Site Reliability Workbook** (free at the same site). The practical companion.
  - Chapter 2: Implementing SLOs
  - Chapter 5: Alerting on SLOs (this is where multi-window multi-burn-rate is documented in detail)

### Books

- **Alex Hidalgo — *Implementing Service Level Objectives*** (O'Reilly). The most practical book on rolling SLOs out at a real organization. Covers the human/political dimensions (who owns the SLO, how to negotiate with product) — what the Google book doesn't.

### Tools

- **Sloth** ([github.com/slok/sloth](https://github.com/slok/sloth)) — Prometheus-native. Define SLOs in YAML; Sloth generates the recording rules and multi-burn-rate alerts. Best place to start in a Prometheus environment.
- **Pyrra** ([github.com/pyrra-dev/pyrra](https://github.com/pyrra-dev/pyrra)) — K8s-native SLO management, with a UI.
- **OpenSLO** ([openslo.com](https://openslo.com/)) — vendor-neutral spec for declaring SLOs.

### Talks

- **Google's "How SRE Relates to DevOps" series on YouTube** — short, foundational.
- **Liz Fong-Jones — talks on observability and SLOs**. Search "Liz Fong-Jones SLO." She's the practical voice; the Google SRE book is the theory.

If you only do two things from this list: read SRE Workbook Chapter 5 (multi-window multi-burn-rate) and try Sloth in a side project. Together they'll lock the concept in.

---

## The 4 dimensions (senior framing)

- **Tech**: SLIs as ratios; multi-window multi-burn-rate alerts; recording rules for performance; Prometheus + Sloth as the standard stack; SLOs at journey level, not microservice level.
- **People**: SLOs are a negotiation tool, not a measurement tool. The work is getting product + dev + SRE aligned on what "reliable enough" means. Pair with product to define CUJs. Document the policy in a shared doc, with sign-off.
- **CI/CD**: enforce error-budget policy in deploy tooling — block production deploys when budget is exhausted (except labeled-reliability or labeled-security PRs). Show current SLO state in the PR-checks UI so devs see it before merging.
- **Operations**: monitor the SLOs themselves, not just the services. A dashboard per SLO showing current SLI, current budget remaining, current burn rate, projected exhaustion. Runbook for "budget burning fast" — what to investigate. Quarterly SLO review: is the target still right? Are we measuring the right CUJ?
