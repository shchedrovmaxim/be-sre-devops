# PR-based Terraform workflows (Atlantis / Spacelift / Terragrunt) — the deep-dive

> **Goal**: by the end you can answer — **"Walk me through a Terraform PR workflow that's safe for prod."** — naming plan-in-PR, apply-on-merge-with-approval, apply locking, OPA/Conftest policy gates, and the governance model for who can approve prod applies.

> Start with the [simple version](./pr-workflows-simple.md) if you haven't read it. The four-eyes-checkout analogy is the spine.

---

## The senior framing — "terraform apply" from a laptop is a smell

Every team starts here: someone SSHes into a bastion or opens their laptop, sets `AWS_PROFILE=prod`, and runs `terraform apply`. It works. It's fast. It's also:

- **Unreviewed**: what did the plan show? Nobody else knows.
- **Untested**: did any policy checks run? Probably not.
- **Unaudited**: the Git history may not reflect what was applied. The AWS CloudTrail does, but not with the human context.
- **Unconstrained**: any engineer with prod credentials can apply anything at any time.

A mature PR-based workflow closes all four gaps. The workflow is the control: no apply without a reviewed plan, no apply without an approved PR, no two applies at the same time, no apply that violates policy.

---

## The canonical PR workflow

```
1. Engineer opens a PR (Terraform code change)
   |
   v
2. Atlantis/Spacelift/GitHub Actions runs `terraform plan`
   Posts plan output as PR comment
   |
   v
3. Reviewer reads the plan — does the diff match intent?
   "This plan shows 2 resources changing — that's expected."
   |
   v
4. Reviewer approves PR
   |
   v
5. PR is merged (merge commit is the audit trail entry)
   |
   v
6. Atlantis/Spacelift detects the merge, acquires apply lock,
   runs `terraform apply`, posts result as a comment
   |
   v
7. Apply completes. Lock released. Result visible in PR comment history.
```

The key invariant: **you cannot reach step 6 without steps 3 and 4.** Branch protection enforces this. The apply tool only applies on merge, not on arbitrary triggers.

---

## Atlantis — self-hosted, GitHub-native

Atlantis is a Go service you run in your cluster. It integrates with GitHub/GitLab/Bitbucket via webhooks. When a PR is opened or a commit is pushed, GitHub sends a webhook to Atlantis; Atlantis runs `terraform plan` and posts the output.

### `atlantis.yaml`

```yaml
version: 3
automerge: false          # don't auto-merge after successful apply
parallel_plan: true       # run multiple plans in parallel (different projects)
parallel_apply: false     # serialize applies (one at a time, safer)

projects:
  - name: prod-networking
    dir: envs/prod/networking
    workspace: default
    autoplan:
      enabled: true
      when_modified:
        - "**/*.tf"
        - "../../modules/**/*.tf"   # also plan if shared modules change

  - name: prod-eks
    dir: envs/prod/eks
    workspace: default
    autoplan:
      enabled: true
      when_modified:
        - "**/*.tf"
```

### PR commands

Once Atlantis is running, team members interact via PR comments:

```
atlantis plan         → run plan for all projects affected by this PR
atlantis plan -p prod-networking  → plan only the networking project
atlantis apply        → apply all planned projects (requires PR approval first)
atlantis apply -p prod-networking → apply only networking
atlantis unlock       → release apply lock (for stuck applies)
```

### Apply locking

Atlantis maintains a workspace-level lock: only one apply runs at a time per workspace. If two PRs are open and both are approved, the second apply waits for the first to complete. This prevents concurrent state modification (on top of the DynamoDB lock).

**The cross-PR locking gotcha**: if PR #1 applies successfully but PR #2's plan was computed before PR #1's apply, PR #2's plan is now stale — it was planned against old state. Atlantis will warn about this and require a re-plan before allowing apply. This is correct behavior; failing to catch it leads to plan/apply inconsistency.

### Atlantis at scale

Atlantis works well up to ~15-20 active workspaces. Beyond that:
- Many concurrent PRs means many concurrent plans, and Atlantis queues them on a single service
- Large plan outputs (500KB+) in PR comments become noise
- No native multi-tenancy (all teams share one Atlantis instance's blast radius)
- Atlantis itself becomes a critical piece of infrastructure — its own availability affects all Terraform work

For larger organizations, Spacelift or Terraform Cloud is the operational upgrade.

---

## Spacelift — managed, policy-rich

Spacelift is a SaaS (or self-hosted enterprise). It connects to your VCS, creates "stacks" (one stack per Terraform root module), and manages the plan/apply lifecycle.

**Key differences from Atlantis**:

| | Atlantis | Spacelift |
|---|---|---|
| Hosting | Self-hosted (Helm chart) | SaaS (or enterprise self-hosted) |
| Policy engine | External (Conftest in CI) | Native OPA integration |
| Drift detection | Manual/external scripting | Built-in, scheduled |
| Multi-account fan-out | Per-project credential config | Stack groups + environment inheritance |
| Approvals | GitHub PR review | GitHub PR review + Spacelift-level policies |
| Cost | Free + infrastructure cost | Per-stack/resource pricing |
| Scale | Good to ~20 workspaces | Designed for hundreds of stacks |

### Spacelift stack config (via Terraform provider)

```hcl
resource "spacelift_stack" "prod_networking" {
  name        = "prod-networking"
  repository  = "mycompany/infra"
  branch      = "main"
  project_root = "envs/prod/networking"
  terraform_version = "1.7.0"

  autodeploy  = false   # don't auto-apply; require manual approval after plan
  autoretry   = false

  labels = ["prod", "networking"]
}

resource "spacelift_drift_detection" "prod_networking" {
  stack_id   = spacelift_stack.prod_networking.id
  schedule   = ["0 8 * * *"]   # daily at 8 AM UTC
  reconcile  = false            # detect only, don't auto-apply
}
```

---

## Terragrunt — the DRY wrapper (not a CI/CD tool)

Terragrunt is a thin wrapper around Terraform that solves two problems:

1. **DRY backend configuration** — instead of copying the S3 backend config into every `envs/<env>/backend.tf`, define it once in a root `terragrunt.hcl`.
2. **DRY variable injection** — environment-specific values flow down from parent `terragrunt.hcl` files to children.

```
infra/
  terragrunt.hcl         # root: defines backend config template, provider version
  envs/
    prod/
      terragrunt.hcl     # sets account_id, region, environment = "prod"
      networking/
        terragrunt.hcl   # calls module, sets networking-specific vars
      eks/
        terragrunt.hcl   # calls module, declares dependency on networking
    staging/
      terragrunt.hcl
      networking/
        terragrunt.hcl
```

```hcl
# infra/terragrunt.hcl (root)
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket         = "mycompany-tfstate-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

locals {
  account_id = get_aws_account_id()
}
```

```hcl
# infra/envs/prod/eks/terragrunt.hcl
terraform {
  source = "../../../../modules/eks"
}

include "root" {
  path = find_in_parent_folders()
}

dependency "networking" {
  config_path = "../networking"   # read outputs from networking state
}

inputs = {
  vpc_id     = dependency.networking.outputs.vpc_id
  subnet_ids = dependency.networking.outputs.private_subnet_ids
  node_count = 3
}
```

### `run-all`

```bash
# Apply all stacks in envs/prod in dependency order
terragrunt run-all apply --terragrunt-working-dir envs/prod

# Plan all stacks in envs/staging
terragrunt run-all plan --terragrunt-working-dir envs/staging
```

`run-all` resolves the dependency graph (EKS depends on networking → run networking first) and applies in the right order. This is invaluable for bootstrapping a new environment from scratch.

**Terragrunt + Atlantis**: Atlantis can be configured to run Terragrunt commands instead of Terraform directly. Set `terraform_bin` in Atlantis config to `terragrunt`. Plans and applies then respect Terragrunt's dependency graph.

---

## OPA/Conftest — policy as code in CI

Before a human reviews the plan, an automated policy check should reject obviously bad patterns. Conftest is the tool; OPA (Open Policy Agent) is the policy language.

```bash
# In CI, before posting the plan to PR:
terraform plan -json -out=tfplan.binary > tfplan.json
conftest test tfplan.json --policy ./policies/
```

### Example policies

```rego
# policies/no_public_s3.rego
package terraform

import input.resource_changes

deny[msg] {
  resource := resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.after.block_public_acls == false
  msg := sprintf("S3 bucket %s must have block_public_acls = true", [resource.address])
}
```

```rego
# policies/encryption_required.rego
package terraform

deny[msg] {
  resource := resource_changes[_]
  resource.type == "aws_db_instance"
  not resource.change.after.storage_encrypted
  msg := sprintf("RDS instance %s must have storage_encrypted = true", [resource.address])
}
```

```rego
# policies/required_tags.rego
package terraform

required_tags := {"Environment", "Owner", "CostCenter"}

deny[msg] {
  resource := resource_changes[_]
  resource.change.actions != ["no-op"]
  missing := required_tags - {k | resource.change.after.tags[k]}
  count(missing) > 0
  msg := sprintf("Resource %s is missing required tags: %v", [resource.address, missing])
}
```

These policies run before the PR comment is posted. If they fail, the CI job fails and the PR is blocked. The reviewer never has to make judgment calls about encryption or tagging — the policy enforces it automatically.

---

## The governance question — who can approve a prod apply

This is the question the interviewer is really asking when they say "walk me through a safe workflow." The technical plumbing is easy; the governance model is the nuanced part.

### Minimum viable governance for prod

1. **Require PR review from the SRE team** (not just any team member). In GitHub, this is a CODEOWNERS rule:
   ```
   # envs/prod/ changes require SRE team review
   envs/prod/    @mycompany/sre-team
   ```

2. **Require at least 2 approvals** (branch protection setting) for prod changes.

3. **Restrict who can merge to main** — in environments with strict compliance, merge is gated on a senior SRE's approval, not just any reviewer.

4. **Apply lock per environment** — Atlantis/Spacelift allow only one apply per stack at a time. No two prod applies run simultaneously.

5. **Mandatory plan review before apply** — in Atlantis, `require_approval = true` blocks apply until a named GitHub team has approved. In Spacelift, this is a stack policy.

### Spacelift policy for prod approval

```rego
# policies/prod_approval.rego
package spacelift

# Block apply if prod stack doesn't have SRE approval
deny["Prod applies require SRE team approval"] {
  input.run.type == "TRACKED"  # tracked runs are the real applies
  input.stack.labels[_] == "prod"
  not sre_approved
}

sre_approved {
  input.reviews[_].author.teams[_] == "sre-team"
}
```

### The "who approves" escalation question

Senior interview signal: be ready to answer "what if the SRE team is unavailable but there's an urgent prod fix needed?"

The answer is a documented break-glass procedure:
- A defined set of senior engineers (usually tech leads or on-call engineers) can approve in an emergency.
- The break-glass apply is still done via PR, not via local `terraform apply` (preserves audit trail).
- A post-mortem ticket is filed explaining why the normal approval chain was bypassed.

"We'd do a local terraform apply" is the wrong answer. "We have a documented break-glass procedure with emergency approvers" is the right one.

---

## Multi-account fan-out patterns

When managing infrastructure across multiple AWS accounts (prod account, staging account, logging account, security account), the plan/apply workflow needs per-account credentials.

### Pattern 1 — Role assumption per stack

Each Atlantis/Spacelift stack is configured with a role to assume:

```yaml
# atlantis.yaml
projects:
  - name: prod-eks
    dir: envs/prod/eks
    execution_order_group: 1
    workflow: prod

workflows:
  prod:
    plan:
      steps:
        - env:
            name: AWS_ROLE_ARN
            value: arn:aws:iam::PROD_ACCOUNT:role/TerraformRole
        - run: aws sts assume-role --role-arn $AWS_ROLE_ARN ...
        - init
        - plan
```

### Pattern 2 — Spacelift environment variables per stack

```hcl
resource "spacelift_environment_variable" "prod_role" {
  stack_id    = spacelift_stack.prod_eks.id
  name        = "AWS_ROLE_ARN"
  value       = "arn:aws:iam::${var.prod_account_id}:role/TerraformRole"
  write_only  = true
}
```

Each stack has its own role ARN. The Spacelift runner assumes that role before running Terraform. Engineers don't have direct prod credentials on their laptops — they only interact through PRs.

---

## The interview answer in 60 seconds

> "The flow: open a PR → Atlantis runs `terraform plan` and posts the output as a PR comment, plus Conftest checks policy gates (no public buckets, encryption required, all tags present) → a reviewer reads the plan output to confirm the changes match intent → at least two approvals, SRE team required for prod → merge → Atlantis detects the merge, acquires the workspace apply lock, runs `terraform apply`, posts the result as a comment.
>
> Safety properties: can't apply without a reviewed plan (it's in the PR), can't apply without approval (branch protection), can't have concurrent applies (Atlantis workspace lock), can't violate policy (Conftest fails CI before human review). Every apply is tied to a Git commit with author, reviewer, and timestamp.
>
> For who approves prod: CODEOWNERS requires SRE team review, two approvals minimum. Emergency break-glass uses named senior engineers but still through a PR — never a local apply."

---

## Self-test drills

### 1. Walk me through a Terraform PR workflow safe for prod.

**Reference answer:**
- PR opened → plan runs in CI, output posted as PR comment.
- OPA/Conftest policy gates check the plan before the comment is posted — fail if public S3, no encryption, missing tags.
- Reviewer reads plan output in the PR — does the diff match intent?
- At least 2 approvals; SRE team required for prod (CODEOWNERS).
- Merge → apply-on-merge; apply locking prevents concurrency.
- Result posted as PR comment. Git history is the audit trail.
- Bonus: break-glass procedure for emergencies — documented, emergency approvers named, post-mortem filed.

### 2. Atlantis vs Spacelift — when would you choose each?

**Reference answer:**
- Atlantis: small/medium teams, up to ~20 workspaces, want to self-host and keep infrastructure simple, don't need advanced policy engine or built-in drift detection. Cost is infrastructure, not per-stack.
- Spacelift: larger organizations, 50+ stacks, need native OPA policies, drift detection, or fine-grained approval policies without custom scripting. The operational overhead of running Atlantis at scale (HA, storage, webhook reliability) exceeds Spacelift's SaaS cost.
- Neither: Terraform Cloud (HCP Terraform) is another option — HashiCorp's managed offering. Similar to Spacelift in the managed-SaaS tier but deeper integration with provider registry and Sentinel policies.

### 3. What's Terragrunt's role in a PR-based workflow?

**Reference answer:**
- Terragrunt is a DRY wrapper for Terraform — not a CI/CD tool. It handles environment-specific backend config and variable injection so you don't copy-paste 90% identical HCL across dev/staging/prod.
- In a PR workflow, Atlantis or Spacelift still runs the CI side. You configure Atlantis to call `terragrunt plan` instead of `terraform plan` so the DRY structure is respected.
- `run-all` is useful for bootstrapping or destroying a full environment in dependency order.
- Key: Terragrunt solves code organization; Atlantis/Spacelift solve CI/CD governance. They complement each other.

### 4. How would you enforce "all prod resources must have an Owner tag" via policy?

**Reference answer:**
- Write an OPA policy (Rego) that checks the plan JSON's `resource_changes` for any resource with `actions != ["no-op"]` and verifies that `tags.Owner` is present in the `after` attributes.
- Run it via Conftest in CI: `conftest test tfplan.json --policy policies/`.
- If the policy fails, CI fails → PR is blocked → the plan can't be applied until the Terraform code is updated to include the Owner tag.
- This runs before a human reviews the PR — human review is for intent verification, not for catching missing tags. Policy gates handle the mechanical checks.
- Bonus: combine with `ignore_changes = [tags["Owner"]]` risk consideration — if someone puts that in their code to bypass the policy, a linter like `tflint` or a secondary policy check can flag the pattern.

---

## Further reading / watching

- **Atlantis docs** — `runatlantis.io`. The quickstart and `atlantis.yaml` reference cover everything needed for a basic setup.
- **Spacelift docs: policies** — `docs.spacelift.io/concepts/policy`. The OPA policy examples for approval and push policies.
- **Terragrunt docs** — `terragrunt.gruntwork.io`. "Keep your Terraform code DRY" and "Module dependency management" sections.
- **Conftest** — `conftest.dev`. The testing tool for OPA policies against structured data (Terraform plans, Kubernetes manifests, Dockerfiles).
- **Open Policy Agent** — `openpolicyagent.org`. The policy language. The Rego playground at `play.openpolicyagent.org` is the fastest way to test policies.
- **"Terraform: Up & Running" by Yevgeniy Brikman** (O'Reilly) — chapter on automated testing and CI/CD for Terraform. Best book-form treatment of the PR workflow topic.

---

## The 4 dimensions (senior framing)

- **Tech**: plan-in-PR (Atlantis/Spacelift), apply-on-merge with lock, OPA/Conftest policy gates in CI, CODEOWNERS for approval routing, Terragrunt for DRY code structure, per-stack IAM role assumption for multi-account security.
- **People**: the workflow removes "who applied that?" ambiguity — every apply is a merge commit with author and approvers. The governance model (who can approve prod) must be explicitly documented and agreed to — not implicit tribal knowledge. New engineers can run applies safely without prod credentials on their laptops.
- **CI/CD**: apply-on-merge is the pattern; local `terraform apply` is the anti-pattern (no review, no audit trail, no lock). Policy gates (Conftest) are part of the CI pipeline, not a separate manual review. The apply pipeline is separate from the plan pipeline — plans are cheap, applies are consequential.
- **Operations**: workspace locking prevents concurrency at the Terraform level (on top of DynamoDB locking). Apply failures are visible in the PR comment thread — no hunting through CI logs. Runbook for "apply failed after merge": check the PR comment for the error, check CloudTrail for what did and didn't change, determine whether to re-apply or roll back at the application level.
