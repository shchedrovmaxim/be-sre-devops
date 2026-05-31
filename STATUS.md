# STATUS — Where we are right now

> Read this first at the start of every session.

**Last updated:** 2026-05-31
**Last session:** Gravitee deep dive (simple version + detailed reference)

---

## Current focus

Closing gaps from a recent senior SRE interview rejection:

1. **Linux/K8s internals** — cgroups memory mgmt; taints/affinity/anti-affinity correlation
2. **Architecture mindset** — simplicity-first; human dimension; CI as architecture
3. **DevOps ecosystem** — Let's Encrypt, multi-stage builds, image automation tooling

Plus broadening to full senior-SRE shape across observability, EKS, Terraform, networking.

---

## Next up (pick one to start next session)

Mixed across areas for variety. Don't pick two from the same area on consecutive sessions.

### Option A — Linux: cgroups deep dive (memory)
- **File:** `topics/linux/cgroups-memory.md` (to write)
- **Why:** rejection feedback called out cgroups memory mgmt explicitly.
- **Goal:** explain at the cgroup level what happens when a pod OOMs.
- **Self-test:** "Walk me through what happens at the cgroup level when a pod exceeds its memory limit."

### Option B — CI/CD: multi-stage Docker builds
- **File:** `topics/ci-cd/multi-stage-builds.md` (to write)
- **Why:** explicit rejection feedback gap.
- **Goal:** write a multi-stage Dockerfile with BuildKit cache mounts; compare image sizes.
- **Self-test:** "Why is multi-stage with distroless final image more secure than single-stage on node:20?"

### Option C — Architecture: KISS / YAGNI / justified complexity
- **File:** `topics/architecture/simplicity.md` (to write)
- **Why:** rejection feedback flagged over-engineering; this is the senior mindset shift.
- **Goal:** practice answering "design X" with the simplest viable approach first; learn when to add complexity.
- **Self-test:** "Why might 'just a CronJob and a script' be a better answer than 'Argo Events + custom CRD'?"

---

## Recent sessions

| Date | Topic | Area | Status |
|---|---|---|---|
| (prior) | Gravitee — simple version + deep dive reference | Gravitee | ✅ |
| (prior) | Istio — full reference (architecture, CRDs, mTLS, troubleshooting, 26 Q&A) | Istio | ✅ |

---

## Rotation history (last 2 area focuses)

`Istio → Gravitee`

Suggestion for next session: any area EXCEPT Istio or Gravitee. Linux / CI/CD / Architecture are the highest-leverage starting areas given the rejection feedback.

---

## Open questions / parked items

- Mock interview cadence — TBD, suggest weekly once core knowledge gaps closed
- Hands-on lab environment — may need a kind/minikube cluster for K8s exercises
- AI/LLM observability — park for now, revisit after fundamentals

---

## Big-picture goal

**Be ready to confidently interview for senior SRE/DevOps roles within 30 days.** That means:
- Linux + K8s internals: walk through cgroups, scheduling, networking without hesitation
- DevOps ecosystem: speak fluently about CI/CD, GitOps, image automation, cert mgmt
- Architecture: propose simple solutions that include people, CI, operations — not just tech
- Communicate it all clearly under pressure
