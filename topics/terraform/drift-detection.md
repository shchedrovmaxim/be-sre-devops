# Drift detection — the deep-dive

> **Goal**: by the end you can answer — **"How would you detect Terraform drift across hundreds of resources?"** — naming scheduled plan in CI, `-detailed-exitcode`, JSON plan output parsing, common drift sources, remediation strategies, Atlantis/Spacelift drift surfacing, and the `ignore_changes` escape hatch.

> Start with the [simple version](./drift-detection-simple.md) if you haven't read it. The manual-override-and-blueprint analogy is the spine.

---

## The senior framing — drift is a compliance problem, not just an operational one

Mid-level engineers treat drift as a nuisance: "someone changed a thing in the console, I'll fix it next sprint."

Senior engineers know drift is a **security and compliance problem**:
- A manually changed security group rule might be the artifact of an incident response — or it might be an unauthorized access change that nobody noticed.
- An S3 bucket that was publicly blocked by Terraform, now with public access enabled (manual change), is an active data exposure.
- IAM policy drift can mean privilege escalation happened and nobody knows.

The purpose of drift detection isn't tidiness. It's **auditability**: every infrastructure change should be traceable to a Terraform PR, not to "someone's console session on a Tuesday."

A drift detection pipeline is the mechanism that enforces this by surfacing all reality-vs-code gaps before they become incidents.

---

## Scheduled `terraform plan` in CI — the full setup

### The basic script

```bash
#!/bin/bash
set -e

cd environments/prod

terraform init -backend-config="role_arn=${TERRAFORM_ROLE_ARN}"

terraform plan \
  -detailed-exitcode \
  -out=tfplan.binary \
  -json \
  2>&1 | tee plan.json

PLAN_EXIT_CODE=${PIPESTATUS[0]}

case $PLAN_EXIT_CODE in
  0)
    echo "No changes detected — infrastructure matches state"
    ;;
  1)
    echo "Error running plan"
    exit 1
    ;;
  2)
    echo "Drift detected — changes planned"
    # Parse the plan, notify, open a ticket
    python3 ./scripts/notify_drift.py plan.json
    exit 2
    ;;
esac
```

The `-out=tfplan.binary` saves the plan to a binary file (needed if you want to `apply` it later with `terraform apply tfplan.binary`). The `-json` flag separately outputs JSON to stdout. These are orthogonal — you can use both.

### GitHub Actions cron example

```yaml
name: Drift Detection

on:
  schedule:
    - cron: "0 8 * * *"   # 8 AM UTC daily
  workflow_dispatch:         # manual trigger

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('TERRAFORM_ROLE_{0}', matrix.environment)] }}
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init
        working-directory: envs/${{ matrix.environment }}

      - name: Detect Drift
        id: plan
        run: |
          terraform plan -detailed-exitcode -out=tfplan.binary 2>&1
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true
        working-directory: envs/${{ matrix.environment }}

      - name: Notify on Drift
        if: steps.plan.outputs.exit_code == '2'
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Terraform drift detected in *${{ matrix.environment }}*",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Drift detected in `${{ matrix.environment }}` — see <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|workflow run>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_DRIFT_WEBHOOK }}
```

**Why `continue-on-error: true`**: the plan step exits with code 2 on drift, which GitHub Actions interprets as failure. `continue-on-error` lets the next step run (to send the notification) without failing the whole job.

---

## Structured plan output — parsing the JSON

The `-json` flag outputs the plan as structured JSON to stdout. This is machine-parseable, unlike the human-readable format:

```json
{
  "format_version": "1.2",
  "terraform_version": "1.7.0",
  "resource_changes": [
    {
      "address": "aws_security_group.app",
      "change": {
        "actions": ["update"],
        "before": {
          "ingress": [{"from_port": 443, "to_port": 443, "protocol": "tcp", "cidr_blocks": ["0.0.0.0/0"]}]
        },
        "after": {
          "ingress": [
            {"from_port": 443, "to_port": 443, "protocol": "tcp", "cidr_blocks": ["0.0.0.0/0"]},
            {"from_port": 22,  "to_port": 22,  "protocol": "tcp", "cidr_blocks": ["0.0.0.0/0"]}  // drift!
          ]
        }
      }
    }
  ]
}
```

Each entry in `resource_changes` has an `actions` array: `["no-op"]`, `["update"]`, `["create"]`, `["delete"]`, `["create", "delete"]` (replace).

A quick parser to extract drifted resources:

```python
import json, sys

with open(sys.argv[1]) as f:
    plan = json.load(f)

drifts = [
    r for r in plan.get("resource_changes", [])
    if r["change"]["actions"] != ["no-op"]
]

for d in drifts:
    print(f"{d['address']}: {d['change']['actions']}")
```

This is useful for Slack notifications with specific resource names, for ticket creation with context, or for triage dashboards.

**Plan output sizes**: a `terraform plan -json` for a large environment (200+ resources) produces a 1–5MB JSON file. Not huge, but you wouldn't want to log it verbatim into Slack — extract the relevant `resource_changes` and summarize.

---

## Common drift sources and their patterns

### 1. Manual console changes (most common)

An ops engineer changes a security group, adjusts an RDS parameter, or modifies an S3 bucket policy during an incident. Often documented in the incident ticket but never reflected back in Terraform code.

**Pattern**: the plan shows `update` on a single resource with one or two attribute changes. The change is narrow and targeted — looks like a surgical fix.

**Remediation**: talk to the person who made the change. If the change is correct (it was the right fix), update the Terraform code to match, commit, and `terraform apply` the code version. If the change was wrong or temporary, re-apply Terraform to reset it.

### 2. Autoscaling / managed service changes

An autoscaling group's `desired_capacity` is managed by Auto Scaling policies. Terraform will plan to reset it to the coded value on every run.

**Pattern**: the plan shows `update` on autoscaling groups with `desired_capacity` changing repeatedly. Often multiple resources, cyclically.

**Remediation**: `ignore_changes = [desired_capacity]` — this is the right tool here. The autoscaler owns this attribute, not Terraform.

```hcl
resource "aws_autoscaling_group" "app" {
  min_size         = 2
  max_size         = 10
  desired_capacity = 3  # initial value; runtime-managed after first apply

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

### 3. Certificate renewals and rotations

AWS Certificate Manager rotates certificates automatically. The `not_after` timestamp on an ACM certificate changes on each renewal. Terraform will detect this as a change.

**Remediation**: `ignore_changes = [not_after]` on the certificate data source (or use a data source instead of a managed resource for certificates you don't control).

### 4. Provider-side resource updates

AWS updates a managed service (RDS engine minor version, EKS control plane patch) automatically. The attribute in state doesn't match the new version.

**Pattern**: the plan shows a replacement or update on a managed resource with a version or configuration attribute.

**Remediation**: update the Terraform code to the new value and apply — or `ignore_changes` if the attribute is dynamically managed by AWS.

### 5. External automation

A Lambda function or CloudFormation stack modifies resources that Terraform also manages. Tag changes are a common one: a cost-allocation tool adds tags; Terraform wants to remove them.

**Pattern**: `tags` attribute shows recurring drift across many resources.

**Remediation**: use `ignore_changes = [tags]` carefully (it will stop Terraform from managing any tags changes) or use `lifecycle { ignore_changes = [tags["CostCenter"]] }` to ignore only specific tag keys.

---

## Remediation strategies

### 1. Reimport

The resource exists in AWS, was created or modified manually, and you want Terraform to manage it going forward:

```bash
# Terraform 1.5+ import block (declarative, preferred)
import {
  to = aws_security_group.app
  id = "sg-0abc123def456"
}
```

Or the classic CLI method:
```bash
terraform import aws_security_group.app sg-0abc123def456
```

After import, run `terraform plan`. If it shows no changes, state matches reality — good. If it shows changes, your code differs from the imported resource's actual config — resolve the diff in code.

### 2. Accept-and-update-code

The manual change was correct. Update the Terraform code to match the current reality:

```hcl
# Before: this was in code but was manually changed in AWS
resource "aws_security_group_rule" "https_ingress" {
  cidr_blocks = ["10.0.0.0/8"]  # was this, manually changed to 10.0.0.0/16 in AWS
  # Fix: update code to match reality
  cidr_blocks = ["10.0.0.0/16"]
}
```

Then `terraform apply` — the plan should show no changes if you've correctly matched reality.

### 3. Re-apply Terraform (reset drift)

The manual change was wrong or unauthorized. Run `terraform apply` to reset the resource to the coded state. This is the "infrastructure is code, not console" stance — Terraform wins, manual change is reverted.

This should be the default for security-sensitive resources (IAM policies, security groups, public access blocks). Manual changes to these should be treated as potential security incidents and reset immediately.

### 4. The `ignore_changes` escape hatch

Use when a resource attribute is legitimately managed outside Terraform (autoscaling, external automation, managed service updates):

```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    ignore_changes = [
      tags["Billing"],       # managed by cost-allocation Lambda
      ami,                   # patched by Systems Manager
    ]
  }
}
```

**When not to use `ignore_changes`**: don't use it to hide drift you don't understand. If you're not sure why there's drift, investigate before suppressing. `ignore_changes` on a security group rule could hide an unauthorized access change.

---

## How Atlantis and Spacelift surface drift as PRs

### Atlantis

Atlantis runs `terraform plan` on PR open. For drift detection, you run it on a schedule without a PR:

```yaml
# atlantis.yaml
automerge: false

projects:
  - name: prod
    dir: envs/prod
    workflow: drift_detect

workflows:
  drift_detect:
    plan:
      steps:
        - run: terraform plan -detailed-exitcode
```

Atlantis doesn't natively open PRs on drift, but you can trigger it via its API from a CI cron:

```bash
# Trigger an Atlantis plan via webhook
curl -X POST https://atlantis.mycompany.com/events \
  -H "X-Atlantis-Token: ${ATLANTIS_TOKEN}" \
  -d '{"repo_rel_dir": "envs/prod", "workspace": "default"}'
```

### Spacelift

Spacelift has native drift detection. You enable it per-stack and configure the schedule:

```yaml
# In Spacelift UI or Terraform provider for Spacelift
drift_detection:
  enabled: true
  schedule: ["0 */6 * * *"]  # every 6 hours
  reconcile: false            # detect but don't auto-apply
  notify_on_no_changes: false # only alert when drift found
```

When drift is detected, Spacelift creates a "drift detection run" (visible in the UI) and can send notifications. With `reconcile: true`, it would automatically apply the Terraform code to correct the drift — powerful but requires high confidence in your code.

---

## `ignore_changes` — anatomy

The lifecycle meta-argument:

```hcl
resource "aws_rds_cluster" "main" {
  engine_version = "15.4"

  lifecycle {
    # Ignore specific attributes
    ignore_changes = [
      engine_version,  # AWS manages minor version upgrades
    ]

    # Or ignore all attribute changes (use with extreme caution)
    # ignore_changes = all
  }
}
```

`ignore_changes = all` means Terraform will never plan changes to this resource after initial creation. This is sometimes used for resources managed by another tool (Helm charts, CloudFormation) but Terraform needs to track. It's a last resort — it means Terraform is effectively read-only for that resource.

**The senior gotcha**: `ignore_changes` affects Terraform's plan, but not AWS reality. If you `ignore_changes` a security group rule and someone adds a dangerous inbound rule manually, Terraform will never flag it. Use only for attributes you explicitly trust external systems to manage, and review those attributes periodically.

---

## The interview answer in 60 seconds

> "I'd run a scheduled `terraform plan -detailed-exitcode` in CI for each environment — daily is typical, hourly for prod in high-compliance setups. Exit code 2 means Terraform planned changes, which means there's drift between state and reality.
>
> I'd save the plan as JSON (`-json` flag) and parse `resource_changes` to extract what drifted. That drives a Slack alert or Jira ticket with the specific resources and change types.
>
> The senior nuance is that not all drift is bad. Autoscaling desired counts change at runtime — I'd use `ignore_changes` for those. The drift that matters is unexpected: someone changed a security group, an IAM policy, or public access settings manually. For those, I'd investigate (was it an authorized hotfix?) and either update the code to match or re-apply Terraform to reset it.
>
> For tooling, Atlantis can be triggered on a schedule for plan-and-alert. Spacelift has native drift detection with a configurable schedule and the option to auto-reconcile — but I'd keep `reconcile: false` until the team has high confidence in the codebase."

---

## Self-test drills

### 1. How would you detect Terraform drift across hundreds of resources?

**Reference answer:**
- Scheduled `terraform plan -detailed-exitcode` in CI per environment (daily or hourly).
- Exit code 2 = drift detected. Save plan with `-json` for parsing.
- Parse `resource_changes` JSON to extract affected resources and action types.
- Alert via Slack/PagerDuty/Jira with specific resource list.
- Bonus: Atlantis/Spacelift can automate the scheduling and notification if already in use.

### 2. A scheduled plan finds that an RDS security group's ingress rules have an extra port 22 open. What do you do?

**Reference answer:**
- This is potentially a security incident — unexpected SSH access to an RDS instance is a red flag.
- Immediate action: determine who changed it and when (CloudTrail `AuthorizeSecurityGroupIngress` event).
- If unauthorized: re-apply Terraform immediately to reset the security group (Terraform wins). File a security incident report.
- If authorized (emergency access during an incident): confirm the emergency is over, remove the rule via Terraform code change, commit, apply.
- Never use `ignore_changes` for this — you want Terraform to flag any future manual changes to this rule.

### 3. What's the difference between `terraform import` and re-applying Terraform?

**Reference answer:**
- `terraform import`: brings a manually-created or externally-managed resource into Terraform state so Terraform can manage it going forward. Doesn't change the resource in AWS — just adds it to state.
- Re-apply Terraform: applies the coded configuration, overwriting any manual changes. Resets reality to match code.
- When to use import: resource was created manually and you want Terraform to own it; you then update code to match and verify no-op plan.
- When to re-apply: a manual change was unauthorized or incorrect and you want to reset it. Code is authoritative.
- Bonus: Terraform 1.5+ supports declarative `import` blocks in HCL, which is better than the CLI command for audit trail purposes (the import intent is documented in code).

### 4. How would you handle drift on an autoscaling group's `desired_capacity`?

**Reference answer:**
- `desired_capacity` is managed by Auto Scaling policies at runtime — it changes constantly as scale-out/in events happen.
- Terraform detecting this as drift on every plan is noise that drowns out real drift signals.
- Fix: `ignore_changes = [desired_capacity]` in the `lifecycle` block. Terraform still manages `min_size` and `max_size` (which are policy constraints, not runtime state), but ignores the runtime `desired_capacity` value.
- Broader principle: only `ignore_changes` on attributes where the external system that manages them is explicitly trusted. Document why you're ignoring each attribute.

---

## Further reading / watching

- **Terraform docs: `-detailed-exitcode`** — `developer.hashicorp.com/terraform/cli/commands/plan`. The flag and its exit code behavior.
- **Terraform docs: `lifecycle` meta-arguments** — `developer.hashicorp.com/terraform/language/meta-arguments/lifecycle`. Full reference for `ignore_changes`, `create_before_destroy`, `prevent_destroy`.
- **Terraform docs: importing resources** — `developer.hashicorp.com/terraform/language/import`. Declarative import blocks (1.5+) vs the `terraform import` CLI command.
- **Atlantis docs: drift detection** — `runatlantis.io/docs/`. Check the "How to use Atlantis for drift detection" section.
- **Spacelift docs: drift detection** — `docs.spacelift.io`. Their native drift detection is one of their differentiating features.
- **AWS CloudTrail** — for investigating who made manual changes. The `LookupEvents` API with `eventName=AuthorizeSecurityGroupIngress` (or similar) is the audit trail for most drift sources.

---

## The 4 dimensions (senior framing)

- **Tech**: scheduled `plan -detailed-exitcode`, exit code 2 = drift, JSON plan parsing via `resource_changes`, `ignore_changes` for expected runtime variation, `terraform import` for manual-resource reconciliation. Atlantis/Spacelift for managed scheduling and notification.
- **People**: drift detection is a team norm, not a platform feature. Engineers need to understand that manual console changes are visible and will trigger drift alerts. The policy question — "what happens when drift is found?" — needs to be answered upfront. A surprise 3 AM alert about drift in prod that nobody knows how to triage is worse than no drift detection.
- **CI/CD**: drift detection is a read-only CI job (plan only, never apply). Keep it separate from the apply pipeline so a drift-detection failure doesn't block deployments. Run it on a schedule (cron), not on PRs. Alerts go to a dedicated channel, not into the general CI noise.
- **Operations**: drift in security-sensitive resources (IAM, security groups, bucket policies) should trigger immediate investigation, not a ticket for next sprint. Write a runbook: "Drift detected in prod security group — check CloudTrail, confirm with on-call, apply or escalate within 1 hour." Track drift frequency as a metric — increasing drift rate is a signal that the team is working around Terraform rather than through it (culture issue, not just a tooling issue).
