# Drift detection — the simple version (the manual override)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) has all the mechanics.

One central idea:

> **Drift is when reality disagrees with your Terraform state. The most common cause is a human logging into the AWS console and changing something. The fix is a scheduled `terraform plan` that finds the gap before it causes an incident.**

---

## The manual override problem

Imagine you're managing a building's electrical system with a blueprint. Every room's wiring is mapped. Someone needs to add an outlet in room 3B urgently — no time to update the blueprint, they just do it.

Six months later, the blueprint says room 3B has two outlets. Reality has three. Now someone runs a renovation based on the blueprint, and the third outlet's circuit gets cut — unexpected behavior, possibly dangerous.

| In the building world | In Terraform world |
|---|---|
| The blueprint | Terraform state (and code) |
| The manual outlet addition | A human changing an AWS resource in the console |
| The renovation based on old blueprint | `terraform apply` that plans based on stale state |
| "Wait, where did this outlet come from?" | `terraform plan` showing unexpected diffs |
| Checking every room against the blueprint weekly | Scheduled `terraform plan` in CI |

Drift is the gap between the blueprint and reality. Detection is running `terraform plan` regularly to find that gap.

---

## The two confusing things

### 1. Drift isn't always bad — sometimes it's a data source

Sometimes drift is intentional: your autoscaling group changes the `desired_capacity` at runtime, and you don't want Terraform to fight it back to the code value. Sometimes an operations team makes a hotfix in the console during an incident, with full intent to codify it later.

The question is always: **does this diff represent a problem, or expected variation?**

```hcl
# Tell Terraform "I know this will change, don't try to reset it"
resource "aws_autoscaling_group" "app" {
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

`ignore_changes` is the "I know there's drift here and it's intentional" escape hatch. Don't use it to hide accidental drift — use it only when the change is expected at runtime.

### 2. `terraform plan` exits with code 2, not 1, when it finds a diff

When you run `terraform plan` and it finds changes to make, it exits with code **2** (not 0, not 1). This is the key to detecting drift in CI:

```bash
terraform plan -detailed-exitcode
# exit code 0 = no changes (no drift)
# exit code 1 = error (something went wrong)
# exit code 2 = changes present (drift found)
```

In a CI script:
```bash
terraform plan -detailed-exitcode -out=tfplan.binary
EXIT_CODE=$?
if [ $EXIT_CODE -eq 2 ]; then
  echo "Drift detected! Plan output saved to tfplan.binary"
  # alert, open a ticket, notify Slack
fi
```

This is the entire drift-detection mechanism: scheduled CI job, `-detailed-exitcode`, alert on exit code 2.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What causes drift? | Manual changes in AWS console, direct AWS CLI without Terraform, autoscaling, external automation |
| How do you detect it? | Scheduled `terraform plan -detailed-exitcode` in CI; exit code 2 = drift |
| How often should you run it? | Daily for most environments; hourly for prod in security-conscious setups |
| What do you do when you find drift? | Investigate: was it intentional? If yes, update the code and `ignore_changes`. If no, either re-apply Terraform or import the manual change. |
| What's `ignore_changes`? | A lifecycle block telling Terraform to ignore specific attributes at plan time — "I know this changes, don't reset it." |
| What's `terraform import`? | Bring an existing AWS resource under Terraform management by adding it to state. Used when a resource was created manually and you want Terraform to own it going forward. |

---

## Self-test (the killer question)

Out loud:

> **"How would you detect Terraform drift across hundreds of resources?"**

**Reference answer (intuitive version):**

"I'd run a scheduled `terraform plan -detailed-exitcode` in CI for each environment — daily is typical, hourly for prod. Exit code 2 means changes were planned, which means drift. I'd save the plan output as JSON (`-json` flag alongside `-out`) and send an alert to a Slack channel or open a Jira ticket automatically.

The senior nuance: not all drift is bad. Autoscaling group desired counts change at runtime; I'd use `ignore_changes` for those. The drift that matters is unexpected — someone changed a security group rule, an S3 bucket's public access settings, or an IAM policy manually. A daily plan catches those before they become a compliance issue or an incident."

---

## Further reading / watching

- **Terraform docs: `-detailed-exitcode`** — `developer.hashicorp.com/terraform/cli/commands/plan`. The flag that makes drift detection scriptable.
- **`ignore_changes` lifecycle docs** — `developer.hashicorp.com/terraform/language/meta-arguments/lifecycle`. When and how to use it without hiding real problems.
- **Atlantis docs: drift detection** — `runatlantis.io`. Atlantis has a built-in drift detection mode that opens PRs when drift is found.

---

## Next: the deep-dive

When the scheduled-plan mechanism and the exit-code pattern are obvious, jump to [`drift-detection.md`](./deep-dive.md). The deep-dive covers:

- Scheduled plan in CI — the full setup (cron, exit codes, JSON output parsing)
- Structured plan output with `-json` — what the JSON contains and how to parse it
- Common drift sources and their patterns
- Remediation strategies: reimport vs accept-and-update-code vs re-apply
- How Atlantis and Spacelift surface drift as PRs
- `ignore_changes` lifecycle blocks — the escape hatch and when not to use it
- 4 self-test drills with reference answers
