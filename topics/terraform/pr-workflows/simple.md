# PR-based Terraform workflows — the simple version (the four-eyes checkout)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) has all the detail.

One central idea:

> **A safe Terraform PR workflow plans on every commit (so reviewers see the impact), only applies on merge (so humans approve before anything changes), and has a policy gate that rejects dangerous patterns before a human ever sees them.**

---

## The four-eyes checkout

Imagine a bank's vault room. To access it, two people must both insert their keys simultaneously — neither one can open it alone. This is "four-eyes control": a change requires a second person to consciously approve it.

A mature Terraform PR workflow is the infrastructure equivalent:

| In the vault world | In Terraform world |
|---|---|
| First key: the requester opens a request | Opening a PR with the infrastructure change |
| Plan output posted as a comment | `terraform plan` output shows exactly what changes |
| Vault manager reviews the request | A human reviews the plan output in the PR |
| Both keys turned at once | "Approve + merge" triggers the apply |
| Vault manager signs the log | Git history is the permanent audit trail |
| One key can't open it alone | `terraform apply` can't run without an approved merged PR |

The workflow makes "what will change" visible *before* anything changes, and requires a deliberate human decision to proceed.

---

## The three tools and what each one is

These tools solve the same problem in different ways:

**Atlantis** — self-hosted, runs in your cluster, talks directly to GitHub/GitLab. When you open a PR, Atlantis runs `plan` and posts the output as a PR comment. When you merge, it runs `apply`. Simple to understand, hard to scale beyond 10–15 workspaces.

**Spacelift** — managed SaaS (you pay per use). More powerful policy engine (OPA natively), better for multi-account fan-out, drift detection built in. Steeper learning curve, more expensive, but less operational overhead.

**Terragrunt** — this is different. It's not a CI/CD tool — it's a DRY wrapper around Terraform. It handles the "I don't want to copy-paste 90% identical code for each environment" problem. Usually combined with Atlantis or Spacelift, not a replacement for them.

---

## The two confusing things

### 1. Terragrunt is not an Atlantis alternative

When people say "we use Terragrunt," they mean their Terraform code is structured using Terragrunt's patterns (hierarchical `terragrunt.hcl` files for DRY). They still need something (Atlantis, Spacelift, GitHub Actions) to run the plans and applies in CI. Terragrunt ≠ CI/CD.

### 2. Apply-on-merge doesn't mean "automatically"

"Apply on merge" means the merge event triggers the apply — but usually there's still a human-approval step in the PR. The flow is:

1. PR opened → plan runs → comment posted
2. Reviewer reads the plan → approves the PR
3. Merge → apply starts

The human approval is step 2. The apply is automatic *after* that deliberate human approval. It's not "any merge applies" — it's "an approved PR merge applies." That distinction is the safety property.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What does Atlantis do? | Self-hosted. Runs `plan` on PRs, posts output as comment, runs `apply` on merge. |
| What does Spacelift do? | Managed SaaS. Same PR workflow but adds native OPA policy, drift detection, better multi-account support. |
| What does Terragrunt do? | DRY wrapper for Terraform code structure. Not a CI/CD tool. |
| What's OPA/Conftest in this context? | Open Policy Agent — lets you write rules like "no public S3 buckets" or "all resources must have owner tag" that fail the plan before a human reviews it. |
| How do you stop a bad prod apply? | Branch protection (require approval), apply locking (one apply at a time), Atlantis/Spacelift permission policies (only certain roles can approve prod applies). |
| Who should be able to approve a prod apply? | Typically: on-call SRE + one additional approver. Not just any team member. |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through a Terraform PR workflow that's safe for prod."**

**Reference answer (intuitive version):**

"The flow: open a PR → CI runs `terraform plan` and posts the output as a PR comment (Atlantis or GitHub Actions) → a reviewer reads the plan to confirm the changes match intent → reviewer approves → merge → `terraform apply` runs automatically.

The safety properties: you can't apply without a reviewed plan output (the plan is posted in the PR, it's part of the review). You can't apply without PR approval (branch protection). You can't have two simultaneous applies (apply locking). And policy gates (OPA/Conftest checks in CI) reject plans that violate security rules — no public S3 buckets, encryption required — before a human even reviews them.

For prod specifically: require a second approver beyond the author, gate apply on the SRE team's approval rather than any team member, and keep the audit trail in Git (every apply is a merge commit)."

---

## Further reading / watching

- **Atlantis docs** — `runatlantis.io`. The quickstart shows the plan/apply PR comment workflow concretely.
- **Spacelift docs** — `docs.spacelift.io`. Their "Getting started" section covers the PR workflow; the policy docs cover OPA integration.
- **Terragrunt docs** — `terragrunt.gruntwork.io`. The "Quick start" and "Keep your Terraform code DRY" sections.
- **OPA/Conftest** — `conftest.dev`. The tool for writing policy-as-code checks against Terraform plan output.

---

## Next: the deep-dive

When the four-eyes analogy and the plan-comment-apply flow are obvious, jump to [`pr-workflows.md`](./deep-dive.md). The deep-dive covers:

- The full Atlantis setup (atlantis.yaml, webhook, workspace locking)
- Spacelift vs Atlantis trade-offs (managed vs self-hosted, policy model, scale)
- Terragrunt's role in the architecture (DRY wrapper, `run-all`, dependency graph)
- OPA/Conftest policy examples (no-public-S3, encryption-required, tags-required)
- The "who can approve a prod apply" governance model
- Multi-account fan-out patterns
- 4 self-test drills with reference answers
