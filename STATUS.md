# STATUS — Where we are right now

> Read this first at the start of every session.

**Last updated:** 2026-06-02
**Last session:** Linux — cgroups CPU deep dive (CFS quota+period, throttling vs starvation, GOMAXPROCS trap, limits-debate, hands-on)

---

## Current focus

Closing gaps from a recent senior SRE interview rejection:

1. **Linux/K8s internals** — cgroups memory mgmt; taints/affinity/anti-affinity correlation
2. **Architecture mindset** — simplicity-first; human dimension; CI as architecture
3. **DevOps ecosystem** — Let's Encrypt, multi-stage builds, image automation tooling

Plus broadening to full senior-SRE shape across observability, EKS, Terraform, networking.

---

## Next up (pick one to start next session)

Mixed across areas for variety. Don't pick two from the same area on consecutive sessions. Last **two** sessions were **Linux** (cgroups memory + cgroups CPU), so next session should DEFINITELY NOT be Linux — rotate hard.

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

### Option C — K8s scheduling: taints + affinity correlation
- **File:** `topics/kubernetes/scheduling-taints-affinity.md` (to write)
- **Why:** rejection feedback explicitly called out the correlation between taints, affinity, anti-affinity.
- **Goal:** clear mental model — "can pod run here (taints) → does pod want here (affinity) → spread evenly (topology)".
- **Self-test:** "Pod won't schedule despite matching nodeSelector — walk me through what to check, in order."

### Option D — Linux (queued for later): conntrack + UDP DNS implications
- **File:** `topics/linux/conntrack-dns.md` (to write)
- **Why:** classic senior gotcha — "intermittent DNS resolution fails under high load" → conntrack table full. Touches kernel networking + K8s CoreDNS + iptables.
- **Goal:** explain `nf_conntrack_max`, why UDP DNS is the canary for table saturation, and how to mitigate (NodeLocal DNS Cache, raising the limit).
- **Self-test:** "5% of pods get intermittent DNS NXDOMAIN under load. Walk me through how you'd diagnose."
- **Rotation note:** queued. Do not pick consecutively after a Linux session.

---

## Recent sessions

| Date | Topic | Area | Status |
|---|---|---|---|
| 2026-06-02 | cgroups CPU — `cpu.max`, CFS quota+period, throttling vs starvation, GOMAXPROCS trap, limits debate | Linux | ✅ |
| 2026-06-01 | cgroups memory — `memory.max`, OOM mechanics, `oom_score_adj` + QoS, hands-on Docker walkthrough | Linux | ✅ |
| (prior) | Gravitee — simple version + deep dive reference | Gravitee | ✅ |
| (prior) | Istio — full reference (architecture, CRDs, mTLS, troubleshooting, 26 Q&A) | Istio | ✅ |

---

## Rotation history (last 4 area focuses)

`Istio → Gravitee → Linux (memory) → Linux (CPU)`

⚠️ Two consecutive Linux sessions — by user request, not by mistake. Next session **must** be non-Linux. CI/CD, Architecture, or K8s scheduling are the highest-leverage picks given the rejection feedback.

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
