# Terraform

IaC for cloud infrastructure. Senior signal: you treat state and module structure as architecture.

## Recommended order

1. **Remote state** — S3 backend + DynamoDB locking
2. **Workspaces vs separate state per env** — why workspaces are usually wrong for prod
3. **Module discipline** — when to write vs use registry
4. **`terraform_data`, `lifecycle` blocks, `moved` blocks**
5. **Drift detection** — scheduled `plan` + alerting
6. **PR-based workflows** — Atlantis / Spacelift / Terragrunt
7. **Testing** — terratest, native `terraform test` (1.6+)
8. **Multi-account AWS patterns** — provider aliases, assume-role chains
9. **Sensitive output handling** — `sensitive = true`, marking state encryption

## Files

| File | Type | Topic | Date |
|---|---|---|---|
| [remote-state-simple.md](./remote-state-simple.md) | Simple companion | Remote state with S3 + DynamoDB locking | 2026-06-05 |
| [remote-state.md](./remote-state.md) | Deep-dive | Remote state with S3 + DynamoDB locking | 2026-06-05 |
| [workspaces-vs-separate-state-simple.md](./workspaces-vs-separate-state-simple.md) | Simple companion | Workspaces vs separate state per environment | 2026-06-05 |
| [workspaces-vs-separate-state.md](./workspaces-vs-separate-state.md) | Deep-dive | Workspaces vs separate state per environment | 2026-06-05 |
| [modules-simple.md](./modules-simple.md) | Simple companion | Module discipline — when to write vs use registry | 2026-06-05 |
| [modules.md](./modules.md) | Deep-dive | Module discipline — when to write vs use registry | 2026-06-05 |
| [drift-detection-simple.md](./drift-detection-simple.md) | Simple companion | Drift detection | 2026-06-05 |
| [drift-detection.md](./drift-detection.md) | Deep-dive | Drift detection | 2026-06-05 |
| [pr-workflows-simple.md](./pr-workflows-simple.md) | Simple companion | Atlantis / Spacelift / Terragrunt — PR-based workflows | 2026-06-05 |
| [pr-workflows.md](./pr-workflows.md) | Deep-dive | Atlantis / Spacelift / Terragrunt — PR-based workflows | 2026-06-05 |

## Why this matters

Terraform is the universal language for cloud infra. Senior interviews probe **state design** and **module organization** more than syntax.

## Common interview questions

- "How do you manage state for a multi-env, multi-region setup?"
- "Module that's used in 50 places needs a breaking change. How do you ship it?"
- "Your `terraform apply` failed half-way. What does state look like? How do you recover?"
- "Where do you put secrets — Terraform variables, Vault, or somewhere else?"

## Hands-on environment

- Local Terraform CLI
- An AWS account with cheap resources (S3 bucket, DynamoDB table)
- A git repo to practice Atlantis-style PR workflow
