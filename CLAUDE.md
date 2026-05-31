# CLAUDE.md — Read first

This is a public study repo for becoming a senior SRE/DevOps engineer.

## How to start a session

**ALWAYS read `STATUS.md` first.** It tells you:
- What we covered in the last session
- What's queued up next
- The user's current focus

Then read this file (CLAUDE.md) to understand how to operate. Then proceed.

## Context

Preparing for senior SRE/DevOps interviews. A recent interview surfaced specific gaps that this repo is structured to close:

1. **Linux/K8s internals** (cgroups memory management; correlation between taints, affinity, anti-affinity)
2. **Architectural mindset** (over-engineering; neglected human dimension and CI; lack of simplicity)
3. **DevOps ecosystem** (Let's Encrypt, multi-stage builds, ArgoCD Image Updater, Renovate, Dependabot)

Strong areas (confirmed by interview feedback): AWS components, communication.

The user responds best to:
- **Simpler language, fewer concepts at once** (this was reinforced when we redid Istio simpler)
- **Bridging from what they know** ("you did X in code → here's how the infra does it")
- **Concrete examples and analogies**, not abstract definitions
- **Show full picture before moving** — propose plan, get confirmation, then execute

## How content is organized

- `topics/<area>/` — study material per topic area, with a README that indexes sub-topics
- `interview-prep/` — STAR stories, troubleshooting drills, mock questions
- `notes/` — user's daily personal notes (YYYY-MM-DD-topic.md)
- `STATUS.md` — current state, last session, what's next
- `PROGRESS.md` — full topic checklist

## Topic rotation philosophy

**No 30-day blocks. No multi-day deep dives in the same area.** We jump between topic areas to avoid burnout and improve retention.

- Don't do 2 consecutive sessions in the same area unless the user asks.
- Each session covers 1-2 topics; mix K8s/Linux internals with CI/CD or Architecture or Observability.
- Aim for 2-3 different topic areas touched per week.

Areas to rotate between:

1. **Linux** — cgroups, namespaces, networking, conntrack
2. **Kubernetes** — scheduling, lifecycle, RBAC, networking, storage
3. **CI/CD** — multi-stage builds, ArgoCD, image security, image updaters
4. **Certificates** — Let's Encrypt, cert-manager, ACME
5. **Secrets** — External Secrets, Sealed Secrets, Vault
6. **Architecture** — KISS, ADRs, runbooks, 4 dimensions
7. **Observability** — Prometheus, OTel, Grafana, SLOs
8. **AWS/EKS** — Karpenter, IRSA, VPC CNI
9. **Terraform** — state, modules, drift
10. **Networking** — CNI, NetworkPolicy, eBPF
11. **Istio** — largely done; reference for review
12. **Gravitee** — largely done; reference for review

## Session structure (1-2 hours)

| Time | Activity |
|---|---|
| 15 min | Read the resource for today's topic |
| 30-45 min | Hands-on exercise (actually do it, don't just read) |
| 15 min | User writes notes in their own words (the act of writing locks it in) |
| 15 min | Self-test out loud (interview answers happen out loud) |
| 5 min | Update STATUS.md and PROGRESS.md |

## How to teach

- Lead with **why** before **how**.
- Use the **app-level → infra-level** bridge wherever it applies.
- Show **one concrete YAML example** rather than describing the schema in prose.
- Add a **self-test question at the end of each topic** — something the user can answer out loud.
- For senior-level questions, always address the **4 dimensions**: Tech, People, CI/CD, Operations.

## End-of-session checklist (DO NOT SKIP)

1. **Update `STATUS.md`:**
   - Add today's session under "Recent sessions"
   - Update "Last session" date and summary
   - Update "Current focus" if it shifted
   - Update "Next up" — propose 2-3 topic options for next session, mixed across areas (not all from the same area)
   - Update "Rotation history" with today's area

2. **Update `PROGRESS.md`:**
   - Tick `[x]` for any sub-topic fully covered today
   - Use `[~]` if covered but needs more depth

3. **Optionally create a daily note:**
   - If the session produced new lasting content (not already in `topics/`), save to `notes/YYYY-MM-DD-topic.md`
   - Notes are user-style, not Claude-style — short, what they learned, what's still fuzzy

4. **Commit** the changes (suggest message: `study: <topic covered> + status update`).

## What NOT to do

- Don't ignore STATUS.md and start blind.
- Don't dive into the same topic area as last session unless user explicitly asks.
- Don't write lecture-style theory dumps. Use the simple framing.
- Don't end a session without updating STATUS.md.
- Don't propose only one next topic — always offer 2-3 mixed-area options.
