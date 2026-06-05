# GHA runners + trunk-based dev + Conftest — the simple version (the assembly line)

> Read this first. Once the analogy clicks, the [deep-dive](./gha-runners-trunk-based-conftest.md) is easy.

This doc explains **one idea**:

> **Modern CI/CD is an assembly line with quality checks at every stage. Trunk-based development is the process discipline. Self-hosted runners are the machines for specialized work. Conftest is the policy checker that runs on the line.**

---

## The assembly line analogy

A car assembly line has:
- **Standard workstations** (GitHub-hosted runners) — standard tasks: welding, painting, installing seats. Works for 90% of operations.
- **Specialized stations** (self-hosted runners) — secure clean rooms, specialized tools, proprietary equipment. Used only when the standard station can't do the job (e.g., access to a private database, a GPU workload, specialized hardware).
- **Quality control checkpoints** (Conftest in CI) — before each stage, an inspector checks: "does this part meet spec?" If not, it doesn't proceed. The check runs on the manifests and config files, not just the code.
- **Process discipline** (trunk-based development) — instead of long-lived feature branches (chaos), everyone works on one main branch with feature flags. The line never stops because a branch gets too far ahead.

---

## Self-hosted runners — when you actually need them

GitHub-hosted runners can't:
- Access your VPC-private resources (internal databases, private registries, internal APIs)
- Run with custom hardware (GPU for ML, specialized networking cards)
- Use non-standard OS configurations
- Provide a predictable local build cache across jobs

Self-hosted runners can, but they cost you:
- Infrastructure to run them (EC2, K8s nodes)
- Security surface (the runner runs your code — it has access to your network)
- Operational maintenance

The senior answer: don't use self-hosted runners unless you have a specific reason. When you do, use the **Actions Runner Controller (ARC)** — it runs ephemeral runner pods on Kubernetes, scaling to zero between jobs. No persistent runner VMs to maintain.

```yaml
# GitHub Actions workflow targeting a self-hosted runner
jobs:
  build:
    runs-on: [self-hosted, linux, vpc-private]   # label selector
```

---

## Trunk-based development — the one big rule

**One main branch. No long-lived feature branches.**

Instead of:
```
feature/new-payment-flow (2 weeks of work, 300 commits, massive merge conflict)
```

You do:
```
main ← all developers commit here daily (or use short-lived branches that merge in < 1 day)
```

For incomplete features: **feature flags**. The code ships but the feature is off for users. When ready, flip the flag.

Why does this matter for CI/CD?

- Merge conflicts are tiny (everyone integrates daily)
- The main branch is always deployable (every commit is a potential release)
- No "works on my branch" — everyone sees the same integration continuously
- Deployments are boring, not events

The promotion model: `main` → staging (automatic) → production (automatic or gated). The same artifact that passed staging is what goes to production. Never rebuild.

---

## Conftest — policy on the manifests, not the code

Conftest uses Rego (OPA's policy language) to test structured files: Kubernetes YAML, Terraform plans, Helm charts, Dockerfiles.

```bash
# In CI, after generating manifests:
conftest test manifests/ --policy policies/
```

What this catches:
- `image: ubuntu:latest` in a Deployment (no pinned tag)
- `runAsRoot: true` in a container
- A Terraform plan that would expose an S3 bucket to the public
- A Kubernetes Service of type `LoadBalancer` in an account that doesn't allow it

These are policy violations you want caught before the manifest reaches ArgoCD, not after. Conftest in CI = shift-left policy enforcement.

---

## Self-test (the killer question)

Out loud:

> **"Walk me through the modern CI/CD architecture you'd recommend for a small SRE team: GHA runners, branching strategy, policy checks."**

**Reference answer (intuitive version):**

"For a small SRE team I'd start with GitHub-hosted runners for 80% of CI — simple, no maintenance, ephemeral. Self-hosted runners only for specific jobs that need VPC access or a local cache: build the image in the runner that can push directly to the private ECR, or run integration tests that hit an internal database. I'd use ARC (Actions Runner Controller) to run those on Kubernetes pods — ephemeral, auto-scaled, no persistent runner VMs.

For branching: trunk-based development. Everyone works off main, short-lived branches only (< 1 day). Feature flags for incomplete work. The main branch is always deployable. Staging deploys automatically on every merge. Production is gated — either manual approval or auto-deploy after staging soak.

For policy: Conftest in CI against Kubernetes manifests and Terraform plans. Rego policies check for the same rules Kyverno enforces in the cluster — no runAsRoot, no `latest` tag, no public S3 buckets. Catch it in the PR before it reaches the cluster."

---

## Further reading

- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller) — K8s-based self-hosted runners
- [Trunk-based development guide](https://trunkbaseddevelopment.com/) — the canonical reference
- [Conftest docs](https://www.conftest.dev/) — policy testing for configuration files

---

## Next: the deep-dive

When the assembly-line analogy feels obvious, jump to [`gha-runners-trunk-based-conftest.md`](./gha-runners-trunk-based-conftest.md). The deep-dive covers:

- GHA self-hosted runner security model
- ARC (Actions Runner Controller) setup
- Trunk-based development mechanics (feature flags, release toggles, short-lived branches)
- Immutable promotion pattern (same artifact through envs)
- Conftest setup and Rego policy writing
- The "CI as architecture" senior framing
- 3 self-test drills + 4-dimensions framing
