# be-sre-devops

A study repo for becoming a senior SRE/DevOps engineer. Built around a personal preparation plan after a real interview rejection surfaced specific knowledge gaps.

## Structure

```
topics/          — Study material per topic area, each with a README index
interview-prep/  — STAR stories, troubleshooting drills, mock questions
notes/           — Daily personal notes (YYYY-MM-DD-topic.md)
STATUS.md        — Current state: last session, what's queued next
PROGRESS.md      — Full topic checklist with status markers
CLAUDE.md        — Instructions for AI co-pilot sessions
```

## How to use this repo

### As a study reference
Browse `topics/` topic by topic. Each topic folder has a README that orders the sub-topics and links to deeper material.

### As a session co-pilot with AI tools
This repo is designed to work with AI tools like Claude Code. Every session:

1. AI reads `STATUS.md` to see where the last session ended.
2. AI proposes 2-3 next-topic options from `STATUS.md`'s "Next up" list (mixed across areas to avoid burnout).
3. Session covers 1-2 topics; user does hands-on work and writes notes in their own words.
4. AI updates `STATUS.md` and `PROGRESS.md` at the end so the next session can resume cleanly.

The repo is its own context: no external memory required to continue across sessions.

### As a personal knowledge base
Long-term, the `topics/` content becomes a reference for actual SRE work, not just interview prep.

## Topic areas

| Area | Focus |
|---|---|
| [Linux](topics/linux/) | cgroups, namespaces, networking, conntrack |
| [Kubernetes](topics/kubernetes/) | scheduling, lifecycle, RBAC, networking, storage |
| [Istio](topics/istio/) | service mesh (mostly complete) |
| [Gravitee](topics/gravitee/) | API management (mostly complete) |
| [CI/CD](topics/ci-cd/) | multi-stage builds, ArgoCD, image security, image updaters |
| [Certificates](topics/certificates/) | Let's Encrypt, cert-manager, ACME |
| [Secrets](topics/secrets/) | Sealed Secrets, External Secrets, Vault |
| [Architecture](topics/architecture/) | KISS, ADRs, runbooks, the 4 dimensions |
| [Observability](topics/observability/) | Prometheus, OTel, Grafana, SLOs |
| [AWS / EKS](topics/aws-eks/) | Karpenter, IRSA, NLB/ALB |
| [Terraform](topics/terraform/) | state, modules, drift detection |
| [Networking](topics/networking/) | CNI, NetworkPolicy, eBPF |

## Philosophy

- **Topic rotation** — alternate between areas across sessions to avoid burnout and improve retention.
- **Hands-on every session** — reading without doing = no retention.
- **Out-loud self-test** — interview answers happen out loud, so practice that way.
- **Notes in own words** — copy-pasting from docs doesn't lock in understanding.
- **Senior mindset** — for every architecture answer, address all 4 dimensions: Tech, People, CI/CD, Operations.

## Current status

See [`STATUS.md`](STATUS.md) for current focus and [`PROGRESS.md`](PROGRESS.md) for the full checklist.
