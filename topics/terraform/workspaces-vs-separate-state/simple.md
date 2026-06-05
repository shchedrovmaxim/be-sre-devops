# Workspaces vs separate state per environment — the simple version (the master key)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) has all the detail.

One central idea:

> **Terraform workspaces let you run the same code against multiple state files using one set of credentials. That convenience is also the danger: one mistake selects the wrong workspace, and you're applying prod changes to prod while thinking you're in dev.**

---

## The master key problem

Imagine a property manager with a master key that opens every apartment in a building. This is convenient — one key, all apartments. It's also terrifying. If that key is used carelessly, one wrong door opens and the wrong tenant's apartment gets renovated.

Workspaces are the master key:

| In the building world | In Terraform world |
|---|---|
| Master key opens any apartment | One AWS credential set works in every workspace |
| You walk into apartment 3B thinking it's 3A | You apply with `workspace select prod` thinking you're in staging |
| All apartments in the same building | All workspaces in the same S3 bucket + DynamoDB table |
| "Wrong apartment" looks identical until it's too late | Wrong workspace looks identical in the terminal until resources are destroyed |

Separate state per environment is separate buildings with separate keys. You need the right key for the right building. Getting into the wrong building requires actively having the wrong credential.

---

## What workspaces actually do

Workspaces are a thin wrapper around state file naming. That's it. When you run:

```bash
terraform workspace new staging
terraform workspace select prod
```

You're just changing which file gets read from and written to in your S3 bucket:

```
s3://my-tfstate-bucket/
  env:/staging/myproject/terraform.tfstate   # staging workspace
  env:/prod/myproject/terraform.tfstate      # prod workspace
  myproject/terraform.tfstate               # default workspace
```

Same code. Same credentials. Same bucket. Just different state files. The only thing protecting prod from your accidental `terraform apply` is the workspace you happen to have selected.

---

## The two confusing things

### 1. Workspaces don't give you IAM isolation

If your Terraform code uses an AWS role that has prod permissions, switching to the `staging` workspace doesn't strip those permissions. You can run `terraform apply` in the `prod` workspace with prod-level permissions, no matter what workspace you're in. The workspace is just a label — the IAM credentials come from your environment.

This is the core risk. In a proper multi-account setup, you'd need to actively assume a different role (different AWS credentials) to touch prod. With workspaces in one account, prod access is always one apply away.

### 2. It's very easy to forget which workspace you're in

Unlike your git branch (visible in your shell prompt if you configure it), the current Terraform workspace is not shown unless you explicitly check:

```bash
terraform workspace show
# prod
```

In practice, people forget. They run `terraform plan` in what they thought was staging, it looks fine, they run `terraform apply` — and only then do they notice the resources it's modifying are prod.

---

## When workspaces ARE the right tool

Workspaces are genuinely useful for **ephemeral environments** — short-lived environments that share a codebase and don't need IAM separation:

- Per-PR preview environments (one workspace per PR, auto-destroyed on merge)
- Temporary load-test environments (spin up with same VPC config, tear down after)
- Feature branch environments for QA testing

The key: these workspaces are throwaway. Destroying one doesn't matter. They don't hold production data. The blast radius of a mistake is low.

---

## The canonical pattern for prod/staging/dev

```
infra/
  modules/
    vpc/
    rds/
    eks/
  envs/
    dev/
      main.tf     # uses shared modules, dev-specific variables
      backend.tf  # backend bucket in dev account
    staging/
      main.tf
      backend.tf  # backend bucket in staging account
    prod/
      main.tf
      backend.tf  # backend bucket in prod account
```

Each `envs/<env>/` directory has its own backend config pointing to a state bucket in its own AWS account. To touch prod, you must have prod AWS credentials. You can't accidentally apply prod while in staging — you'd need the wrong credentials.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What does `terraform workspace select prod` actually do? | Switches which state file key is read/written in S3. Nothing else. |
| Does selecting a workspace change my AWS credentials? | No. Credentials come from your environment (env vars, profile, instance role). |
| What's the blast radius of workspace-per-env? | One wrong `workspace select` and you're applying to the wrong env. High. |
| What's the blast radius of separate-state-per-env? | You'd need the wrong AWS credentials to touch the wrong env. Much lower. |
| When are workspaces fine? | Ephemeral preview environments — throwaway, no prod data, low stakes. |
| What's the senior answer for prod/staging/dev? | Separate directories + separate backends + separate AWS accounts. |

---

## Self-test (the killer question)

Out loud:

> **"Why might using `terraform workspace` for prod/staging/dev be a bad idea?"**

**Reference answer (intuitive version):**

"The core problem is that workspaces don't give you IAM isolation. If I'm authenticated with prod credentials, switching to the `staging` workspace doesn't strip those credentials — I can still run `terraform apply` against prod-level resources. The workspace is just a prefix on the state file path.

On top of that, it's easy to forget which workspace you're in. Unlike a git branch, the current workspace isn't shown in your terminal by default. Senior engineers know that 'easy to forget' is an incident waiting to happen.

The safe pattern is separate directories with separate backend configs pointing to separate S3 buckets in separate AWS accounts. To touch prod, you need prod credentials. That's enforced at the AWS level, not by a `terraform workspace show` discipline."

---

## Further reading / watching

- **Terraform docs: workspaces** — `developer.hashicorp.com/terraform/language/state/workspaces`. Good for understanding what workspaces actually do — and the explicit note in the docs about workspaces not being a replacement for multi-environment architecture.
- **"How to manage multiple environments with Terraform" — Gruntwork blog** — search for this title. Gruntwork (makers of Terragrunt) have written the most cited pieces on environment separation patterns.

---

## Next: the deep-dive

When the master-key analogy and the IAM-isolation gap are obvious, jump to [`workspaces-vs-separate-state.md`](./deep-dive.md). The deep-dive covers:

- Workspace state naming internals (the S3 key prefix structure)
- The blast-radius argument in full
- Why same-account workspaces fail security reviews
- Drift visibility: how separate state makes cross-env drift obvious
- Detailed canonical pattern with Terragrunt DRY wrapper
- When workspaces are the right answer (ephemeral PR envs, feature branches)
- 4 self-test drills with reference answers
