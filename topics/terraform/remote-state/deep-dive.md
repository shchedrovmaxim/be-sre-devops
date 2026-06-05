# Remote state with S3 + DynamoDB locking — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Two engineers run `terraform apply` simultaneously on a state with S3 backend + DynamoDB locking. What happens in detail?"** — naming the DynamoDB conditional write, lock item contents, error message, the crash-recovery flow, state encryption at rest, and S3 versioning as the undo mechanism.

> Start with the [simple version](./simple.md) if you haven't read it. The safety-deposit-box analogy is the spine of everything here.

---

## The senior framing — state is the most dangerous file in your infrastructure

Mid-level engineers think Terraform state is just a cache — "it just tracks what Terraform made."

Senior engineers know state is **the ground truth Terraform uses to plan every future change**. If state diverges from reality (drift), Terraform can destroy resources that exist or create resources that already exist. If state is corrupted, you've lost the ability to safely manage your infrastructure until you reconstruct it — manually, resource by resource.

The two failure modes that keep seniors up at night:
1. **Concurrent modification** — two applies running simultaneously, second overwrites first's plan, infrastructure is now undefined.
2. **State corruption** — a crash mid-apply leaves state partially written; now state says you have 3 subnets but you only have 2.

S3 + DynamoDB solves both. Not perfectly, but well enough for almost every use case.

---

## The minimal setup

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-tfstate"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/mrk-abc123"
    dynamodb_table = "terraform-state-locks"
  }
}
```

The DynamoDB table needs exactly one attribute: a `LockID` string as the partition key. No sort key, no other attributes. On-demand billing is fine — writes happen at apply frequency, which is low.

```hcl
# Create the table (typically bootstrapped separately, outside Terraform)
resource "aws_dynamodb_table" "tfstate_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**The bootstrap problem**: you can't use Terraform to create its own state backend. The S3 bucket and DynamoDB table must exist before you run `terraform init`. Most teams create them once with the AWS CLI or a separate "bootstrap" Terraform workspace with a local backend, then never touch them again.

---

## What's actually in the state file

The state file is plain JSON. Knowing what's in it explains why it must be encrypted:

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 42,
  "lineage": "6c4f5a8b-...",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_db_instance",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "identifier": "prod-postgres-main",
            "endpoint": "prod-postgres-main.xxxx.us-east-1.rds.amazonaws.com",
            "password": "p@ssw0rd-in-plaintext",
            "username": "admin"
          }
        }
      ]
    }
  ]
}
```

The RDS password is in plaintext. So is any `aws_secretsmanager_secret_version` value you've created. **State contains secrets by design** — Terraform needs to know the current values to plan diffs. This is why:

1. `encrypt = true` in the backend is non-negotiable in production
2. The S3 bucket must block public access
3. Access to the S3 bucket = access to every secret Terraform manages

`serial` is the state revision number. Each apply increments it. If two applies ran concurrently and somehow both wrote state (theoretical without locking), Terraform would detect the serial mismatch and reject the lower-serial write. DynamoDB locking is the first line of defense; serial mismatch is a second, weaker backstop.

`lineage` is a UUID stamped when the state is first created. It ties all versions of a state file to the same lineage — a guard against accidentally mixing state files from different environments.

---

## DynamoDB locking — the mechanics

When `terraform apply` starts, Terraform calls DynamoDB's `PutItem` with a `ConditionExpression`:

```
ConditionExpression: "attribute_not_exists(LockID)"
```

This means: "write this item, but only if no item with this partition key exists." If the item is already there (from a concurrent apply), DynamoDB rejects with `ConditionalCheckFailedException`.

The item Terraform writes:

```json
{
  "LockID": { "S": "my-company-tfstate/prod/networking/terraform.tfstate" },
  "Info": {
    "S": "{\"ID\":\"abc-123-def\",\"Operation\":\"OperationTypeApply\",\"Info\":\"\",\"Who\":\"alice@laptop\",\"Version\":\"1.7.0\",\"Created\":\"2024-01-15T14:23:00Z\",\"Path\":\"prod/networking/terraform.tfstate\"}"
  }
}
```

The `LockID` is the full path of the state file in S3 — this means multiple different state files in the same bucket each have their own independent lock row. One table works for an entire organization.

When the apply finishes (success or graceful failure), Terraform calls `DeleteItem` on the lock. If Terraform crashes or the machine dies, the `DeleteItem` never runs — the item stays in DynamoDB until manually removed.

### What the error looks like

```
Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException

  Lock Info:
    ID:        abc-123-def-456
    Path:      prod/networking/terraform.tfstate
    Operation: OperationTypeApply
    Who:       alice@company-laptop
    Version:   1.7.0
    Created:   2024-01-15 14:23:00.234521476 +0000 UTC
    Info:

Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

The `Who` field tells you exactly which machine holds the lock and when it started. For the stale-lock recovery, that timestamp is how you determine whether the apply is still running or the machine died.

---

## The crash-recovery flow

This is a senior-level question in itself: "your `terraform apply` failed half-way through. What does state look like and how do you recover?"

**What happened**: Terraform applied 4 of 7 planned changes. On the 5th, the machine died (or the network dropped, or Terraform was killed with SIGKILL).

**State now**:
- S3 contains the state *as of the last successful operation before the crash*. If Terraform had written partial updates, the state could show 4 resources created. The state may or may not reflect the 5th.
- DynamoDB still has the lock item — Terraform never reached the `DeleteItem`.

**Recovery steps**:

1. **Verify the apply is truly dead.** Check the machine, the CI system, wherever it ran. A false unlock of an active apply is destructive.

2. **Force-unlock**:
   ```bash
   terraform force-unlock abc-123-def-456
   ```
   This deletes the DynamoDB item. It does not modify S3 state.

3. **Run `terraform plan`** — not apply yet. This shows you the diff between state and reality. Resources that were created but not in state will need to be imported. Resources that are in state but don't exist will be planned for creation.

4. **If resources exist in AWS but not in state** (created by the partial apply but Terraform crashed before writing them to state), import them:
   ```bash
   terraform import aws_subnet.private_a subnet-0abc123
   ```

5. **If state is corrupted beyond repair** — this is rare but happens. Fall back to S3 versioning: find the last known-good version, download it, push it back as the current state. Then re-plan.

6. **Re-run `terraform apply`** with the repaired state.

**The "partial apply" gotcha**: Terraform does not guarantee atomicity. A plan with 10 changes will apply them sequentially. If it stops at #5, changes 1–5 happened in AWS, changes 6–10 didn't, and state reflects whatever Terraform managed to write before dying. This is expected, not a bug — it's why the recovery flow exists.

---

## S3 versioning — the undo

Enable versioning on the state bucket:

```hcl
resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

Every `terraform apply` writes a new version of the state file to S3. S3 versioning keeps the old versions. If a bad apply corrupts state:

```bash
# List versions
aws s3api list-object-versions \
  --bucket my-company-tfstate \
  --prefix prod/networking/terraform.tfstate

# Download the version before the bad one
aws s3api get-object \
  --bucket my-company-tfstate \
  --key prod/networking/terraform.tfstate \
  --version-id "PREVIOUS_VERSION_ID" \
  terraform.tfstate.backup

# Inspect it, then push it back as current
aws s3 cp terraform.tfstate.backup \
  s3://my-company-tfstate/prod/networking/terraform.tfstate
```

**State file sizes** in practice: a typical state file for a mid-size module (VPC + subnets + security groups + RDS + a few EKS node groups) runs 500KB–2MB. A very large platform state file with hundreds of resources can reach 10–20MB. This matters for two reasons: S3 GET latency on large state (usually not a problem), and readability — if your state is 20MB, you need to split your modules.

---

## KMS encryption at rest

`encrypt = true` in the backend config enables S3 server-side encryption. By default this uses SSE-S3 (AWS-managed keys). For production, use a customer-managed KMS key (`kms_key_id`) so you can:

- Audit every state read/write via CloudTrail (KMS API calls are logged)
- Rotate the key on a schedule
- Revoke access at the key level in an emergency (revoke the key → nobody can read state → break-glass procedure)

The KMS key policy is a common gotcha. The bucket policy allows certain IAM roles to access S3, but those roles also need `kms:Decrypt` and `kms:GenerateDataKey` permissions on the key. Missing the KMS grants is a common "why can't CI apply state?" support ticket.

```hcl
# IAM policy for the CI/CD role
data "aws_iam_policy_document" "tfstate_access" {
  statement {
    actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"]
    resources = [
      aws_s3_bucket.tfstate.arn,
      "${aws_s3_bucket.tfstate.arn}/*"
    ]
  }
  statement {
    actions   = ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"]
    resources = [aws_dynamodb_table.tfstate_locks.arn]
  }
  statement {
    actions   = ["kms:Decrypt", "kms:GenerateDataKey"]
    resources = [aws_kms_key.tfstate.arn]
  }
}
```

---

## Multi-region considerations

If you're running infrastructure in multiple AWS regions, you have two choices:

**Option A — One S3 bucket in us-east-1, all state files in it.** State files for eu-west-1 resources still live in the us-east-1 bucket. Works fine — the state bucket's region doesn't need to match the resources it describes. Simpler. The risk: if us-east-1 is down, you can't apply state for any region. In practice, S3's regional availability means this is acceptable for most organizations.

**Option B — State buckets per region.** Each region has its own S3 bucket and DynamoDB table. More complex bootstrapping. Required if your compliance posture demands data residency (state contains secrets → state must stay in the EU for GDPR, for example).

**Cross-account state**: When one account's Terraform needs to read outputs from another account's state, use the `terraform_remote_state` data source. The accessing role needs cross-account S3 read access and KMS decrypt on the source account's key.

```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket  = "shared-networking-tfstate"
    key     = "prod/vpc/terraform.tfstate"
    region  = "us-east-1"
    role_arn = "arn:aws:iam::NETWORKING_ACCOUNT:role/TerraformStateReader"
  }
}

# Access outputs from the networking state
locals {
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id
}
```

This is the standard pattern for multi-account setups — the networking account owns the VPC, app accounts reference it by reading state.

---

## Alternatives

### Terraform Cloud / HCP Terraform

HashiCorp's hosted offering. State is managed by HCP, no S3 or DynamoDB to run yourself. Lock and unlock are built in. Free tier for small teams. Cost: ~$20/resource-month for the paid tier, which adds remote execution, SSO, Sentinel policy enforcement.

Senior trade-off: you gain operational simplicity (no bucket to manage), you lose ownership (state lives in HashiCorp's infrastructure; you're dependent on their uptime; sensitive state is on their servers unless you use a HashiCorp private cloud deployment).

### Atlantis-managed state

Atlantis is a self-hosted PR automation tool (more in `pr-workflows.md`). It runs `terraform plan` on PRs and `terraform apply` on merge. It doesn't replace the S3 backend — it still uses S3 + DynamoDB for state. But it does add an apply-locking mechanism at the PR level: only one PR can be applying per workspace at a time. Atlantis's locking is on top of DynamoDB locking, not instead of it.

### OpenTofu

The open-source Terraform fork. State mechanics are identical to Terraform — same S3 backend, same DynamoDB locking. Relevant if your organization has decided to fork away from HashiCorp's BSL license.

---

## The interview answer in 60 seconds

> "Engineer A hits `terraform apply`. Terraform's first call is `PutItem` to DynamoDB with `ConditionExpression: attribute_not_exists(LockID)` — it writes a lock item containing the operator's identity and timestamp. Engineer B then hits apply. Terraform calls the same `PutItem`, DynamoDB sees the item already exists, returns `ConditionalCheckFailedException`. Terraform on B's machine prints the lock-info error with A's details and exits cleanly. Infrastructure and state are untouched.
>
> When A finishes, Terraform calls `DeleteItem`. B can now acquire the lock and proceed.
>
> If A's machine crashes, the `DeleteItem` never runs. Stale lock. B sees it, checks the timestamp and confirms A's apply is dead, then runs `terraform force-unlock <id>` to delete the DynamoDB item manually. Re-runs apply. Terraform reconciles state with reality — resources already created are skipped; remaining resources are applied.
>
> For production, I'd add S3 versioning (undo for corrupt state), KMS encryption (state contains plaintext secrets), and IAM policies that include `kms:Decrypt` on the key — the missing KMS grant is the most common CI/CD 'why can't I apply' debugging session."

---

## Self-test drills

### 1. Two engineers apply simultaneously. What happens, in detail?

**Reference answer:**
- Engineer A acquires the DynamoDB lock via conditional `PutItem` (`attribute_not_exists(LockID)`).
- Engineer B attempts the same `PutItem`, DynamoDB rejects → `ConditionalCheckFailedException`.
- Terraform on B's side prints the lock-info error (who, when, which path) and exits without modifying anything.
- When A finishes, Terraform calls `DeleteItem`, releasing the lock.
- B can re-run and will now succeed.
- Bonus: the S3 state's `serial` field would also catch a concurrent write if locking somehow failed — Terraform rejects a write whose serial doesn't match what it read.

### 2. `terraform apply` crashed mid-flight. What does state look like? How do you recover?

**Reference answer:**
- State reflects whatever Terraform successfully wrote before the crash — could be partial.
- DynamoDB lock item still exists (DeleteItem never ran).
- Recovery: verify the crashed apply is truly dead → `terraform force-unlock <id>` → run `terraform plan` to see the diff → `terraform import` any resources that were created but aren't in state → re-run `terraform apply`.
- If state itself is corrupt: use S3 versioning to roll back to the previous version.
- Terraform does not guarantee atomicity; partial applies are normal; recovery from them is a known operational procedure.

### 3. Why does Terraform state contain plaintext secrets? What do you do about it?

**Reference answer:**
- Terraform stores the full current attributes of every resource it manages, including sensitive fields like passwords and private keys — it needs these to plan diffs.
- Mitigations: `encrypt = true` (S3 SSE), customer-managed KMS key (audit trail via CloudTrail, revocation capability), strict bucket policy blocking public access, IAM least-privilege for state access.
- Alternative approach: store secrets in AWS Secrets Manager / Vault and only reference them by ARN in Terraform — the Terraform state then only holds the ARN, not the secret value. For new secrets generated by Terraform (like an RDS password), use `aws_secretsmanager_secret_version` with a separately generated value.
- Bonus: `sensitive = true` on Terraform outputs redacts the value from CLI output but does NOT protect it in state — it's still plaintext JSON in S3.

### 4. How would you structure state for a multi-account AWS setup?

**Reference answer:**
- Separate state files per account + per layer (networking, platform, app). Don't put everything in one state file — blast radius and state file size both get out of hand.
- Shared resources (VPCs, IAM roles) in a "networking" or "platform" state; app teams read outputs from it via `terraform_remote_state` data source.
- Cross-account S3 access: the reading account's IAM role needs explicit cross-account grants and KMS decrypt permission on the source key.
- State bucket: either one central bucket (simpler) or one per account (data residency/compliance). Central bucket with per-account prefixes is the most common pragmatic choice.
- Bonus: S3 bucket must have versioning enabled, block-all-public-access, and MFA delete for the state bucket in regulated environments.

---

## Further reading / watching

- **Terraform docs: S3 backend** — official documentation at `developer.hashicorp.com/terraform/language/settings/backends/s3`. Covers every configuration option.
- **"How Terraform State Works" — HashiCorp blog** — search for this title on `hashicorp.com`. The canonical post on state internals.
- **AWS docs: DynamoDB condition expressions** — `docs.aws.amazon.com`, search "DynamoDB ConditionExpression". Understanding the underlying mechanism makes the locking model obvious.
- **Brendan Thompson — "Terraform at Scale" talk** — search YouTube. Good walkthrough of multi-account state structuring.

---

## The 4 dimensions (senior framing)

- **Tech**: S3 backend (state file JSON, versioning as undo), DynamoDB locking (conditional `PutItem`, `LockID` item), KMS encryption (audit trail + revocation), `terraform_remote_state` for cross-account output sharing. State serial and lineage as secondary safety nets.
- **People**: State corruption is a major operational incident — when something goes wrong with state, it's all-hands. Document the crash-recovery runbook before you need it. Everyone on the team should know `terraform force-unlock` and the S3 version rollback procedure.
- **CI/CD**: CI/CD roles need exactly S3 read/write, DynamoDB read/write, and KMS decrypt — not AdministratorAccess. Apply the bucket policy and IAM in Terraform itself (bootstrapped separately). Consider a dedicated "terraform-ci" role per environment rather than sharing credentials.
- **Operations**: Enable S3 versioning from day one — retrofitting it later means the version history starts from when you turned it on, not from the beginning. Monitor the DynamoDB table for stuck locks (CloudWatch alarm on items older than 2 hours that aren't from a known-long apply). Alert on failed applies in CI — a failed apply that created partial resources is an incident, not just a failed job.
