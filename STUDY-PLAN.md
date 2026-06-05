# STUDY-PLAN — 6 weeks to interview-ready

> **Pace**: 1.5 h/day, 7 days/week. Total: ~63 hours.
> **Output**: by day 42, you've read every simple companion, drilled every killer question out loud, done 2 mock interviews, and shored up your weak spots.
> **Flexibility**: if you miss a day, slip the next day forward — don't double up. The plan has 1 light day per week as built-in buffer.

> If you only have 1 h/day, stretch each week to 9-10 days → ~8-week plan. Same content, slower pace.

---

## Each day has the same shape

| Block | Time | What |
|---|---|---|
| **Read** | 30-45 min | Read the day's listed simple companions (5-8 min each). |
| **Drill** | 30 min | For each topic, answer the killer question **out loud**, no peeking. Then compare to the reference answer at the bottom of the doc. |
| **Note** | 15-30 min | In `notes/YYYY-MM-DD-recap.md`: 3 bullets of what clicked, 1 bullet of what's still fuzzy. The act of writing locks it in. |

**Out loud is non-negotiable.** Interview answers happen out loud, so practice happens out loud. Talk to your monitor, your dog, the ceiling — but vocalize.

---

## Phase overview

| Phase | Weeks | Goal |
|---|---|---|
| **1 — First-pass intuition** | 1-3 | Read all 80+ simple companions, build the mental model for every topic |
| **2 — Apply + mocks** | 4-5 | Mock interview 1, address weak areas, deep-dives on what you fumbled |
| **3 — Polish** | 6 | Mock interview 2, behavioral, final drilling |

---

## Progress checklist

Tick each day when complete. Each day, also write a recap in `notes/YYYY-MM-DD.md` (template at `notes/_daily-template.md`). At end of each week, write `notes/week-N-checkin.md`. Append to `notes/weak-spots.md` whenever you fumble.

### Week 1 — Linux + K8s scheduling foundation
- [ ] **Day 1** — Re-anchor: cgroups + 3-question scheduling flow
- [ ] **Day 2** — Linux: cgroups v1-vs-v2 + IO + namespaces
- [ ] **Day 3** — Linux: runc + iptables
- [ ] **Day 4** — Linux: systemd + ulimits
- [ ] **Day 5** — K8s scheduling deep + priority + preemption
- [ ] **Day 6** — Mid-week catch-up + first hands-on
- [ ] **Day 7** — Light: weakest Linux topic deep-dive

### Week 2 — K8s workloads, lifecycle, networking, storage
- [ ] **Day 8** — Pod lifecycle (the killer interview topic)
- [ ] **Day 9** — Workloads: Deployment + StatefulSet
- [ ] **Day 10** — Workloads: DaemonSet + Job/CronJob + HPA
- [ ] **Day 11** — Resource management
- [ ] **Day 12** — K8s Networking: Services, Endpoints, Ingress/Gateway
- [ ] **Day 13** — K8s Storage: CSI + StorageClass + PVC lifecycle
- [ ] **Day 14** — Light: Volume expansion + snapshots

### Week 3 — K8s security/ops + Certificates + Secrets
- [ ] **Day 15** — RBAC + ServiceAccounts
- [ ] **Day 16** — PSS + securityContext + image pull secrets
- [ ] **Day 17** — etcd + control plane
- [ ] **Day 18** — Upgrades + kubeadm-vs-managed
- [ ] **Day 19** — Certificates
- [ ] **Day 20** — Secrets
- [ ] **Day 21** — Light: Vault PKI + cert rotation + secrets comparison

### Week 4 — CI/CD + AWS + Mock Interview 1
- [ ] **Day 22** — Multi-stage + image scanning + SBOM
- [ ] **Day 23** — Cosign + Kyverno + OPA
- [ ] **Day 24** — ArgoCD foundation
- [ ] **Day 25** — ArgoCD advanced
- [ ] **Day 26** — AWS/EKS core
- [ ] **Day 27** — AWS networking + ops
- [ ] **Day 28** — 🎯 **MOCK INTERVIEW 1**

### Week 5 — Observability + Architecture + Terraform + Networking
- [ ] **Day 29** — Address mock weak areas
- [ ] **Day 30** — Observability: SLOs + Prometheus
- [ ] **Day 31** — PromQL + recording rules + cardinality
- [ ] **Day 32** — OTel + long-term storage
- [ ] **Day 33** — Architecture senior mindset
- [ ] **Day 34** — Terraform
- [ ] **Day 35** — Networking

### Week 6 — Drill, mock, finalize
- [ ] **Day 36** — Drill day 1 (Linux + K8s + CI/CD)
- [ ] **Day 37** — Drill day 2 (Observability + AWS + Architecture + others)
- [ ] **Day 38** — Deep-dive on weakest 3 topics
- [ ] **Day 39** — Architecture design drill (3 scenarios)
- [ ] **Day 40** — Behavioral + STAR practice
- [ ] **Day 41** — 🎯 **MOCK INTERVIEW 2**
- [ ] **Day 42** — Final polish + plan beyond

---

# PHASE 1 — Foundation & First-pass intuition

## Week 1 — Linux internals + K8s scheduling foundation

### Day 1 — Re-anchor: cgroups + the 3-question scheduling flow

- **Read:** [`linux/cgroups-memory.md`](./topics/linux/cgroups-memory/deep-dive.md) (re-read top sections — bridge from `kubectl describe pod`), [`linux/cgroups-cpu-simple.md`](./topics/linux/cgroups-cpu/simple.md), [`kubernetes/scheduling-affinity.md`](./topics/kubernetes/scheduling-affinity/deep-dive.md) (the 3-question flow only).
- **Drill out loud:**
  - "Walk me through what happens at the cgroup level when a pod exceeds its memory limit."
  - "A pod averages 30% CPU but feels slow. What's happening?"
  - "Walk me through what to check, in order, when a pod won't schedule."

### Day 2 — Linux: cgroups v1-vs-v2 + IO + namespaces

- **Read:** [`cgroups-v1-vs-v2-simple.md`](./topics/linux/cgroups-v1-vs-v2/simple.md), [`cgroups-io-simple.md`](./topics/linux/cgroups-io/simple.md), [`namespaces-simple.md`](./topics/linux/namespaces/simple.md).
- **Drill out loud:**
  - "Why did cgroups v2 win, and what's the unified hierarchy?"
  - "Walk me through what each Linux namespace isolates."

### Day 3 — Linux: runc + iptables

- **Read:** [`runc-internals-simple.md`](./topics/linux/runc-internals/simple.md), [`iptables-nftables-simple.md`](./topics/linux/iptables-nftables/simple.md).
- **Drill out loud:**
  - "From config.json + rootfs, what syscalls does runc make?"
  - "Walk me through how kube-proxy iptables routes a Service IP to a pod IP."

### Day 4 — Linux: systemd + ulimits

- **Read:** [`systemd-basics-simple.md`](./topics/linux/systemd-basics/simple.md), [`ulimits-fds-simple.md`](./topics/linux/ulimits-fds/simple.md).
- **Drill out loud:**
  - "Kubelet is misbehaving. How do you diagnose at the systemd layer?"
  - "Service is crashing with 'too many open files'. Walk through the layers."

### Day 5 — K8s scheduling deep + priority + preemption

- **Read:** [`scheduling-affinity.md`](./topics/kubernetes/scheduling-affinity/deep-dive.md) (full deep-dive), [`priority-preemption-simple.md`](./topics/kubernetes/priority-preemption/simple.md).
- **Drill out loud:**
  - "GPU node tainted, pod has toleration and nodeAffinity for amd64, node is arm64. Will it schedule?"
  - "Pod anti-affinity deadlocks rolling updates. Why? Fix?"
  - "Explain priorityClass + preemption. When does it kick in?"

### Day 6 — Mid-week catch-up + first hands-on

- **Hands-on:** redo the cgroups memory + CPU OOM/throttle demo (Docker Desktop + nsenter1 from cgroups-memory.md and cgroups-cpu.md). Watch `memory.events` and `cpu.stat` change live.
- **Drill out loud:** all 12 killer questions from days 1-5 in sequence. Time yourself.

### Day 7 — Light: weakest Linux topic deep-dive

- **Read:** whichever Linux deep-dive (not the simple) felt fuzziest from days 2-4. Just one.
- **Note:** write a `notes/2026-XX-XX-weak-spots.md` listing every topic where you fumbled. Keep updating it.

---

## Week 2 — K8s workloads, lifecycle, networking, storage

### Day 8 — Pod lifecycle (the killer interview topic)

- **Read:** [`pod-lifecycle-simple.md`](./topics/kubernetes/pod-lifecycle/simple.md), [`pod-lifecycle.md`](./topics/kubernetes/pod-lifecycle/deep-dive.md) (full deep-dive — this one's worth it).
- **Drill out loud:**
  - "Rolling deploy is dropping requests despite green probes. Walk me through what to check."
  - "What problem do native sidecars (K8s 1.29+) solve?"

### Day 9 — Workloads: Deployment + StatefulSet

- **Read:** [`deployment-rolling-update-simple.md`](./topics/kubernetes/deployment-rolling-update/simple.md), [`statefulset-simple.md`](./topics/kubernetes/statefulset/simple.md).
- **Drill out loud:**
  - "Walk me through what happens when I push a new image tag to a Deployment with maxSurge: 25%, maxUnavailable: 0."
  - "What problems does StatefulSet solve, and what are the gotchas?"

### Day 10 — Workloads: DaemonSet + Job/CronJob + HPA

- **Read:** [`daemonset-simple.md`](./topics/kubernetes/daemonset/simple.md), [`job-cronjob-simple.md`](./topics/kubernetes/job-cronjob/simple.md), [`hpa-simple.md`](./topics/kubernetes/hpa/simple.md).
- **Drill out loud:**
  - "When would you use a DaemonSet, and how does it schedule on tainted nodes?"
  - "HPA isn't scaling under load. Walk me through what to check, in order."

### Day 11 — Resource management

- **Read:** [`resource-mgmt-requests-limits-simple.md`](./topics/kubernetes/resource-mgmt-requests-limits/simple.md).
- **Drill out loud:**
  - "Two Burstable pods on the same node, node hits memory pressure. Who gets evicted first?"
  - "Explain QoS classes and eviction order."

### Day 12 — K8s Networking: Services, Endpoints, Ingress/Gateway

- **Read:** [`services-simple.md`](./topics/kubernetes/services/simple.md), [`endpoints-endpointslices-simple.md`](./topics/kubernetes/endpoints-endpointslices/simple.md), [`ingress-vs-gateway-api-simple.md`](./topics/kubernetes/ingress-vs-gateway-api/simple.md).
- **Drill out loud:**
  - "Walk me through how a request to ClusterIP gets to a pod."
  - "Why is Gateway API replacing Ingress?"

### Day 13 — K8s Storage: CSI + StorageClass + PVC lifecycle

- **Read:** [`csi-architecture-simple.md`](./topics/kubernetes/csi-architecture/simple.md), [`storageclass-simple.md`](./topics/kubernetes/storageclass/simple.md), [`pvc-pv-lifecycle-simple.md`](./topics/kubernetes/pvc-pv-lifecycle/simple.md).
- **Drill out loud:**
  - "What's WaitForFirstConsumer vs Immediate volume binding, and when does it matter?"
  - "I deleted a PVC and my StatefulSet pod can't come back. What happened?"

### Day 14 — Light: Volume expansion + snapshots

- **Read:** [`volume-expansion-simple.md`](./topics/kubernetes/volume-expansion/simple.md), [`volume-snapshots-simple.md`](./topics/kubernetes/volume-snapshots/simple.md).
- **Drill out loud:**
  - "PVC expanded but didn't grow. What's blocking it?"

---

## Week 3 — K8s security/ops + Certificates + Secrets

### Day 15 — RBAC + ServiceAccounts

- **Read:** [`rbac-simple.md`](./topics/kubernetes/rbac/simple.md), [`serviceaccounts-simple.md`](./topics/kubernetes/serviceaccounts/simple.md).
- **Drill out loud:**
  - "A pod's SA can read Secrets cluster-wide. Audit + lock down — walk me through."
  - "How does a ServiceAccount token get into a pod (modern projected-token model)?"

### Day 16 — PSS + securityContext + image pull secrets

- **Read:** [`pod-security-standards-simple.md`](./topics/kubernetes/pod-security-standards/simple.md), [`security-context-simple.md`](./topics/kubernetes/security-context/simple.md), [`image-pull-secrets-simple.md`](./topics/kubernetes/image-pull-secrets/simple.md).
- **Drill out loud:**
  - "How would you enforce 'restricted' PSS across the cluster?"
  - "Every securityContext setting you'd insist on for a stateless web service."

### Day 17 — etcd + control plane

- **Read:** [`etcd-basics-simple.md`](./topics/kubernetes/etcd-basics/simple.md), [`control-plane-simple.md`](./topics/kubernetes/control-plane/simple.md).
- **Drill out loud:**
  - "Walk me through what you need to know about etcd to be on-call for self-managed K8s."
  - "Walk me through how `kubectl apply` flows through the control plane."

### Day 18 — Upgrades + kubeadm-vs-managed

- **Read:** [`upgrade-flow-simple.md`](./topics/kubernetes/upgrade-flow/simple.md), [`kubeadm-vs-managed-simple.md`](./topics/kubernetes/kubeadm-vs-managed/simple.md).
- **Drill out loud:**
  - "Walk me through a safe K8s minor version upgrade. CP and 100 nodes."
  - "When would you run kubeadm instead of EKS/GKE/AKS?"

### Day 19 — Certificates

- **Read:** [`acme-letsencrypt-simple.md`](./topics/certificates/acme-letsencrypt/simple.md), [`cert-manager-simple.md`](./topics/certificates/cert-manager/simple.md), [`acme-challenges-simple.md`](./topics/certificates/acme-challenges/simple.md).
- **Drill out loud:**
  - "How does Let's Encrypt verify domain ownership?"
  - "When would you use DNS-01 instead of HTTP-01?"

### Day 20 — Secrets

- **Read:** [`sealed-secrets-simple.md`](./topics/secrets/sealed-secrets/simple.md), [`external-secrets-operator-simple.md`](./topics/secrets/external-secrets-operator/simple.md), [`sops-simple.md`](./topics/secrets/sops/simple.md).
- **Drill out loud:**
  - "Why might Sealed Secrets be bad for multi-cluster GitOps?"
  - "Walk through what happens when you create an ExternalSecret referencing AWS Secrets Manager."

### Day 21 — Light: Vault PKI + cert rotation + secrets comparison

- **Read:** [`vault-pki-simple.md`](./topics/certificates/vault-pki/simple.md), [`cert-rotation-simple.md`](./topics/certificates/cert-rotation/simple.md), [`secrets-comparison-simple.md`](./topics/secrets/secrets-comparison/simple.md).
- **Drill out loud:**
  - "Why use Vault PKI for internal mTLS instead of Let's Encrypt?"
  - "Team needs secrets via GitOps. Sealed Secrets vs ESO trade-offs?"

---

# PHASE 2 — Apply + Mock Interview 1

## Week 4 — CI/CD + AWS + Mock Interview 1

### Day 22 — Multi-stage + image scanning + SBOM

- **Read:** [`multi-stage-builds-simple.md`](./topics/ci-cd/multi-stage-builds/simple.md) (refresh), [`image-scanning-simple.md`](./topics/ci-cd/image-scanning/simple.md), [`sbom-syft-simple.md`](./topics/ci-cd/sbom-syft/simple.md).
- **Drill out loud:**
  - "Why is multi-stage + distroless more secure than `FROM node:20`?"
  - "How does Trivy actually find vulnerabilities? Trade-off vs Wiz?"

### Day 23 — Cosign + Kyverno + OPA

- **Read:** [`cosign-sigstore-simple.md`](./topics/ci-cd/cosign-sigstore/simple.md), [`kyverno-simple.md`](./topics/ci-cd/kyverno/simple.md), [`opa-gatekeeper-simple.md`](./topics/ci-cd/opa-gatekeeper/simple.md).
- **Drill out loud:**
  - "Walk me through Cosign keyless signing. What does 'keyless' actually mean?"
  - "Kyverno vs OPA Gatekeeper. When each?"

### Day 24 — ArgoCD foundation

- **Read:** [`argocd-application-appproject-simple.md`](./topics/ci-cd/argocd-application-appproject/simple.md), [`argocd-sync-waves-hooks-simple.md`](./topics/ci-cd/argocd-sync-waves-hooks/simple.md).
- **Drill out loud:**
  - "How does AppProject map to a multi-team setup?"
  - "DB migration before app rollout — how with sync waves + hooks?"

### Day 25 — ArgoCD advanced

- **Read:** [`argocd-ignoredifferences-ssa-simple.md`](./topics/ci-cd/argocd-ignoredifferences-ssa/simple.md), [`argocd-appofapps-applicationset-simple.md`](./topics/ci-cd/argocd-appofapps-applicationset/simple.md), [`argocd-image-updater-simple.md`](./topics/ci-cd/argocd-image-updater/simple.md).
- **Drill out loud:**
  - "HPA modifying replicas vs ArgoCD reverting. How to fix?"
  - "app-of-apps vs ApplicationSet. When each?"

### Day 26 — AWS/EKS core

- **Read:** [`karpenter-vs-cas-simple.md`](./topics/aws-eks/karpenter-vs-cas/simple.md), [`irsa-simple.md`](./topics/aws-eks/irsa/simple.md), [`vpc-cni-simple.md`](./topics/aws-eks/vpc-cni/simple.md).
- **Drill out loud:**
  - "What does Karpenter solve that Cluster Autoscaler doesn't?"
  - "Pod with IRSA-annotated SA calls `aws s3 ls`. Walk me through."

### Day 27 — AWS networking + ops

- **Read:** [`aws-lb-controller-simple.md`](./topics/aws-eks/aws-lb-controller/simple.md), [`eks-upgrades-simple.md`](./topics/aws-eks/eks-upgrades/simple.md), [`external-dns-simple.md`](./topics/aws-eks/external-dns/simple.md).
- **Drill out loud:**
  - "NLB target-type IP vs instance. When + why?"
  - "Zero-downtime EKS minor version upgrade across CP + 100 nodes."

### Day 28 — 🎯 MOCK INTERVIEW 1 (2 h block)

- **Find a partner**: friend who's senior SRE, interviewing.io, Pramp, paid platform.
- **Focus**: K8s + Linux + AWS + general SRE. 60-90 min interview + 30 min retro.
- **Take notes**: every question they asked, every answer you fumbled.
- **Update `notes/weak-spots.md`**: append everything you got wrong or felt unsure on.

---

## Week 5 — Address weaknesses + Observability + Architecture + Terraform + Networking

### Day 29 — Address mock weak areas

- **Re-read** the deep-dive (not simple) for each topic you fumbled in Mock 1.
- **Re-drill** those killer questions until clean.
- **Time-box**: stop at 1.5h even if not done. Continue tomorrow.

### Day 30 — Observability: SLOs + Prometheus

- **Read:** [`slos-simple.md`](./topics/observability/slos/simple.md) (refresh if already done), [`prometheus-architecture-simple.md`](./topics/observability/prometheus-architecture/simple.md).
- **Drill out loud:**
  - "Set up SLOs for a service with both latency + availability requirements. Error budget policy?"
  - "Walk me through Prometheus scrape + relabeling."

### Day 31 — PromQL + recording rules + cardinality

- **Read:** [`promql-essentials-simple.md`](./topics/observability/promql-essentials/simple.md), [`recording-vs-alerting-rules-simple.md`](./topics/observability/recording-vs-alerting-rules/simple.md), [`cardinality-discipline-simple.md`](./topics/observability/cardinality-discipline/simple.md).
- **Drill out loud:**
  - "Compute p99 latency from a histogram with PromQL."
  - "Prometheus is OOMing. What does cardinality explosion mean and how do you find the culprit?"

### Day 32 — OTel + long-term storage

- **Read:** [`otel-collector-simple.md`](./topics/observability/otel-collector/simple.md), [`otel-conventions-sampling-simple.md`](./topics/observability/otel-conventions-sampling/simple.md), [`long-term-storage-simple.md`](./topics/observability/long-term-storage/simple.md).
- **Drill out loud:**
  - "OTel agent vs gateway. Tail vs head sampling."
  - "Thanos vs Mimir vs VictoriaMetrics. When each?"

### Day 33 — Architecture senior mindset

- **Read:** [`simplicity-simple.md`](./topics/architecture/simplicity/simple.md) (refresh), [`postmortems-five-whys-simple.md`](./topics/architecture/postmortems-five-whys/simple.md), [`ci-as-architecture-simple.md`](./topics/architecture/ci-as-architecture/simple.md).
- **Drill out loud:**
  - "Why CronJob + script over Argo Events + custom CRD?"
  - "Walk through a P1 postmortem. What does 'blameless' actually mean?"
  - "Why is CI an architectural concern, not a tooling detail?"

### Day 34 — Terraform

- **Read:** [`remote-state-simple.md`](./topics/terraform/remote-state/simple.md), [`workspaces-vs-separate-state-simple.md`](./topics/terraform/workspaces-vs-separate-state/simple.md), [`modules-simple.md`](./topics/terraform/modules/simple.md), [`drift-detection-simple.md`](./topics/terraform/drift-detection/simple.md).
- **Drill out loud:**
  - "Two engineers run `apply` simultaneously with S3+DynamoDB locking. What happens?"
  - "Why is using `workspace` for prod/staging/dev usually wrong?"

### Day 35 — Networking

- **Read:** [`cni-comparison-simple.md`](./topics/networking/cni-comparison/simple.md), [`network-policy-simple.md`](./topics/networking/network-policy/simple.md), [`kube-proxy-modes-simple.md`](./topics/networking/kube-proxy-modes/simple.md), [`conntrack-dns-simple.md`](./topics/networking/conntrack-dns/simple.md).
- **Drill out loud:**
  - "Why pick Cilium over Calico in 2026?"
  - "5% of pods get intermittent DNS NXDOMAIN under load. Diagnosis?"

---

# PHASE 3 — Polish + Mock Interview 2

## Week 6 — Drill, mock, finalize

### Day 36 — Drill day 1 (cross-area)

- **Drill out loud, no peeking** — for each, answer for ~1 min, then read the reference answer:
  - All 7 Linux killer questions
  - All 5 main K8s killer questions (scheduling, lifecycle, RBAC, services, upgrades)
  - All 3 CI/CD killer questions (multi-stage, ArgoCD sync waves, Cosign)
- 30 sec rest between each. Treat as interview pace.

### Day 37 — Drill day 2 (cross-area)

- All Observability killer questions
- All AWS/EKS killer questions
- All Architecture killer questions
- All Terraform + Networking + Secrets + Certificates killer questions

### Day 38 — Deep-dive on weakest 3 topics

- From `notes/weak-spots.md`, pick the 3 topics where you still fumble.
- Read the **deep-dive** (not simple) version of each.
- Re-drill the killer question until clean.

### Day 39 — Architecture design drill

- Pick 3 design questions:
  1. "Design a notification system for subscription expiry."
  2. "Design a safe ML model rollout system."
  3. "Design on-demand PostgreSQL provisioning for internal teams."
- For each, answer with full **4-dimensions framework** (Tech + People + CI/CD + Operations).
- Time-box 15 min per question. Compare to the reference answers in [`simplicity.md`](./topics/architecture/simplicity/deep-dive.md).

### Day 40 — Behavioral + STAR

- Re-read [`interview-prep/`](./interview-prep) STAR templates (whichever stories are there).
- Practice 4-5 "tell me about a time" questions out loud:
  - "Tell me about a time you disagreed with a senior engineer."
  - "Tell me about an incident you led the response to."
  - "Tell me about a time you simplified an over-engineered system."
  - "Tell me about a project you'd do differently in hindsight."
  - "Why are you leaving your current role?"

### Day 41 — 🎯 MOCK INTERVIEW 2 (2 h block)

- **Comprehensive**: K8s + design + behavioral. Ideally a senior SRE you respect, or paid interview platform.
- **Compare to Mock 1**: are you cleaner? More confident? Hitting all 4 dimensions in design questions?

### Day 42 — Final polish + plan beyond

- **Address Mock 2 weak spots** — final pass.
- **Decide on next steps**:
  - Apply broadly? — yes, you're ready
  - Need another mock? — schedule for next week
  - Specific company prep — read their engineering blog, last 12 posts

---

## Daily reminders (keep these visible)

- **Out loud** is non-negotiable. Vocalize. Always.
- **Time yourself** drilling killer questions. Aim for clean 60-90 sec answers.
- **Update `notes/weak-spots.md`** continuously. It becomes your final-week priority list.
- **Don't read passively**. Read the simple companion, then immediately drill the killer Q. The read primes, the drill locks in.
- **Sleep matters**. Memory consolidation is a sleep function. Don't sacrifice sleep for one more topic.
- **Hands-on > reading** for any topic that has a hands-on (cgroups OOM, CPU throttle, multi-stage builds, etc.). Do it.

## What this plan does NOT cover (and how to plug those gaps)

- **GCP / Azure specifics** — if your target uses those, allocate weekend extra time to read official docs for IAM Workload Identity (GCP) or Managed Identity (Azure).
- **System design at distributed-systems scale** — for staff/principal interviews, read "Designing Data-Intensive Applications" alongside this plan. ~1 chapter/week.
- **Specific company stacks** — last week of any interview prep, read the company's engineering blog (last ~12 months). 30 min/day.
- **Coding/algorithms** — if their role does a coding round, 30 min/day of LeetCode-medium on top of this plan.

## Progress check-ins

At the end of each week, write a 5-bullet self-assessment in `notes/week-N-checkin.md`:
1. What topics am I now confident on?
2. What topics am I still fumbling?
3. What's the single weakest area going into next week?
4. How many killer questions did I answer cleanly this week (out of total drilled)?
5. Adjustment for next week?

If you're consistently behind schedule by day 14, this plan is too aggressive — drop to 1h/day and add 2 weeks at the end.

## When you can stop

You're interview-ready when **all of these** are true:
- You can answer 80%+ of the killer questions cleanly in under 90 sec each.
- You can do a design question hitting all 4 dimensions naturally, without prompting.
- Mock interviewers say "yeah, I'd hire you" without major caveats.
- You have 3-5 STAR stories you can deliver smoothly across behavioral themes.

That's it. Show up, get the offer, ship the work.
