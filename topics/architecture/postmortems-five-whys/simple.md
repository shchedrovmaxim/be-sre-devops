# Blameless postmortems + 5 whys — the simple version (the broken-staircase analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes obvious.

This doc only explains **one idea**:

> **A blameless postmortem finds the broken staircase, not the person who tripped on it. The 5 whys is how you trace the broken staircase back to its root.**

That's it. Everything else (the Google SRE template, action item tracking, the "theater" anti-pattern) is precision on top.

---

## The broken staircase analogy

The office staircase has a broken step. Bob trips on it, drops his laptop, breaks production.

Two responses:

**The blame response**: "Bob should have watched where he was going. Bob gets a warning."

Result: The step is still broken. The next person to trip is Carol. Carol gets a warning. Nothing changes. Production keeps breaking.

**The blameless response**: "Why did production break? Bob tripped. Why did Bob trip? The step is broken. Why is the step broken? Facilities hasn't done a safety audit in 6 months. Why not? There's no policy requiring it. Fix: add a quarterly safety audit to the facilities checklist."

Result: The step gets fixed. No one trips on it again. Bob doesn't feel punished for walking up stairs.

The blameless framing isn't about being soft on failures. It's about identifying **the staircase** (the system) rather than **the tripper** (the person) — because fixing the staircase prevents all future trips, not just Bob's.

---

## The one mental flip

> "Bob missed the alert" → "The alert didn't page Bob in time."

Same situation. Different framing. The first puts the failure on Bob. The second puts the failure on the alerting system — something that can be fixed.

Every time you hear yourself say "X should have..." in a postmortem, stop. Reframe as: "The system didn't make X's correct action the easiest action."

That's what "blameless" means in practice. Not "no one is accountable" — but "the root cause is almost never a human, it's the system that made the human's mistake possible or inevitable."

---

## The 5 whys — how you trace the staircase

Start with the symptom. Ask why. Take the answer. Ask why again. Repeat until you hit a system-level root cause you can actually fix.

**Example: production database connection pool exhausted, service went down for 20 minutes.**

| Why # | Question | Answer |
|---|---|---|
| 1 | Why did the service go down? | Connection pool exhausted — no connections available |
| 2 | Why did the pool exhaust? | A slow query started running at 14:00 holding connections open |
| 3 | Why was a slow query running? | A deploy at 13:55 changed a query that was missing an index |
| 4 | Why wasn't the missing index caught? | The staging database has 100 rows; production has 10M — the slow query only appeared under load |
| 5 | Why does staging have so little data? | No one ever set up a data generation process for staging |

**Root cause**: staging data volume doesn't reflect production. **Fix**: add a staging data refresh process + a query execution plan check in CI (EXPLAIN on any new query).

You could have stopped at Why #1 ("add more connections to the pool") — that's the patch, not the fix. You could have stopped at Why #2 ("Bob deployed a bad query") — that's blaming the tripper. Why #5 is the system fix.

---

## The 3 questions you ask in any postmortem

1. **What happened?** (symptoms, timeline, impact — the facts)
2. **Why did it happen?** (5 whys down to a system-level root cause — not "human error")
3. **What prevents recurrence?** (action items with owners and due dates)

That's the whole postmortem in three questions.

---

## The trap: stopping at "human error"

The most common postmortem failure: "Root cause: human error. Bob clicked the wrong button."

Human error is never the root cause. It's always the symptom of one of:
- **Missing safeguard**: the system allowed the wrong action. (Why could Bob click that button in prod?)
- **Missing signal**: Bob didn't have the information to know it was wrong. (Why didn't the UI warn him?)
- **Missing process**: there was no check that would have caught it. (Why was there no staging test?)
- **Missing recovery**: when the wrong thing happened, there was no fast way to undo it. (Why is there no one-click rollback?)

"Human error" closes the postmortem without learning anything. It's the staircase response, not the system response.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is "blameless"? | Focus on the system, not the person. "The alert didn't fire" not "Bob missed the alert." |
| What is the 5 whys? | Ask why iteratively until you reach a system-level root cause — usually 4-6 iterations. Stop when the answer is a fixable system gap. |
| What's the 5 whys trap? | Stopping at "human error." It's almost never the root cause. |
| What are the 3 things a postmortem produces? | Timeline/impact, root cause (5 whys), action items with owners. |
| What happens to action items? | They get tracked to closure in a ticket system. Untracked action items = postmortem theater. |
| Do near-misses get postmortems? | Yes. A near-miss is a free lesson. The post mortem template applies equally. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through the postmortem process for a P1 incident. What does 'blameless' actually mean, and how do you avoid it becoming theater?"**

**Reference answer (intuitive version):**

"When a P1 hits, the immediate focus is recovery — get service back. The postmortem starts within 24-48 hours while the event is fresh.

The postmortem covers: what happened (timeline of events, impact in user-hours lost or SLO budget burned), why it happened (5 whys down to a system root cause — never stopping at 'human error'), and what we're doing about it (specific action items with a named owner and a due date).

Blameless means the framing focuses on the system, not the person. Instead of 'Bob missed the alert,' it's 'the alert didn't fire in time.' Instead of 'Carol deployed broken code,' it's 'the CI pipeline didn't catch the regression.' The goal is: would a different engineer in the same situation have done the same thing? If yes, the system is the problem, not the engineer.

It becomes theater when action items aren't tracked to closure. You write a beautiful postmortem, five people nod in the meeting, and three months later nothing has changed. The fix is simple: every action item gets a JIRA ticket, an owner, and a due date. The postmortem isn't done until the tickets are resolved."

If that came out clearly, jump to the [deep-dive](./deep-dive.md) for the Google SRE template, the 5-whys technique in detail, the postmortem-as-training-artifact senior framing, and near-miss postmortems.

---

## Further reading / watching

- **Google SRE Book** (free at [sre.google/books](https://sre.google/books/)) — Chapter 15: Postmortem Culture: Learning from Failure. This is the canonical source.
- **Google postmortem template** — search "Google SRE postmortem template." The template itself is the fastest way to understand the structure.
- **John Allspaw — "Blameless Postmortems and Just Culture"** — the original Etsy blog post (2012) that popularized the term. Search the title.

---

## Next: the deep-dive

When the broken-staircase analogy and the three-questions structure feel obvious, jump to [`postmortems-five-whys.md`](./deep-dive.md). The deep-dive covers:

- The Google SRE postmortem template in full (all 7 sections)
- Why postmortems are "blameless" without being "accountability-free"
- The 5 whys technique: worked example, when to stop, the branching variant
- The "postmortem as training artifact" framing — how a good postmortem prevents the next 10 incidents
- How to run a postmortem meeting that doesn't become theater
- Tracking action items to closure
- Near-miss and non-incident postmortems
- The 4-dimensions framing and self-test drills
