# STAR Behavioral Stories — Senior SRE/DevOps Template

> Behavioral wins or loses you the offer in the final round. Senior SRE JDs typically emphasize: **high autonomy, ownership, proactive mindset, fast-scaling platforms**.
>
> If you've worked solo or in a small team — that's a structural advantage, not a weakness. Solo = highest-autonomy job there is. Lean into "I've owned every layer, made every call, run my own production." That maps directly to senior SRE roles where you share ownership of a business unit with a small group.

---

## STAR refresher

- **S** — Situation: brief context. 1-2 sentences.
- **T** — Task: what you needed to do, and the constraint that made it hard.
- **A** — Action: what *you specifically* did. Use "I," not "we." Show decision-making and trade-offs.
- **R** — Result: concrete outcome with numbers. What you learned.

Common failures:
- No numbers ("it was much faster" → how much faster?)
- Hiding behind "we" so they can't tell what you did

---

## Have 5 stories ready

Adapt these templates to your actual experience. Each story has the **structure** + a **role-tie-in** at the end.

---

## Story 1 — Ownership end-to-end (the cornerstone)

**Maps to:**
- "Tell me about a project you owned end-to-end."
- "Tell me about a complex system you built."
- "Walk me through something you're proud of."

### Template

**Situation:** "I built and ran [your project] as the [solo / lead / sole] engineer. That meant owning everything from [scope] to [scope] to incident response — no separate team to escalate to."

**Task:** Pick ONE concrete thing that shows infra/SRE ownership. Examples:
- Designing the production environment from scratch
- Migrating from one infrastructure approach to another
- Standing up observability that scaled with user growth
- Implementing the resilience patterns (timeouts, retries, circuit breaking) across services

Pick something specific. Not "I built the whole thing" — say "I designed the production deployment pipeline."

**Action:** Spend the most time here. Show:
- The decisions you made and why — "I chose managed Kubernetes over self-hosted because…"
- The trade-offs you considered — "Cost vs operational overhead vs flexibility — I picked X because of Y."
- The actual technical work — name tools, patterns, YAML.
- The constraints — "[Solo / small team], so anything requiring ongoing manual ops was a non-starter."

**Result:**
- Concrete: "App scaled to N users with M req/s, p99 under X ms, zero unplanned downtime for Y months."
- What you learned: "I learned the value of investing in [whatever] early — paid off when [moment]."

### Role tie-in
> "What I want to bring to [target role] is that operator's mindset — owning a slice of infrastructure end-to-end, making the technical calls, and being accountable for the result. That experience means I'm comfortable with the autonomy this role calls for, and I'm ready to scale that ownership into a larger team context."

### What NOT to do
- Don't pretend you have team-lead experience you don't.
- Don't apologize for being solo / small-team. Frame as strength.
- Don't list every tech. Pick ONE decision and go deep.

---

## Story 2 — Proactive prevention

**Maps to:**
- "Tell me about a time you spotted a problem before it became one."
- "Give an example of being proactive."
- "How do you balance reliability vs feature delivery?"

### Template

**Situation:** "Early in [project], I noticed we were [specific weakness — e.g. running on a single-AZ deployment, no monitoring on the critical path, an unbounded queue]."

(Or: observability gap closed before outage, dependency removed before it bit you, security fix before a vulnerability.)

**Task:** "I had to decide whether to invest engineering time fixing this NOW — before any user noticed — or wait until something forced my hand. The constraint: [time / cost / team capacity]."

**Action:**
- "I did a risk-vs-cost analysis. [Specific reasoning about the failure mode and its likelihood.] Migration cost: ~N days work + X% infra spend. Decision: do it now."
- "I migrated to [the fix] with [the specific patterns: PodAntiAffinity / Multi-AZ / cross-zone LB / etc.]."
- "I tested the failure path by [simulating the failure] and confirming the system handled it cleanly."

**Result:**
- "[N months] later, [the predicted failure actually happened]. Our system didn't notice / recovered automatically / kept serving."
- "Lesson: invest in resilience BEFORE you have an incident. The cost of prevention is always less than the cost of recovery."

### Role tie-in
> "Users of [target product / company] care a lot about reliability. Proactive reliability work isn't a nice-to-have; it's existential at that level of trust. That's the discipline I'd bring."

### What NOT to do
- Don't pick trivial prevention ("I added a try/catch"). Pick something with real cost/benefit.
- Don't pretend you saw the future. Frame as **risk analysis**.

---

## Story 3 — Production incident

**Maps to:**
- "Tell me about a production incident."
- "Tell me about a time something broke."
- "How do you handle high-pressure situations?"

You need at least one **real** incident. Even small. Embellishing gets caught in follow-up questions.

### Template

**Situation:** "On [date], [the failing component] started [the symptom]. Our [affected system] started [the secondary failure mode — e.g. piling up requests, returning 5xx, hitting timeout limits]."

**Task:** "I needed to (1) stop the bleeding for users, (2) understand what was happening, (3) fix it so it couldn't happen the same way again."

**Action:**
- "First I confirmed scope — was it [hypothesis 1] or [hypothesis 2]? It was [the actual scope]."
- "Looked at metrics — [the signal that pointed to root cause]. [What that signal indicated]."
- "Mitigation: [the fix that stopped the bleeding — circuit breaker, kill switch, traffic shift]. Within minutes the system stabilized; users got [degraded experience] instead of [full outage]."
- "Then tracked down the actual root cause — [what it actually was]."
- "Post-incident: [what you changed structurally so it can't recur]."

**Result:**
- "Users experienced ~N minutes of [degradation] instead of [full outage]. [Concrete impact metric — no customer churn, X% error rate during incident, etc.]."
- "Lesson: [a real lesson — fault isolation vs fault tolerance, dependency mapping, alert design, whatever applies]."

### Role tie-in
> "Exactly what you'd want at [target company] — one upstream goes bad, the whole product can't go down. Same patterns whether it's [my context] or [their context]."

### What NOT to do
- Don't pick a story where you were the cause and don't acknowledge it.
- Don't tell without a postmortem-style lesson.
- Don't over-dramatize.

---

## Story 4 — Technical decision/trade-off

**Maps to:**
- "Tell me about a significant technical decision."
- "How do you choose between options?"
- "When have you pushed back on a popular approach?"

### Template

**Situation:** "Early on I had to decide [the decision]." (database choice, CI/CD, monorepo vs polyrepo, observability stack, etc.)

**Task:** "The options were: (1) [option A], (2) [option B], (3) [option C]."

**Action:**
- "I evaluated on three axes: [axis 1], [axis 2], [axis 3]. (Common axes: operational complexity, cost at expected scale, time-to-deliver.)"
- "[Option A] was cheap but [the tradeoff]."
- "[Option B] was simplest but [the tradeoff]."
- "[Option C] had highest ops complexity but [the upside]."
- "Went with **[the chosen option] initially** because [the scale-appropriate reasoning], and migrated to [next option] when we hit [specific scale event]."
- "Documented in an ADR so future-me would remember the reasoning."

**Result:**
- "Migration happened cleanly because I'd designed [the abstraction layer] from the start."
- "Lesson: **defensible decisions**. The right answer at one scale is the wrong answer at another. Senior move is making the decision *for the scale you have*, not the scale you imagine."

### Role tie-in
> "Same thinking I'd bring to [target role]: decisions that fit the current reality with a clear migration path when assumptions change, rather than over-engineering for hypothetical futures or under-engineering and getting stuck."

### What NOT to do
- Don't tell a story where the answer is obvious in hindsight.
- Don't pick the cool/trendy thing. Pick the **pragmatic** thing.

---

## Story 5 — Failure / learning

The **trust test**. Refuse to give a real failure and you lose them. Story should be:
- **Real** (they can smell fake)
- **Bounded** (not "I crashed prod for a week")
- **Educational**
- **Self-aware** (take responsibility, don't blame)

### Template

**Situation:** "I overinvested in [thing] before I had [the actual need]."

Examples:
- Sophisticated CI/CD before anything to deploy.
- Multi-region from day one with 30 users.
- Custom observability tooling when off-the-shelf would have worked.
- Microservice boundary that didn't pay back.

**Task:** "I spent [N weeks] building [thing] because I convinced myself I'd need it later."

**Action:**
- "Built it. Worked. I was proud."
- "But it added operational complexity I had to maintain — patches, upgrades, runbooks I had to write for myself."
- "[N months] in, realized I was spending more time maintaining the [thing] than using it. The actual problem it solved didn't materialize at the scale I expected."

**Result:**
- "Ripped it out, replaced with [simpler approach]. Simpler approach handled 100% of what I actually needed."
- "Lesson: **YAGNI**, hard. I'd carried over a 'design for scale' instinct from patterns I'd read about, but the right time to add complexity is when you actually need it."
- "Now I default to the simplest thing that works, and write down what trigger would cause me to graduate to a more sophisticated approach. The trigger is the green light, not my imagination."

### Role tie-in
> "I bring that discipline now — at [target company]'s scale, complexity has real cost, but so does under-investing in resilience for [their context, e.g. a security-critical product]. The judgment is knowing where you actually are on that spectrum."

### What NOT to do
- Don't tell a "failure" that's a humblebrag ("worked too hard").
- Don't blame others.
- Don't tell a failure with no learning.

---

## The big one: "Why are you leaving your current setup?"

Will come up. Have a clean answer ready.

### Bad versions
- ❌ "It's exhausting." (burnout signal)
- ❌ "I need a stable income." (mercenary)
- ❌ "[Project] didn't work out." (failure framing)

### Good version (adapt to your situation)
> "[Current work] taught me that I love the operator work — building reliable systems, owning production, the puzzle of making things work at scale. What I miss is the leverage of a team — being a force multiplier for other engineers, learning from senior peers, and tackling problems too big for one person. [Target role]'s setup is exactly the operating model I want: real ownership at the [appropriate scope] level, but with peers to think alongside. And the technical depth — [name 3-4 things from JD] — is the work I want to be doing for the next several years."

Wins because:
- **Positive framing** (loved current, want the next thing)
- **Specific about what's missing** (team leverage, peers)
- **Specific about what target role offers** (referenced from JD)
- **Forward-looking**

---

## Hiring-manager-stage signals

In the final round, the hiring manager is assessing **culture fit + autonomy ceiling**:

1. **Genuine curiosity about their problems.** Reference JD specifics — name the actual tech, the actual team model, the actual rhythm they mentioned.
2. **Realistic self-assessment.** Don't claim expertise you don't have. "I haven't operated X, I'd ramp on Y-style concepts" beats bluffing.
3. **A view on the work, not just the role.** Have opinions.
4. **Calm under pressure questions.** Defensive = bad. Curious = good. "Good challenge — at that scale we used X, but if I'd had your throughput, Y would have been the right call."

---

## Quick self-test

Can you tell each in 2-3 minutes, with clear Situation / Action / Result, tied to the role you're interviewing for?

1. "Walk me through something you're proud of."
2. "Tell me about a production incident you handled."
3. "Tell me about a technical decision you made and the trade-offs."
4. "Tell me about a time you failed or made a mistake."
5. "Why are you considering leaving your current setup?"

Yes to all five = ready for final round.

---

## What NOT to say (global)

- ❌ "I don't really know"
- ❌ "We…" repeatedly (hides what *you* did)
- ❌ "It was a team effort" (correct but say what YOU contributed)
- ❌ Numbers that sound made up
- ❌ Buzzword bingo
- ❌ Bashing previous tools/teams
- ❌ Long stories without a point — keep STAR tight, 2-3 min max
