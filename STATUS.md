# STATUS — Where we are right now

> Read this first at the start of every session.

**Last updated:** 2026-06-05
**Last session:** Architecture — simplicity / KISS / YAGNI / 4-dimensions framework / ADRs / runbooks / 3 design-question drills. Two docs (simple "moat-instead-of-a-fence" analogy + deep-dive reference with curated further-reading section). **🎉 All 3 rejection-gap areas now closed** (Linux internals, DevOps ecosystem, Architecture mindset).

---

## Current focus

Closing gaps from a recent senior SRE interview rejection:

1. **Linux/K8s internals** — cgroups memory mgmt; taints/affinity/anti-affinity correlation
2. **Architecture mindset** — simplicity-first; human dimension; CI as architecture
3. **DevOps ecosystem** — Let's Encrypt, multi-stage builds, image automation tooling

Plus broadening to full senior-SRE shape across observability, EKS, Terraform, networking.

---

## Next up (pick one to start next session)

Rejection-gap area is fully covered. Now broadening across the senior-SRE curriculum. Don't pick Architecture next. Strongest unaddressed areas are Observability, AWS/EKS internals, and Networking — pick one of A/B/C.

### Option A — Observability: SLI/SLO/SLA + error budgets (Recommended — most-asked senior topic)
- **File:** `topics/observability/sli-slo-slas.md` (to write)
- **Why:** classic senior SRE territory; comes up in nearly every senior interview. Tied directly to the 4-dimensions framework's Operations dimension you just learned.
- **Goal:** define SLI/SLO/SLA cleanly; understand error budgets and how they drive engineering velocity; pick SLIs that actually matter.
- **Self-test:** "How would you set up an SLO for a service with both a latency and availability requirement? What's the error budget policy?"

### Option B — AWS / EKS: Karpenter vs Cluster Autoscaler + IRSA token exchange
- **File:** `topics/aws-eks/karpenter-irsa.md` (to write)
- **Why:** AWS is your strong area per interview feedback, but Karpenter and IRSA *internals* often come up at senior level. Karpenter has eaten Cluster Autoscaler in modern EKS.
- **Goal:** explain Karpenter's NodePool / EC2NodeClass model + how it differs from CAS; walk through the IRSA token exchange end-to-end (pod → projected SA token → STS:AssumeRoleWithWebIdentity → AWS creds).
- **Self-test:** "Walk me through what happens when a pod with an IRSA-annotated ServiceAccount calls `aws s3 ls`."

### Option C — Linux: conntrack + UDP DNS implications
- **File:** `topics/linux/conntrack-dns.md` (to write)
- **Why:** classic senior gotcha — "intermittent DNS resolution fails under high load" → conntrack table full. Touches kernel networking + K8s CoreDNS + iptables.
- **Goal:** explain `nf_conntrack_max`, why UDP DNS is the canary for table saturation, and how to mitigate (NodeLocal DNS Cache, raising the limit).
- **Self-test:** "5% of pods get intermittent DNS NXDOMAIN under load. Walk me through how you'd diagnose."

### Option D — Certificates: Let's Encrypt + cert-manager + ACME
- **File:** `topics/certificates/letsencrypt-cert-manager.md` (to write)
- **Why:** explicit rejection-feedback gap (Let's Encrypt); cert-manager is K8s-native and runs in nearly every cluster.
- **Goal:** ACME protocol basics; HTTP-01 vs DNS-01 vs TLS-ALPN-01; when DNS-01 is required (wildcards); Issuer / ClusterIssuer / Certificate CRDs; rate limits; staging environment.
- **Self-test:** "When would you use DNS-01 instead of HTTP-01, and what infra setup does each need?"

---

## Recent sessions

| Date | Topic | Area | Status |
|---|---|---|---|
| 2026-06-05 | Architecture — simplicity/KISS/YAGNI, Ousterhout complexity, 4-dimensions framework, ADRs, runbooks, 3 design-question drills (simple + deep-dive w/ further reading) | Architecture | ✅ |
| 2026-06-04 | K8s pod lifecycle — graceful shutdown, SIGTERM, preStop, probes, PDB, init containers, native sidecars (simple + deep-dive) | K8s | ✅ |
| 2026-06-04 | CI/CD — multi-stage Docker builds + BuildKit cache mounts + distroless (deep + simple companion); security framing via Wiz/Kyverno bridge | CI/CD | ✅ |
| 2026-06-04 | K8s scheduling (continued) — deepened topologySpreadConstraints: maxSkew math, layered constraints, matchLabelKeys, minDomains, nodeAffinityPolicy | K8s | ✅ |
| 2026-06-03 | K8s scheduling — taints, affinity, anti-affinity, topology spread; 3-question mental model; "won't schedule" troubleshooting flow | K8s | ✅ |
| 2026-06-02 | cgroups CPU — `cpu.max`, CFS quota+period, throttling vs starvation, GOMAXPROCS trap, limits debate (+ simple companion w/ buffet analogy) | Linux | ✅ |
| 2026-06-01 | cgroups memory — `memory.max`, OOM mechanics, `oom_score_adj` + QoS, hands-on Docker walkthrough | Linux | ✅ |
| (prior) | Gravitee — simple version + deep dive reference | Gravitee | ✅ |
| (prior) | Istio — full reference (architecture, CRDs, mTLS, troubleshooting, 26 Q&A) | Istio | ✅ |

---

## Rotation history (last 8 area focuses)

`Istio → Gravitee → Linux (memory) → Linux (CPU) → K8s (scheduling) → CI/CD (multi-stage) → K8s (lifecycle) → Architecture (simplicity)`

Rotation clean. Next session should not be Architecture. **All 3 rejection-gap areas are now closed.** From here, focus shifts to **breadth across the senior-SRE curriculum** — Observability, AWS/EKS internals, Networking, Terraform, Secrets, Certificates are all completely untouched.

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
