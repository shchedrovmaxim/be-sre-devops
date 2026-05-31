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

(none yet — to be written)

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
