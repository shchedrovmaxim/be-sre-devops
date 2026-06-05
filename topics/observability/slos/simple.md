# SLI / SLO / Error budgets — the simple version (the monthly data plan)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **An error budget is the amount of badness your service is allowed before someone has to slow down and fix things. It's the *operationalized* version of "how reliable is reliable enough?"**

That's the whole concept. Everything else (SLI, SLO, SLA, burn rate, alert design) is just precision on top.

---

## Your service is on a monthly data plan

You have a phone with a **100 GB / month** data plan.

| In the phone world | In the SRE world |
|---|---|
| 100 GB per month | The **SLO**: 99.9% of requests succeed in 30 days |
| Each GB you use | An **error** burning your budget |
| Each GB you have left | Your **error budget** |
| The data meter (currently using 2 MB/min) | Your **SLI**: a live measurement |
| Streaming 4K (uses 7 GB/hr) | A **high burn rate** — you'll blow your budget by Tuesday |
| Reading email (uses 5 MB/hr) | A **low burn rate** — you'll barely touch the budget |
| Halfway through the month, 95 GB used → carrier throttles you | You've blown your error budget → **deploy freeze**, focus on reliability |
| End of month with 50 GB unused → carrier doesn't care | You have budget left → **ship that risky change**, take some chances |

That's it.

The crucial mental flip:

> **You don't try to use ZERO data. You try to stay within budget.**
> **You don't try for ZERO errors. You try to stay within budget.**

100% reliability is the wrong goal. It's expensive (every nine costs ~10× more than the last), and your customers can't tell the difference between 99.99% and 99.999% — but they can tell the difference between "we ship features fast" and "we ship features slow." **The error budget is what buys you the right to take risks.**

---

## The 3 things you need to define

For any service, define:

### 1. SLI — what you measure

A single, specific, machine-readable signal about whether the service is doing its job. The standard formulation:

> **good events / valid events**

Examples:
- Successful HTTP requests / total HTTP requests → **availability SLI**
- HTTP requests served in < 200 ms / total HTTP requests → **latency SLI**
- Messages processed within 1 minute / total messages → **queue freshness SLI**

The SLI is *measurable right now*, every minute, by your monitoring system. It's a ratio between 0 and 1.

### 2. SLO — the target

A threshold on the SLI, evaluated over a time window.

Example: **"99.9% of requests succeed, measured over a rolling 30-day window."**

That's an SLO. Three parts:
- **The SLI** ("requests succeed")
- **The target** (99.9%)
- **The window** (rolling 30 days)

### 3. Error budget — the inverse

If your SLO is 99.9%, your error budget is **0.1%**. In a 30-day window with 1 million requests, you're "allowed" 1,000 failed requests before you blow the SLO.

The budget isn't free money — every error you have, you spend. But it gives you **explicit room to take risks**: a risky deploy, a chaos experiment, an aggressive optimization. As long as you stay within budget, those are fine.

---

## What does the error budget actually BUY you?

This is the senior insight. The budget exists to **resolve the eternal tension between dev velocity and reliability**.

- **Lots of budget left** → ship the risky thing. Try the new deploy strategy. Run the chaos experiment. Push the optimization. You can afford some failure.
- **Budget spent** → freeze deploys. Stop the feature work. Spend the next week on reliability fixes. Pay down the debt that caused the burn.

This isn't a punishment. It's a forcing function that turns "we should do reliability work eventually" into "we must do reliability work this week because the budget says so."

That makes "should we ship?" a **data-driven decision**, not a political one. No more "the CEO wants this feature, ship it" overriding "the SRE team thinks it's risky." The answer is in the budget.

---

## Burn rate — the speed at which you're spending

If your error budget is the GB of data left, the **burn rate** is the MB/sec you're currently using.

- **Burn rate 1×** = you're spending the budget exactly as fast as the SLO permits. You'll exit the window with budget around zero. Fine, expected.
- **Burn rate 10×** = you're burning 10× faster than allowed. You'll blow the budget in 1/10 the window. **Page someone now.**
- **Burn rate 50×** = something catastrophic. You'll blow a 30-day budget in less than a day. **Page urgently.**

This is how modern SRE alerts are built. Instead of alerting on "error rate > 1%," you alert on "burn rate > 14× for 5 minutes" — which directly tells you "we're about to blow the budget."

The Google SRE book calls this **"multi-window multi-burn-rate"** alerting. The deep-dive has the math.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's an SLI? | A measurable signal: good events / valid events. Always a ratio. |
| What's an SLO? | A target on the SLI over a time window: "99.9% in 30 days." |
| What's an SLA? | The legal/contractual version of an SLO, with consequences (refunds, penalties). Usually weaker than the internal SLO. |
| What's an error budget? | The inverse of the SLO: how much badness you're allowed. |
| Why does it matter? | Because **100% is the wrong target**. The budget operationalizes "how reliable is reliable enough" so dev and SRE stop arguing. |
| What's burn rate? | How fast you're spending the budget. 10× burn = exhausting the budget 10× faster than planned. |
| What's an error budget policy? | The pre-agreed rule for what happens when you blow the budget. Common one: deploy freeze until budget recovers. |
| Should every service have an SLO? | No. SLOs are work. Only set them for services that have *real users* depending on them. Internal tools sometimes don't need one. |

---

## Self-test (one question — the killer one)

Out loud:

> **"How would you set up an SLO for a service that has both a latency requirement AND an availability requirement? What's the error budget policy?"**

**Reference answer (intuitive version):**

"I'd define **two SLIs** — they're different signals. The latency SLI is the ratio of requests served within a threshold (say, p99 < 200 ms), and the availability SLI is the ratio of successful responses (non-5xx). Each gets its own SLO — for example, 99% of requests under 200 ms, and 99.9% successful, both over a rolling 30-day window. That gives me two separate error budgets to manage; either one being exhausted matters.

The error budget policy I'd write down beforehand has two tiers. **Tier 1 — minor burn**: if either budget drops below 25% remaining, dev velocity is OK but new feature work needs an SRE pair-review before merging. **Tier 2 — budget exhausted**: deploy freeze on the affected service except for reliability fixes and security patches. Both teams agree to this in writing *before* there's a problem, so the freeze isn't a political fight when it happens.

For alerting, I'd use multi-window multi-burn-rate alerts on each budget — a high-severity page if we're burning at >14× rate for 5 minutes (catches fast outages), and a medium-severity ticket if we're burning at >3× for an hour (catches slow regressions). Same for both latency and availability. That way an outage pages immediately, but a gradual reliability degradation also gets attention before it becomes an outage."

If you got that out cleanly — naming **two SLIs**, **two budgets**, a **pre-agreed policy**, and **burn-rate alerts** — you've internalized this. The deep-dive adds the precise math, the SLI selection process, and 3 worked examples.

---

## Further reading / watching

The single most influential resource:

- **Google SRE Book** (free online at [sre.google/books](https://sre.google/books/)). Chapters 3, 4, 6 are the SLI/SLO/SLA + error budget canon. The companion "Site Reliability Workbook" expands chapters 2 and 5 with practical patterns including multi-window multi-burn-rate alerts.

For getting started:

- **Alex Hidalgo — *Implementing Service Level Objectives*** (book). The most practical book on actually rolling SLOs out at a real organization. Covers the human/political side (who owns the SLO, how to negotiate with product), not just the math.

Tools you'll see in the wild:

- **Sloth** ([github.com/slok/sloth](https://github.com/slok/sloth)) — a tool that generates Prometheus recording rules and multi-burn-rate alerts from a simple SLO YAML definition. Great way to start.
- **Pyrra** ([github.com/pyrra-dev/pyrra](https://github.com/pyrra-dev/pyrra)) — similar idea, K8s-native, with a UI for SLO management.
- **OpenSLO** ([openslo.com](https://openslo.com/)) — a spec for declaring SLOs in YAML; vendor-neutral.

---

## Next: the deep-dive

When the data-plan analogy and the "good / valid events" formulation feel obvious, jump to [`slos.md`](./deep-dive.md). The deep-dive covers:

- The senior framing: SLOs are about negotiating risk, not measuring perfection
- What makes a **good SLI** vs a useless one (the CUJ approach)
- Latency SLI nuances — percentile thresholds, latency buckets, the bucketing gotcha
- Availability SLI nuances — what counts as a "valid event"
- The **error budget policy** template
- **Multi-window multi-burn-rate** alerts in detail (with the actual math)
- 3 worked examples: web service (availability + latency), batch worker (queue freshness), data pipeline (freshness + completeness)
- Common SLO mistakes (the "shadow SLA" trap, the dashboard-without-policy trap)
- 4 self-test drills with reference answers
- The 4-dimensions framing (Tech + People + CI/CD + Operations) applied to SLO work

The deep-dive is the reference. This doc is the mental model. The data-plan analogy is what you should keep in your head during any SLO discussion.
