# Workspaces vs separate state per environment — the deep-dive

> **Goal**: by the end you can answer — **"Why might using `terraform workspace` for prod/staging/dev be a bad idea?"** — naming IAM isolation failure, blast-radius, drift invisibility, same-creds-across-workspaces, and the canonical alternative.

> Start with the [simple version](./workspaces-vs-separate-state-simple.md) if you haven't read it. The master-key analogy is the spine.

---

## The senior framing — workspaces are convenient and wrong for most things

Workspaces look like environment isolation. They feel like environment isolation. Terraform documentation describes them as "multiple named instances of your root module." That framing is misleading for a senior audience.

Workspaces provide **state isolation** (different state files) but **zero security isolation** (same credentials, same backend, same code path). The difference between staging and prod in a workspace-based setup is which state file gets read — and that distinction is ephemeral, easily reversed with a single command, and invisible in the terminal by default.

A senior engineer's bar for environment isolation:

- You **cannot** accidentally operate on the wrong environment without actively using the wrong credentials.
- A mistake in staging **cannot** affect prod state, even if you're careless.
- An audit trail can show who touched prod, when, why — and the trail lives at the AWS level, not just in Terraform history.

Workspaces fail all three.

---

## What workspaces actually do (internals)

When you create a workspace and select it, Terraform changes the state file key in S3:

```
# Default workspace
s3://tfstate-bucket/myproject/terraform.tfstate

# With workspace "staging"
s3://tfstate-bucket/env:/staging/myproject/terraform.tfstate

# With workspace "prod"
s3://tfstate-bucket/env:/prod/myproject/terraform.tfstate
```

The `env:/` prefix is hardcoded in Terraform. You can't change it. All workspaces live under the same bucket, the same DynamoDB table (separate lock items per workspace key), the same backend config.

Inside your Terraform code, you can read the current workspace name:

```hcl
locals {
  environment = terraform.workspace  # "prod", "staging", "dev"

  instance_count = {
    prod    = 3
    staging = 1
    dev     = 1
  }
}

resource "aws_instance" "app" {
  count = local.instance_count[local.environment]
  # ...
}
```

This looks elegant. This is where the temptation lives. And this is also where the "I selected the wrong workspace" incident originates.

### Workspace state is not tied to any IAM permission boundary

The AWS session that runs `terraform apply` is determined by your environment:

```bash
export AWS_PROFILE=prod-admin  # or AWS_ACCESS_KEY_ID + SECRET, or EC2 instance role
terraform workspace select staging
terraform apply  # applies against STAGING state, with PROD IAM permissions
```

The workspace selection is completely orthogonal to your credentials. Terraform doesn't check whether your credentials match the workspace you selected. There's no guard.

---

## The blast-radius argument

With workspaces:

```
prod workspace         staging workspace
  state file A           state file B
      |                      |
      +--------+-------------+
               |
         Same AWS account
         Same IAM role
         Same S3 bucket
         Same code
```

One wrong `terraform workspace select prod && terraform apply` — and you're running prod changes while thinking you're in staging. The only thing that saves you is either:
- Catching it from the plan output ("wait, why are there prod-looking resource names in this plan?")
- Having a pre-commit hook that refuses to apply without a workspace confirmation prompt

Both of these are human controls. They fail.

With separate state per environment + separate AWS accounts:

```
prod (AWS account A)       staging (AWS account B)
  own S3 bucket              own S3 bucket
  own DynamoDB table         own DynamoDB table
  own IAM role               own IAM role
```

To touch prod, you must actively assume prod credentials. If your shell is pointed at the staging profile, `terraform apply` in the `envs/prod/` directory will fail because the staging role doesn't have access to the prod state bucket. The boundary is enforced at the AWS IAM level, not by human discipline.

---

## Why drift between environments is hard to see with workspaces

When prod has a change that staging doesn't, the difference lives buried in state:

```bash
terraform workspace select prod
terraform show | grep instance_type  # r5.2xlarge, 6 nodes

terraform workspace select staging
terraform show | grep instance_type  # t3.large, 1 node
```

You have to actively switch workspaces and compare. There's no structural diff. In a separate-directory approach, comparing `envs/prod/main.tf` with `envs/staging/main.tf` via `git diff` tells you immediately where they've diverged in code — and a scheduled `terraform plan` in CI for each directory shows you where they've diverged in reality (drift).

Drift between prod and staging is often intentional (different instance sizes, different replica counts) but sometimes a symptom of a problem (prod has a manual change that never made it back to code, or a one-off fix was applied to staging but not prod). Separate directories make this visible. Workspaces bury it.

---

## IAM separation failure — why this matters for security reviews

Most security audits will flag workspaces-for-environments as a control gap:

- **Same credentials can operate on prod** even when a developer thinks they're in staging.
- **No access control between environments** at the Terraform level — the gate is entirely in the developer's head.
- **Audit trail ambiguity** — if something changes in prod, did the operator intend to touch prod? You have to check CloudTrail for which API calls were made and which resources they touched, rather than having an apply-level record tied to a specific "touch prod" credential assumption.

Many compliance frameworks (SOC 2, PCI DSS) require demonstrable segregation of duties between production and non-production environments. "We use terraform workspace select" is not a control. "Production applies require assuming a dedicated prod IAM role that is gated by MFA and audit-logged" is a control.

---

## The canonical pattern

```
infra/
  modules/
    vpc/       # shared module, no environment logic
    rds/
    eks/
  envs/
    dev/
      main.tf        # module calls with dev-specific inputs
      variables.tf
      outputs.tf
      backend.tf     # S3 bucket in dev AWS account
      terraform.tfvars
    staging/
      main.tf
      variables.tf
      outputs.tf
      backend.tf     # S3 bucket in staging AWS account
      terraform.tfvars
    prod/
      main.tf
      variables.tf
      outputs.tf
      backend.tf     # S3 bucket in prod AWS account
      terraform.tfvars
```

Each `envs/<env>/` directory is a fully independent Terraform root. No shared backend. No shared state. The modules in `modules/` are the shared logic.

To apply staging:
```bash
export AWS_PROFILE=staging-operator
cd infra/envs/staging
terraform init
terraform apply
```

To apply prod:
```bash
export AWS_PROFILE=prod-operator  # different creds — meaningful boundary
cd infra/envs/prod
terraform init
terraform apply  # if AWS_PROFILE is wrong, this errors on S3 access
```

### The DRY problem

The canonical pattern has a real cost: lots of duplicated HCL. `envs/dev/main.tf` and `envs/staging/main.tf` might be 90% identical, differing only in variable values. This is where **Terragrunt** comes in — it adds a DRY wrapper so you write the module invocation once and inject environment-specific values at each level.

```hcl
# envs/prod/terragrunt.hcl
inputs = {
  environment    = "prod"
  instance_count = 3
  instance_type  = "r5.2xlarge"
}

# Root terragrunt.hcl (applies to all envs)
terraform {
  source = "../../modules/eks"
}

remote_state {
  backend = "s3"
  config = {
    bucket = "tfstate-${local.account_id}"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

Terragrunt runs `terraform init` + `terraform apply` under the hood, injecting the right backend config and inputs for each environment. More on this in `pr-workflows.md`.

---

## When workspaces are the right answer

There are two scenarios where workspaces are genuinely correct:

### 1. Ephemeral PR preview environments

```bash
# In CI, on PR open
terraform workspace new "pr-${PR_NUMBER}"
terraform apply -var="environment=preview"

# On PR merge
terraform destroy
terraform workspace delete "pr-${PR_NUMBER}"
```

Each PR gets a fresh, isolated environment. Low blast radius — these are throwaway. Same credentials are fine because preview environments don't hold production data. You're not promoting a preview to production; you're just spinning up a throwaway to test changes.

### 2. Feature branch / QA environments

Same logic: short-lived, throwaway, identical to preview in risk profile.

The key distinguisher: **ephemeral environments where the consequence of a mistake is "I destroyed a temporary environment."** Not: "I destroyed something a customer was using."

---

## The interview answer in 60 seconds

> "Workspaces don't give you IAM isolation — they only switch which state file you're reading. If I have prod credentials and I `workspace select staging`, I can still apply against prod-level AWS resources. The only thing separating staging and prod is which workspace I happen to have selected, which is one command away from changing.
>
> The blast radius is high: one `workspace select prod` in the wrong terminal, and I'm running a prod apply while thinking I'm in staging. That's an incident that's happened to everyone who's used workspaces for environments.
>
> The canonical pattern is separate directories per environment, each with its own backend config pointing to its own S3 bucket in its own AWS account. To touch prod, you must actively assume prod credentials — that's enforced at the IAM level, not by developer discipline.
>
> The one place workspaces are correct is ephemeral PR preview environments — throwaway, same credentials are fine, blast radius is 'I wasted 10 minutes recreating a temp environment.'"

---

## Self-test drills

### 1. Why might using workspaces for prod/staging/dev be a bad idea?

**Reference answer:**
- Workspaces only switch state files — same credentials, same bucket, same code.
- No IAM isolation: a dev with prod creds can apply to prod regardless of which workspace they're in.
- Easy to forget current workspace (not shown in prompt by default); incident-prone.
- Audit trail ambiguity: no clear "you assumed prod role" boundary.
- Drift between envs invisible without active comparison.
- Fails security audits requiring demonstrable env segregation.

### 2. What's the canonical alternative to workspaces for env separation?

**Reference answer:**
- Separate directory per environment (`envs/dev`, `envs/staging`, `envs/prod`).
- Each has its own `backend.tf` pointing to its own S3 bucket in its own AWS account.
- To touch prod, you must actively assume prod-account credentials — IAM-enforced, not convention-enforced.
- Modules in `modules/` are shared; environment-specific values are in `terraform.tfvars` or Terragrunt inputs.
- Bonus: Terragrunt removes the DRY problem by letting you write the module invocation once.

### 3. When would you actually use workspaces?

**Reference answer:**
- Ephemeral, throwaway environments: per-PR preview environments, feature branch QA envs.
- Low blast radius — these don't hold production data and are designed to be created and destroyed.
- Same AWS account is fine because there's no prod-vs-non-prod separation needed.
- Key: the workspace is short-lived. You'd never run a workspace for longer than the PR or sprint it's tied to.

### 4. How does Terragrunt help with the "separate state per env" pattern?

**Reference answer:**
- Separate-directory pattern has DRY cost: `envs/dev/main.tf` and `envs/prod/main.tf` are nearly identical.
- Terragrunt is a thin wrapper that generates `backend.tf` and variable inputs from a hierarchy of `terragrunt.hcl` files.
- Write the module invocation once in a root `terragrunt.hcl`; each env-level `terragrunt.hcl` overrides only what differs (instance sizes, replica counts, account IDs).
- Terragrunt calls `terraform init` and `terraform apply` underneath — it's not a replacement for Terraform, it's a DRY layer on top.
- Bonus: Terragrunt also supports `run-all` to apply changes across multiple environments in dependency order — useful for bootstrapping a full env from scratch.

---

## Further reading / watching

- **Terraform docs: workspaces** — `developer.hashicorp.com/terraform/language/state/workspaces`. Note the explicit docs caveat that workspaces are not a replacement for environment separation.
- **Gruntwork blog: "How to manage multiple environments with Terraform"** — search title on `gruntwork.io` or web. The canonical article on env separation patterns; Gruntwork's opinion is informed by having productionized Terragrunt.
- **Terragrunt docs** — `terragrunt.gruntwork.io`. The "Getting started" and "DRY backends" sections cover the patterns referenced here.
- **"Terraform: Up & Running" by Yevgeniy Brikman** (O'Reilly) — chapter on state and workspace management. Brikman is the Terragrunt author; his treatment of workspaces vs separate-state is the most practical in book form.

---

## The 4 dimensions (senior framing)

- **Tech**: workspace = state-prefix change, not credential change. Separate-state = different S3 buckets, different DynamoDB tables, different AWS account credentials. `terraform_remote_state` for cross-account output sharing. Terragrunt as the DRY wrapper.
- **People**: "workspace select prod" incidents are a rite of passage — someone on your team will do it. Separate-state makes the wrong-environment mistake impossible without actively having wrong creds. Lower cognitive load: `cd envs/prod && terraform plan` is unambiguous; `terraform workspace select prod && terraform plan` is one typo from disaster.
- **CI/CD**: CI/CD pipelines for separate-state envs use separate IAM roles per environment. The pipeline for staging assumes `staging-terraform-role`; prod pipeline assumes `prod-terraform-role` — gated by branch protection or manual approval. Workspaces can't offer this without additional wrappers.
- **Operations**: Drift detection (scheduled `terraform plan` in CI) is straightforward with separate directories — one job per env, each with the right credentials. With workspaces, you'd have to script credential switching + workspace switching, which is error-prone. Audit trail for prod applies lives in CloudTrail as `AssumeRole` calls — clear evidence that someone specifically used prod credentials.
