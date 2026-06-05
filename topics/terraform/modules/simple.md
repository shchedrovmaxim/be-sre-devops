# Module discipline — the simple version (the rule of three)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) has all the detail.

One central idea:

> **Don't write a Terraform module until you have three concrete, different callers that need the same thing. Before that, you're building an abstraction for a problem you don't understand yet.**

This is the same YAGNI rule from the architecture doc, applied to infrastructure code. The instinct to "wrap this in a module so we can reuse it later" usually fires too early.

---

## The wrong way to think about modules

A module is not a nice way to organize code. It's an API — a contract between the module author and the caller. Once you publish a module and two teams start using it, changing it becomes expensive. Adding a required variable is a breaking change. Removing an output that someone's using is a breaking change. Renaming a resource inside the module can cause Terraform to try to destroy and recreate real infrastructure.

The temptation:
```hcl
# "This looks like something we'll use a lot, let me wrap it"
module "s3_bucket" {
  source = "./modules/s3_bucket"
  name   = "my-app-data"
}
```

One month later, the second caller needs the bucket to be in a different region. The third caller needs versioning disabled. Now you're maintaining a module with 20 variables trying to cover cases you didn't anticipate when you wrote it.

---

## The rule of three (and why it works)

Write the concrete resource first:

```hcl
# In envs/prod/main.tf — raw, concrete, no abstraction
resource "aws_s3_bucket" "app_data" {
  bucket = "my-company-app-data-prod"
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration { status = "Enabled" }
}
```

When the second team needs the same pattern, copy it. Yes, copy it. Don't abstract yet — you still don't know which parts are genuinely shared and which will diverge.

When the **third** team needs it, you now have three concrete examples. Look at them side by side. The things that are identical across all three are the module's core. The things that differ are the module's input variables. Write the module with only those variables. Nothing else.

This is the rule of three, and it produces modules that are right-sized instead of modules that try to anticipate every possible future caller.

---

## When to use the public registry vs write your own

The public registry (`registry.terraform.io`) has modules for almost everything — VPCs, EKS clusters, RDS, S3. These modules are:

- Immediately available, no writing required
- Battle-tested by thousands of users
- Maintained by someone else (good and bad)

The tradeoff:

| Registry module | Custom module |
|---|---|
| You don't own the code | You own the code |
| Can't merge your own bugfixes | You control the release timeline |
| May have features you don't need (complexity) | Right-sized for your needs |
| Version upgrades are someone else's work | Version upgrades are your work |
| No secrets about how it works | You know exactly how it works |

**Senior default**: use the registry module for standard AWS building blocks (VPCs, IAM) where the module is well-established. Write your own for business-logic-heavy infrastructure (your specific EKS node group configuration, your company's opinionated RDS setup with encryption, parameter groups, and specific backup windows). The question is whether the registry module's opinions match yours.

---

## The SemVer contract — why module versioning is serious

When your module is consumed by other teams, a version bump is a **promise**:

```hcl
module "rds" {
  source  = "git::https://github.com/mycompany/terraform-modules//modules/rds?ref=v2.1.0"
  # ...
}
```

- `v2.1.0` → `v2.1.1`: bug fix, safe to upgrade, callers shouldn't notice
- `v2.1.0` → `v2.2.0`: new optional variable, safe to upgrade, existing callers aren't affected
- `v2.1.0` → `v3.0.0`: breaking change (removed a variable, renamed a resource, changed output type)

The `v3.0.0` bump requires every caller to update their code. If 50 repos use your module and you push a breaking change, you've just created 50 PRs that other teams need to merge before they can get any other updates. That's why module breaking changes are expensive — they create coordination work across every team that depends on you.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| When should I write a module? | When you have three concrete callers that need the same thing. Before that, write the resources directly. |
| Registry vs custom? | Registry for standard AWS building blocks. Custom for opinionated, business-specific patterns. |
| What makes a good module variable? | Only the things that genuinely differ between callers. Don't expose internals for the sake of "flexibility." |
| What's wrong with too many variables? | Every variable is cognitive load for the caller. A 40-variable module is harder to use correctly than the raw resources. |
| How do I avoid breaking changes? | Additive changes only in minor bumps (new optional variables). Renaming or removing anything requires a major version. |
| What's `terraform test`? | Native Terraform testing (added in 1.6). Lets you write test cases in HCL that call your module and assert on outputs. Replaces the need for Terratest (Go) for many use cases. |

---

## Self-test (the killer question)

Out loud:

> **"When would you write a custom Terraform module vs reuse one from the registry?"**

**Reference answer (intuitive version):**

"I use the registry for standard AWS building blocks — VPCs, security groups, IAM patterns — where the community module is well-established and its opinions match ours. The value is not writing and maintaining that code ourselves.

I write a custom module when our infrastructure has strong opinions that the registry module doesn't share — specific encryption settings, our backup windows, our tagging convention, our cost-control limits. A registry module I can't trust or can't read safely is worse than a simpler custom one.

Either way, I follow the rule of three: I don't abstract until I have three concrete users of the same pattern. Before that, I'm writing an abstraction for a problem I don't fully understand yet — and breaking changes to modules are expensive once other teams depend on them."

---

## Further reading / watching

- **Terraform docs: modules** — `developer.hashicorp.com/terraform/language/modules`. The official overview; good for understanding source types (local, registry, git).
- **Terraform Registry** — `registry.terraform.io`. Browse existing modules for VPC, EKS, RDS — see what a mature public module looks like.
- **"Terraform: Up & Running" by Yevgeniy Brikman** — chapters on module design and versioning. Best book-form treatment of when and how to write modules.
- **`terraform test` docs (1.6+)** — `developer.hashicorp.com/terraform/language/tests`. Native testing without requiring Go.

---

## Next: the deep-dive

When the rule of three and the SemVer contract feel obvious, jump to [`modules.md`](./deep-dive.md). The deep-dive covers:

- Module source types (local, registry, git refs)
- The module versioning contract in detail — what counts as breaking
- Composition vs inheritance anti-pattern
- Registry module trade-offs — the bugs-you-don't-own problem
- Internal module monorepo patterns
- Testing modules: `terraform test` (native 1.6+) vs Terratest (Go)
- The YAGNI parallel — when the registry module is the wrong abstraction
- 4 self-test drills with reference answers
