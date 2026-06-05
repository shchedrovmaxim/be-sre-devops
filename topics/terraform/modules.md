# Module discipline — the deep-dive

> **Goal**: by the end you can answer — **"When would you write a custom Terraform module vs reuse one from the registry?"** — naming the rule of three, composition over inheritance, the SemVer breaking-change cost, registry trade-offs, testing options, and when the abstraction is the wrong choice.

> Start with the [simple version](./modules-simple.md) if you haven't read it. The rule of three is the spine.

---

## The senior framing — a module is an API contract, not an organizational tool

Mid-level engineers write modules to "keep things tidy." They wrap every resource group in a module because it feels cleaner.

Senior engineers know that **a module is an API**. The moment another team (or another directory) depends on your module, you have a contract. Breaking that contract has concrete costs: every consumer needs to update their code, often at the same time, often blocked by other work. Module breaking changes are the Terraform equivalent of a library API change in production software — you're coordinating work across multiple teams.

The discipline: **don't publish an API until you understand the problem it's solving, with at least three concrete consumers to inform what the interface should be.**

---

## Module source types

```hcl
# Local path (same repo, same team, easy to iterate)
module "vpc" {
  source = "./modules/vpc"
}

# Public registry (community module, pinned to version)
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.1"
}

# Git reference (internal shared repo, pinned to tag or SHA)
module "rds" {
  source = "git::https://github.com/mycompany/tf-modules//modules/rds?ref=v3.2.0"
}

# S3 (when your modules are distributed as archives)
module "security_groups" {
  source = "s3::https://s3.amazonaws.com/my-company-modules/security-groups-v1.0.0.zip"
}
```

**Local path**: no versioning. Changes are immediate. Use for modules within a single repo where you're the only consumer. The right choice when you're still discovering what the interface should be.

**Public registry**: versioned. Terraform registry protocol. `version` is a SemVer constraint (`"~> 20.0"` matches `>= 20.0, < 21.0`). Pins are essential — without them, `terraform init -upgrade` can pull breaking changes.

**Git reference**: versioned via tag or SHA. `?ref=v3.2.0` for tags, `?ref=abc1234` for SHAs. SHAs are immutable; tags can be force-pushed (rare but it happens). The `//` in the URL separates the repo root from the module subdirectory.

**The gotcha with double slashes**: `git::https://github.com/mycompany/tf-modules//modules/rds` — the `//` is the Terraform syntax to indicate "the module is in the `modules/rds` subdirectory of this repo." A single `/` means something different. A common confusing error.

---

## The rule of three — the abstraction timing principle

From the architecture doc's YAGNI section, applied to modules:

1. **First use**: write the concrete resources directly. No module.
2. **Second use**: copy the resources. Don't abstract yet — you still have only one example of how they diverge.
3. **Third use**: now you have three concrete examples. Look at all three. The things that are identical across all three are the module's core. The things that differ become input variables. Write the module from the diff, not from imagination.

The reason this works: at use #1, you're guessing what the interface should be. At use #3, the interface is obvious from the data.

### What happens when you break the rule

Write a module at use #1:

```hcl
# "I'll definitely need to reuse this RDS config everywhere"
variable "engine" {}
variable "engine_version" {}
variable "instance_class" {}
variable "allocated_storage" {}
variable "backup_retention" {}
variable "multi_az" {}
variable "deletion_protection" {}
variable "skip_final_snapshot" {}
# 20 more variables anticipating every possible caller
```

Use #2 arrives. They need a different parameter group. Add a variable. Use #3 needs cross-region replication. Now you're adding outputs and data sources. Each addition is either breaking (if you remove something) or additive (if you add optional variables). The module grows toward a generic wrapper for `aws_db_instance` — which is already a general abstraction. You've built an abstraction on top of an abstraction that doesn't capture any company-specific opinions.

The right module at use #3 might only have 6 variables — the ones that genuinely differ across your three concrete callers. Everything else is baked in (your encryption key ARN, your parameter group family, your standard backup window, your deletion protection default). That's a useful, opinionated module. The 26-variable module is not.

---

## Composition over inheritance

Terraform modules are composed, not inherited. There's no `extends` in HCL. You compose by calling modules from modules:

```hcl
# High-level "service" module composes lower-level modules
module "web_service" {
  source = "./modules/web_service"
  name   = "checkout"
  # ...
}

# Inside modules/web_service/main.tf:
module "ecs_service" {
  source     = "./modules/ecs_service"
  name       = var.name
  image      = var.image
}

module "alb_target_group" {
  source = "./modules/alb_target_group"
  # ...
}

module "cloudwatch_alarms" {
  source = "./modules/cloudwatch_alarms"
  # ...
}
```

This is fine. The anti-pattern is deeply nested module hierarchies where you can't tell which `aws_security_group` will actually be created without tracing through 4 levels of module calls. Keep hierarchies shallow (2 levels is comfortable, 3 is the limit before it becomes hard to reason about).

**The inheritance trap**: building "base modules" that other modules extend via some convention (e.g., calling a `common` module in every other module and relying on shared outputs). This creates implicit coupling between modules that should be independent. When `common` changes, every module that calls it potentially breaks. Explicit inputs > implicit coupling.

---

## Registry modules — the bugs you don't own problem

Public registry modules are written by someone else. This is simultaneously the value and the risk.

**The upside**:
- `terraform-aws-modules/vpc/aws` (the canonical community VPC module) has been used by tens of thousands of teams and its bugs have been found and fixed by many contributors.
- You get features for free — multi-AZ, flow logs, private/public subnet segregation, NACL configs — that you'd have to write yourself.
- Community PRs improve the module without your effort.

**The downside**:
- You can't merge your own bugfixes directly — you have to open a PR and wait.
- The module may enforce conventions you don't share (naming, tagging, resource structure).
- Upgrading across major versions is your problem, but the breaking changes were someone else's decision.
- "We don't fully understand what this module creates" — in a security incident, you need to know what resources exist in your account. If `module.eks.aws_iam_role.cluster` is broken, can you debug it without reading the module source?

**Senior rule of thumb**: use the registry module if:
1. The module is widely used (10,000+ downloads, active maintenance)
2. Its opinions match yours (or you're happy to inherit them)
3. You've read the source for the parts that matter most (IAM, networking — trust but verify)

Write your own if:
1. The registry module has more variables/resources than you'll ever use
2. Your company has specific opinions that differ from the module's defaults
3. The module hasn't been updated in a year and is behind on provider API support

---

## The YAGNI parallel — the "we'll add it to the module" trap

> "Let's add that option to the module so the next team doesn't have to ask us."

This is designed-in complexity (from the architecture doc). The next team hasn't asked yet. Their use case is unknown. The variable you add for the hypothetical second caller will either:
- Be wrong for the actual second caller (they needed something different)
- Add cognitive load for every future caller who sees a variable they don't understand

Every variable in a module's interface is a decision the caller has to make. Minimize them. If you're unsure whether to expose something, don't. Callers can open a PR to add variables when they need them — at that point you have a concrete requirement.

---

## Internal module monorepo patterns

For teams with multiple infrastructure repos, the module-sharing question is: where do shared modules live?

**Pattern 1 — Monorepo**: all Terraform code (modules + environment configs) in one repo.
```
infra/
  modules/vpc/
  modules/rds/
  modules/eks/
  envs/dev/
  envs/staging/
  envs/prod/
```
Simpler. Easy to make changes atomically. All modules are local references. Risk: PRs that touch a module and multiple environment configs are large and hard to review.

**Pattern 2 — Separate module repo**:
```
# repo: github.com/mycompany/terraform-modules
  modules/vpc/
  modules/rds/

# repo: github.com/mycompany/infra-prod
  main.tf (references terraform-modules at a tag)
```
Modules are versioned independently. Environment configs reference specific tags. Breaking changes are contained. Risk: coordination work across repos; you can't test a module change and an environment config change in one PR.

**Pattern 3 — Terragrunt monorepo with separate module repo** (the most common "enterprise" pattern): modules in one repo, Terragrunt configurations in another. Modules are published as tags; Terragrunt configs reference specific versions. Module authors run `terraform test` before tagging; consumers upgrade on their schedule.

---

## Testing modules

### `terraform test` (native, 1.6+)

Added in Terraform 1.6. Write tests in `.tftest.hcl` files alongside your module:

```hcl
# modules/rds/main.tftest.hcl
run "creates_rds_instance" {
  variables {
    identifier    = "test-db"
    instance_class = "db.t3.micro"
    engine_version = "15.4"
  }

  assert {
    condition     = aws_db_instance.main.engine == "postgres"
    error_message = "Engine should be postgres"
  }

  assert {
    condition     = aws_db_instance.main.deletion_protection == true
    error_message = "Deletion protection must be enabled"
  }
}
```

Run with `terraform test`. Terraform actually creates the resources (by default) and destroys them after. Use `command = plan` in a `run` block to test without creating resources (for fast iteration or when real resources are expensive).

**When to use**: any module that will be consumed by more than one team. Tests give consumers confidence that the module works as advertised and alert module authors when a change breaks behavior.

### Terratest (Go)

The pre-1.6 standard. You write Go code that calls Terraform, deploys real infrastructure, runs assertions, and destroys. More powerful (you can make HTTP requests, check actual AWS API responses, test full system behavior), but requires Go knowledge and has much longer feedback cycles (real AWS resources take minutes to create).

```go
func TestRdsModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../../modules/rds",
        Vars: map[string]interface{}{
            "identifier": "test-db-" + random.UniqueId(),
        },
    }
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    dbEndpoint := terraform.Output(t, terraformOptions, "endpoint")
    assert.NotEmpty(t, dbEndpoint)
}
```

**When to use**: complex modules where you need to verify actual behavior, not just resource attributes. Network connectivity, IAM role assumption, service reachability — things `terraform test` can't check because they require running processes.

**The `terraform test` vs Terratest trade-off**: `terraform test` is simpler and native; use it first. Fall back to Terratest only when you need to verify runtime behavior that can't be expressed as resource attribute assertions.

---

## The interview answer in 60 seconds

> "My default is to use the public registry module for standard AWS building blocks — VPCs, EKS, RDS — where the community module is mature and its opinions match ours. I read the source for the security-critical parts before trusting it.
>
> I write a custom module when the registry module doesn't match our opinions, when it has more variables than we'll ever use, or when our organization has specific standards (encryption keys, tagging, backup windows) that should be baked in, not configurable.
>
> Either way, I follow the rule of three: I don't write the abstraction until I have three concrete callers showing me what the interface should be. Before that, I copy-paste. The reason is that module breaking changes are expensive — once other teams depend on you, adding a required variable or renaming a resource is a coordinated change across multiple repos.
>
> For testing, I use `terraform test` (native since 1.6) for unit-style assertions; Terratest (Go) when I need to verify actual runtime behavior like network connectivity or IAM role assumption."

---

## Self-test drills

### 1. When would you write a custom module vs use the registry?

**Reference answer:**
- Registry for mature, widely-used modules where the module's opinions match yours. Read the security-critical source before trusting it.
- Custom when: registry module has wrong defaults, too many variables, or doesn't match company standards. Also custom when the module is business-logic-heavy (your specific EKS setup, not a generic EKS module).
- The rule of three: don't abstract until three concrete callers show you what the interface needs to be.

### 2. What makes a module change "breaking"?

**Reference answer:**
- Removing or renaming a required input variable.
- Adding a new required input variable (callers must now provide it).
- Removing an output that callers may be using.
- Renaming a resource (Terraform plans destroy + recreate — potentially destructive for real infrastructure).
- Changing a resource's `lifecycle` rules in ways that affect existing deployments.
- Additive changes (new optional variables with defaults, new outputs) are non-breaking → minor version bump.
- Bonus: changing an optional variable's default value can be breaking if callers relied on the old default — treat with caution.

### 3. How do you test a Terraform module?

**Reference answer:**
- `terraform test` (native since 1.6): HCL-based tests alongside the module, `assert` blocks check resource attributes, `command = plan` for fast iteration without real resources.
- Terratest (Go): deploy real resources, make runtime assertions (HTTP calls, AWS SDK checks), destroy after. More powerful, more setup.
- Start with `terraform test`; add Terratest only for behavior that can't be expressed as resource attribute assertions.
- Always run tests against a real AWS account (separate test account) — mocking AWS is fragile and doesn't catch IAM policy mistakes.

### 4. What's the "composition over inheritance" principle in Terraform modules?

**Reference answer:**
- Terraform has no inheritance mechanism — modules compose by calling other modules.
- Anti-pattern: deep call chains where a "web_service" module calls a "base_service" module that calls a "common" module — implicit coupling; when "common" changes, everything potentially breaks.
- Good pattern: flat composition. A "web_service" module calls independent "ecs_service", "alb", and "cloudwatch_alarms" modules. Each is independently testable and can be changed without affecting the others.
- Keep module hierarchy shallow: two levels is comfortable, three is the limit before reasoning about what resources will be created becomes difficult.

---

## Further reading / watching

- **Terraform docs: modules** — `developer.hashicorp.com/terraform/language/modules`. Official reference for source types, versioning constraints, module composition.
- **`terraform test` docs** — `developer.hashicorp.com/terraform/language/tests`. Added in 1.6; the native testing option.
- **Terratest** — `github.com/gruntwork-io/terratest`. The Go testing library; has examples for every major AWS resource type.
- **Terraform Registry** — `registry.terraform.io`. Browse `terraform-aws-modules` namespace — reading mature modules is the best way to understand good module design.
- **"Terraform: Up & Running" by Yevgeniy Brikman** (O'Reilly) — the practical book on Terraform at scale. Chapters on modules and testing are the most relevant.

---

## The 4 dimensions (senior framing)

- **Tech**: module source types and version pinning; rule of three for timing; composition over deep inheritance; `terraform test` for unit assertions; Terratest for runtime behavior verification. Registry modules: read the source for security-critical parts.
- **People**: module breaking changes create coordination work across every dependent team. Write modules with the consumers in mind — minimize required variables, document each variable, write tests that consumers can run to verify upgrades are safe. The module author is responsible for communicating breaking changes and versioning clearly.
- **CI/CD**: modules in their own repo need a CI pipeline: `terraform validate`, `terraform test`, static analysis (`tflint`, `checkov`). Tag and publish on passing CI — don't publish untested modules. Consumers should pin to specific versions; use Renovate or Dependabot to automate upgrade PRs.
- **Operations**: know which module version is deployed to prod (it's in state). When a critical bug is found in a registry module, you need to know which environments use which version — keep module versions in a discoverable format (separate `versions.tf` per environment, or tracked in a CMDB). Upgrade path for breaking changes: write a migration guide before the major version bump, not after.
