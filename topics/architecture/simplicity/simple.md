# Simplicity — the simple version (the moat-instead-of-a-fence analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes obvious.

This doc only explains **one thing**:

> **The senior failure mode is over-engineering — solving a bigger problem than the one you actually have.**

That's it. The rejection feedback you got was about this exact pattern: "the proposed solution lacked simplicity." This doc is about how to stop doing that.

---

## You need a fence. You built a moat.

Your neighbour asks you to help: their dog keeps escaping the backyard. The dog needs to stay in the yard.

Three responses, ranked by "how senior the engineer is":

| Response | What gets built | What happens later |
|---|---|---|
| **Junior** | A 4-foot wooden fence around the yard. | Done. Dog stays in. Neighbour happy. |
| **Mid-level** | An 8-foot fence with motion sensors, an automated electronic gate, GPS dog collar with mobile alerts. | Done. Dog stays in. Neighbour spends 2 hours a week troubleshooting the gate; collar battery dies; sensors trigger on raccoons; neighbour eventually unplugs everything and gets a regular fence. |
| **Senior over-engineer** | A moat. With water. And a drawbridge. | Dog stays in. Also: neighbour's lawn floods, mosquitoes breed in summer, the drawbridge motor seizes in winter, the HOA sues them, and the moat is now the most expensive thing on the property. |

**The moat solves the problem. The moat also creates 10 new problems.** That's over-engineering. Every solution you ship has an ongoing cost — operational, cognitive, financial, social. Senior engineering is feeling those costs *before* you build the thing.

The mid-level engineer's solution actually *works* on day one. But you should still pick the junior solution, because the dog needed a fence. A 4-foot wooden fence is the **simplest viable answer**.

---

## The single question to ask before adding complexity

Before you propose any solution, ask yourself:

> **"What problem am I actually solving, and who pays the ongoing cost?"**

If you can't answer cleanly, you're moat-building. Examples:

- "Let's add Kafka" → What problem? "Event-driven decoupling." Who pays? "On-call gets 2am pages when the broker's disk fills up." Are you solving a real problem? Or building for an imagined future scale? → **YAGNI** ("You Aren't Gonna Need It").

- "Let's build a custom Kubernetes operator for this" → What problem? "We need to provision X automatically." Who pays? "The next engineer who has to maintain the operator + the CRD + the controller + the RBAC." Could you just write a `CronJob` that runs a script? → **KISS** ("Keep It Stupid Simple").

- "Let's use service mesh for everything" → What problem? "Observability and mTLS." Who pays? "Every engineer has to understand Envoy, every debug session adds an extra hop." Do you actually need mTLS between every internal service? → Probably not all of them.

The question isn't "is this cool?" or "is this what big companies do?" The question is "given the actual problem I have today, is the simplest solution good enough?"

---

## The worked example: CronJob vs Argo Events + custom CRD

This is the killer interview question your rejection touched on.

**The problem**: every night at 2 AM, archive yesterday's data from PostgreSQL to S3.

**The over-engineered answer:**

> "I'd set up Argo Events to listen for a scheduled trigger, then trigger an Argo Workflow that runs a custom CRD-based job. The CRD lets us standardize archive jobs across services. We'd also need a controller to manage the CRD lifecycle..."

**The senior answer:**

> "A K8s `CronJob`. The pod runs a shell script that calls `pg_dump | aws s3 cp`. The script is 30 lines. The CronJob YAML is 20 lines. Total operational footprint: one resource, one image, no controllers, no CRDs, no custom Helm chart. If it fails, you read the pod logs. If you need to debug, you can run the script locally. Onboarding cost for a new engineer: 5 minutes. If we ever need to standardize across many services, we add that complexity *then* — when there's an actual second service that needs it. Right now, there's one."

What makes the senior answer senior:
1. **It solves the actual problem** (archive at 2am) and nothing else.
2. **It uses tools the team already operates** (CronJobs, scripts, AWS CLI).
3. **It's debuggable by anyone** in 5 minutes, not just the original author.
4. **It can be replaced or extended later** without ripping anything out.
5. **It names the cost of the alternative** ("controllers to maintain") instead of just describing it.

If the interviewer pushes back ("but what if we need many of these?"), the senior answer is: **"Then I add the abstraction when there's a second one. Today there's one. The cost of premature abstraction is higher than the cost of duplicating later."**

That's YAGNI in one sentence.

---

## The senior mindset shift

| Mid-level thinking | Senior thinking |
|---|---|
| "What's the most general solution?" | "What's the simplest solution that solves *this* problem?" |
| "What's the best practice?" | "What's the smallest thing I can ship?" |
| "What about scale?" | "What's our actual traffic? What problem am I optimizing for?" |
| "Let's use the cool tool" | "Let's use the boring tool we already operate" |
| "We need this for flexibility later" | "We need this when we actually need it — not before" |
| "Here's the architecture" | "Here's the architecture, the runbook, the on-call story, and how a new engineer gets onboarded in a week" |

The last row is the rejection-feedback verbatim. It's also the meta-pattern: **a senior architectural answer always includes who's going to live with the system, not just what the system is.**

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| "Is this over-engineered?" | Ask: "What's the simplest viable alternative? If I shipped *that* instead, would the problem be solved?" If yes, you're probably over-engineering. |
| "What's KISS?" | "Keep It Stupid Simple." The simplest solution that works. |
| "What's YAGNI?" | "You Aren't Gonna Need It." Don't build for hypothetical future requirements. |
| "When IS more complexity justified?" | When there's a *concrete, current* requirement the simple solution can't meet. Not "we might need it." |
| "How do I argue for the simple solution?" | Name the **ongoing cost** of the complex one. Cost is what people forget. Not building cost — operational cost, learning cost, debugging cost, pager cost. |
| "What if it's not enough later?" | Then you add complexity later, when "later" arrives. You can always add. You can almost never subtract. |

---

## Self-test (one question — the killer one)

Out loud:

> **"You need to archive yesterday's PostgreSQL data to S3 every night. Why might 'just a CronJob and a script' be a better answer than 'Argo Events + a custom CRD'?"**

**Reference answer (intuitive version):**

"Because the problem is small and the simple solution is enough. A CronJob with a 30-line bash script gives me: one resource to manage, one image to maintain, no custom controllers, no CRDs, no Helm chart. If it fails, I read the pod logs. If I need to debug, I can run the script locally. A new engineer onboards in 5 minutes. The Argo Events + CRD solution *also* solves the problem — but it adds three components I now have to operate, learn, document, and page someone about at 2 AM when it breaks. The cost isn't building it; the cost is the next two years of operating it. If I genuinely needed to standardize archives across many services, I'd add that abstraction **when the second service shows up** — not before. Premature abstraction is more expensive than duplicating once. The senior question isn't 'what's the best architecture?' — it's 'what's the simplest thing that solves the actual problem today, and is the cost of the more complex alternative justified by a concrete current need?'"

If you got that out cleanly, you've internalized the discipline. The deep-dive doc adds the formal definitions (Ousterhout's complexity model), the 4-dimensions framework, ADRs, and 3 design-question drills.

---

## Further reading / watching

The single most influential resource on this topic:

- **John Ousterhout — "A Philosophy of Software Design"** (Stanford talk, 60 min, free on YouTube — search the title + "Stanford"). The talk distills the book. If you watch one architecture talk this year, watch this.

For senior-mindset background:

- **Dan McKinley — "Choose Boring Technology"** ([boringtechnology.club](https://boringtechnology.club/)). Short read. Famous for the "innovation tokens" framing — you only get 3, spend them carefully.
- **Google SRE Book** ([sre.google/books](https://sre.google/books/)) — Chapters 1, 6, 17, 22. Free online.

For the rejection-gap specifically:

- **DORA — *Accelerate*** (book by Forsgren / Humble / Kim). The data behind what high-performing teams actually do. The answer is almost always: simpler, faster, smaller batches, more boring tech.

---

## Next: the deep-dive

When the moat analogy and the "what problem am I solving" question feel obvious, jump to [`simplicity.md`](./deep-dive.md). The deep-dive covers:

- KISS vs YAGNI — what each really means and common misuses
- Ousterhout's **complexity model**: change amplification, cognitive load, unknown unknowns
- The 3 sources of complexity: essential, accidental, designed-in
- The **decision framework** for "is this complexity justified?"
- The **4 dimensions framework** (Tech + People + CI/CD + Operations) — the actual deliverable of senior architecture
- **ADRs** (Architecture Decision Records) — template + when to write them
- **Runbooks as design deliverables**, not afterthoughts
- **Onboarding in <1 week** as an explicit design constraint
- **3 design-question drills** answered with full 4-dimension treatment
- 4 self-test drills + further reading

The deep-dive is the reference. This doc is the mental model. The moat analogy is what you should keep in your head during any design discussion.
