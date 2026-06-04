# STATUS — Where we are right now

> Read this first at the start of every session.

**Last updated:** 2026-06-04
**Last session:** K8s pod lifecycle — graceful shutdown, SIGTERM, preStop, probes, PDB, init containers, native sidecars (K8s 1.29+ solving Istio-sidecar-dies-first). Two docs (simple airplane-deplaning analogy + deep-dive reference). Killer interview question covered: "rolling deploy dropping requests despite green probes."

---

## Current focus

Closing gaps from a recent senior SRE interview rejection:

1. **Linux/K8s internals** — cgroups memory mgmt; taints/affinity/anti-affinity correlation
2. **Architecture mindset** — simplicity-first; human dimension; CI as architecture
3. **DevOps ecosystem** — Let's Encrypt, multi-stage builds, image automation tooling

Plus broadening to full senior-SRE shape across observability, EKS, Terraform, networking.

---

## Next up (pick one to start next session)

Mixed across areas for variety. Don't pick two from the same area on consecutive sessions. Last session was **K8s** (pod lifecycle), so next session shouldn't be K8s. **Architecture is still the last uncovered rejection-gap topic** — strong recommendation to pick it next.

### Option A — Architecture: KISS / YAGNI / justified complexity (Recommended — last rejection-gap topic)
- **File:** `topics/architecture/simplicity.md` (to write)
- **Why:** rejection feedback flagged over-engineering; this is the senior mindset shift. After this, all three rejection-gap areas are covered.
- **Goal:** practice answering "design X" with the simplest viable approach first; learn when complexity is justified vs gratuitous.
- **Self-test:** "Why might 'just a CronJob and a script' be a better answer than 'Argo Events + custom CRD'?"

### Option B — Observability: SLI/SLO/SLA + error budgets
- **File:** `topics/observability/sli-slo-slas.md` (to write)
- **Why:** classic senior SRE territory; comes up in nearly every senior interview.
- **Goal:** define SLI/SLO/SLA cleanly; understand error budgets and how they drive engineering velocity; pick SLIs that actually matter.
- **Self-test:** "How would you set up an SLO for a service with both a latency and availability requirement? What's the error budget policy?"

### Option C — Linux: conntrack + UDP DNS implications
- **File:** `topics/linux/conntrack-dns.md` (to write)
- **Why:** classic senior gotcha — "intermittent DNS resolution fails under high load" → conntrack table full. Touches kernel networking + K8s CoreDNS + iptables.
- **Goal:** explain `nf_conntrack_max`, why UDP DNS is the canary for table saturation, and how to mitigate (NodeLocal DNS Cache, raising the limit).
- **Self-test:** "5% of pods get intermittent DNS NXDOMAIN under load. Walk me through how you'd diagnose."

### Option D — AWS / EKS: Karpenter vs Cluster Autoscaler + IRSA token exchange
- **File:** `topics/aws-eks/karpenter-irsa.md` (to write)
- **Why:** AWS is your strong area per interview feedback, but Karpenter and IRSA *internals* often come up at senior level. Karpenter has eaten Cluster Autoscaler in modern EKS.
- **Goal:** explain Karpenter's NodePool / EC2NodeClass model + how it differs from CAS; walk through the IRSA token exchange end-to-end (pod → projected SA token → STS:AssumeRoleWithWebIdentity → AWS creds).
- **Self-test:** "Walk me through what happens when a pod with an IRSA-annotated ServiceAccount calls `aws s3 ls`."

---

## Recent sessions

| Date | Topic | Area | Status |
|---|---|---|---|
| 2026-06-04 | K8s pod lifecycle — graceful shutdown, SIGTERM, preStop, probes, PDB, init containers, native sidecars (simple + deep-dive) | K8s | ✅ |
| 2026-06-04 | CI/CD — multi-stage Docker builds + BuildKit cache mounts + distroless (deep + simple companion); security framing via Wiz/Kyverno bridge | CI/CD | ✅ |
| 2026-06-04 | K8s scheduling (continued) — deepened topologySpreadConstraints: maxSkew math, layered constraints, matchLabelKeys, minDomains, nodeAffinityPolicy | K8s | ✅ |
| 2026-06-03 | K8s scheduling — taints, affinity, anti-affinity, topology spread; 3-question mental model; "won't schedule" troubleshooting flow | K8s | ✅ |
| 2026-06-02 | cgroups CPU — `cpu.max`, CFS quota+period, throttling vs starvation, GOMAXPROCS trap, limits debate (+ simple companion w/ buffet analogy) | Linux | ✅ |
| 2026-06-01 | cgroups memory — `memory.max`, OOM mechanics, `oom_score_adj` + QoS, hands-on Docker walkthrough | Linux | ✅ |
| (prior) | Gravitee — simple version + deep dive reference | Gravitee | ✅ |
| (prior) | Istio — full reference (architecture, CRDs, mTLS, troubleshooting, 26 Q&A) | Istio | ✅ |

---

## Rotation history (last 7 area focuses)

`Istio → Gravitee → Linux (memory) → Linux (CPU) → K8s (scheduling) → CI/CD (multi-stage) → K8s (lifecycle)`

Rotation clean. Next session should not be K8s. **Architecture is still the last unaddressed rejection-gap area** — strongly consider picking it next.

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
