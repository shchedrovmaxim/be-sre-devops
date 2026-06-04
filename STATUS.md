# STATUS — Where we are right now

> Read this first at the start of every session.

**Last updated:** 2026-06-04
**Last session:** K8s scheduling (continued) — deepened `topologySpreadConstraints` section: maxSkew math diagrams, layered constraints, `matchLabelKeys` rolling-update fix, `minDomains`, `nodeAffinityPolicy/nodeTaintsPolicy`, cluster-wide defaults, real-world patterns; added 1 self-test drill on the rollout deadlock

---

## Current focus

Closing gaps from a recent senior SRE interview rejection:

1. **Linux/K8s internals** — cgroups memory mgmt; taints/affinity/anti-affinity correlation
2. **Architecture mindset** — simplicity-first; human dimension; CI as architecture
3. **DevOps ecosystem** — Let's Encrypt, multi-stage builds, image automation tooling

Plus broadening to full senior-SRE shape across observability, EKS, Terraform, networking.

---

## Next up (pick one to start next session)

Mixed across areas for variety. Don't pick two from the same area on consecutive sessions. Last session was **K8s** (scheduling), so next session shouldn't be K8s. Two rejection-gap topics are still uncovered — CI/CD and Architecture — both should come soon.

### Option A — CI/CD: multi-stage Docker builds
- **File:** `topics/ci-cd/multi-stage-builds.md` (to write)
- **Why:** explicit rejection feedback gap.
- **Goal:** write a multi-stage Dockerfile with BuildKit cache mounts; compare image sizes; understand distroless.
- **Self-test:** "Why is multi-stage with distroless final image more secure than single-stage on `node:20`?"

### Option B — Architecture: KISS / YAGNI / justified complexity
- **File:** `topics/architecture/simplicity.md` (to write)
- **Why:** rejection feedback flagged over-engineering; this is the senior mindset shift.
- **Goal:** practice answering "design X" with the simplest viable approach first; learn when to add complexity.
- **Self-test:** "Why might 'just a CronJob and a script' be a better answer than 'Argo Events + custom CRD'?"

### Option C — Observability: SLI/SLO/SLA + error budgets
- **File:** `topics/observability/sli-slo-slas.md` (to write)
- **Why:** classic senior SRE territory; comes up in nearly every senior interview.
- **Goal:** define SLI/SLO/SLA cleanly; understand error budgets and how they drive engineering velocity; pick SLIs that actually matter.
- **Self-test:** "How would you set up an SLO for a service that has both a latency requirement and an availability requirement? What's the error budget policy?"

### Option D — Linux (queued for later): conntrack + UDP DNS implications
- **File:** `topics/linux/conntrack-dns.md` (to write)
- **Why:** classic senior gotcha — "intermittent DNS resolution fails under high load" → conntrack table full. Touches kernel networking + K8s CoreDNS + iptables.
- **Goal:** explain `nf_conntrack_max`, why UDP DNS is the canary for table saturation, and how to mitigate (NodeLocal DNS Cache, raising the limit).
- **Self-test:** "5% of pods get intermittent DNS NXDOMAIN under load. Walk me through how you'd diagnose."
- **Rotation note:** queued for later — fine to pick next since the last session was K8s, not Linux.

---

## Recent sessions

| Date | Topic | Area | Status |
|---|---|---|---|
| 2026-06-04 | K8s scheduling (continued) — deepened topologySpreadConstraints: maxSkew math, layered constraints, matchLabelKeys, minDomains, nodeAffinityPolicy | K8s | ✅ |
| 2026-06-03 | K8s scheduling — taints, affinity, anti-affinity, topology spread; 3-question mental model; "won't schedule" troubleshooting flow | K8s | ✅ |
| 2026-06-02 | cgroups CPU — `cpu.max`, CFS quota+period, throttling vs starvation, GOMAXPROCS trap, limits debate (+ simple companion w/ buffet analogy) | Linux | ✅ |
| 2026-06-01 | cgroups memory — `memory.max`, OOM mechanics, `oom_score_adj` + QoS, hands-on Docker walkthrough | Linux | ✅ |
| (prior) | Gravitee — simple version + deep dive reference | Gravitee | ✅ |
| (prior) | Istio — full reference (architecture, CRDs, mTLS, troubleshooting, 26 Q&A) | Istio | ✅ |

---

## Rotation history (last 5 area focuses)

`Istio → Gravitee → Linux (memory) → Linux (CPU) → K8s (scheduling)`

Rotation now clean again. Next session should not be K8s. CI/CD and Architecture are the remaining rejection-gap areas — prioritize one of them.

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
