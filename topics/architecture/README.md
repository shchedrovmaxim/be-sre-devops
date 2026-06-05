# Architecture (senior mindset) ⚠️ THE rejection gap

> "Unconvincing Architectural Approach: The proposed solution lacked simplicity. The candidate focused excessively on pure infrastructure, neglecting the human dimension (team training and onboarding) and CI-related aspects."

This is **the most important area in the repo.** Closing it is the difference between mid and senior.

## The senior mindset in one sentence

**Every architecture answer must address 4 dimensions, with simplicity as the default:**
1. **Tech** — what components, what data flow
2. **People** — who runs it, how they learn it, on-call story
3. **CI/CD** — how code reaches prod, how we roll back
4. **Operations** — SLOs, monitoring, runbooks, incident response

If you only talk about Tech, you sound mid-level.

## Recommended order

1. **KISS, YAGNI, justified complexity** — the simplicity discipline
2. **A Philosophy of Software Design (Ousterhout)** — at minimum ch. 1-3
3. **ADRs (Architecture Decision Records)** — Context / Decision / Consequences
4. **Runbooks as deliverables** — not afterthoughts
5. **Onboarding new engineers in <1 week** as a design constraint
6. **Blameless postmortems** (Google SRE book pattern)
7. **The 5 whys** for root cause
8. **CI as architecture** — immutable promotion, trunk-based, feature flags
9. **The 4 dimensions framework** — Tech + People + CI/CD + Operations
10. **Practice** — 3 design questions answered with all 4 dimensions

## Files

| Sub-topic | File | Date covered |
|---|---|---|
| Simplicity — simple version (moat-instead-of-a-fence analogy) | [`simplicity-simple.md`](./simplicity-simple.md) | 2026-06-05 |
| Simplicity — KISS, YAGNI, Ousterhout complexity, 4-dimensions framework, ADRs, runbooks, 3 design-question drills | [`simplicity.md`](./simplicity.md) | 2026-06-05 |

## Why this matters

The rejection feedback was explicit: the proposed solution lacked simplicity and ignored the human/CI dimensions. That's the difference between "good engineer" and "senior engineer who can be trusted to design a system."

## The mental flip

| Mid-level thinking | Senior thinking |
|---|---|
| "Let's set up X" | "Let's set up the simplest thing that solves the actual problem" |
| "Here's the architecture" | "Here's the architecture + the runbook + how a new engineer gets onboarded in a week + how it gets to prod" |
| "We'll add monitoring later" | "Here are the SLOs, here's how we'll know when it's broken, here's the alert that pages someone" |
| "It works in dev" | "It survives a pod restart, a node failure, a region outage, a bad deploy, a junior engineer making a typo" |
| "Use the cool new tool" | "Use the boring tool we already operate; only add complexity if the requirement justifies it" |

## Exercise to do early

Look back at any architectural problem where you've been told you over-engineered. Write a **5-line alternative** that's the simplest viable approach. That's the muscle to train.
