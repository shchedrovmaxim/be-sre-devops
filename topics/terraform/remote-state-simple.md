# Remote state with S3 + DynamoDB — the simple version (the safety deposit box)

> Read this first. Once the analogy clicks, the [deep-dive](./remote-state.md) becomes easy.

This doc explains one idea:

> **Terraform state is the record of what infrastructure you own. Remote state puts that record somewhere everyone can see it, with a lock so two people can't edit it at the same time.**

That's it. Everything else (S3 versioning, DynamoDB conditional writes, KMS encryption, crash recovery) is just making that idea production-safe.

---

## Your state file is a shared ledger at a bank

Imagine a safety deposit box at a bank that holds the official ledger of everything your company owns. When you make a change, you:

1. **Check out the ledger** (download state from S3)
2. **Make your change** (add an EC2 instance, modify an RDS config)
3. **Write it back** (upload updated state to S3)

The problem: two engineers trying to check out the ledger at the same time. Without a mechanism to prevent this, one engineer's write silently overwrites the other's — and now your state disagrees with reality.

| In the bank world | In Terraform world |
|---|---|
| The safety deposit box | S3 bucket (stores `terraform.tfstate`) |
| The ledger inside | State file — the record of all managed resources |
| The bank's single key | DynamoDB lock (only one person can apply at a time) |
| Signing in at the counter | `terraform apply` acquires the lock |
| Signing out when you leave | Lock released when apply finishes (or crashes) |
| Bank keeps photo copies | S3 versioning — previous state versions retained |

---

## The two confusing pieces

### 1. Why DynamoDB, not just S3?

S3 doesn't have a native "one writer at a time" primitive. You can't make S3 say "refuse this write if someone else is writing right now."

DynamoDB has **conditional writes** — you can say "write this item, but only if no item with this key exists." Terraform uses a `LockID` item keyed on the state path. First `apply` puts the item in. Second concurrent `apply` tries to put the same item → DynamoDB rejects it → Terraform prints an error with the existing lock's owner info and exits.

The lock is just a row in a table. That's it.

```
LockID: "my-project/terraform.tfstate"
Info:   {"Operation":"OperationTypeApply","Who":"alice@laptop","Created":"2024-01-15T14:23:00Z"}
```

### 2. What happens when an `apply` crashes mid-flight?

The apply started, created some resources, then your laptop died. The lock is still in DynamoDB — Terraform never got to release it.

Next `apply` attempt:

```
Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException
Lock Info:
  ID:        abc-123-def
  Path:      my-project/terraform.tfstate
  Operation: OperationTypeApply
  Who:       alice@laptop
  Created:   2024-01-15 14:23:00.000000000 +0000 UTC
```

You look at that timestamp. Alice's laptop crashed 3 hours ago. You verify she's not running anything. You then run:

```bash
terraform force-unlock abc-123-def
```

That deletes the DynamoDB item. Now you can re-run apply. Terraform will reconcile: resources that were already created won't be re-created (state has them); resources that weren't created will be.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Why not just commit the state file to git? | Git has no locking. Two PRs merged simultaneously corrupt state. Also: state contains secrets (passwords, certs) in plaintext. |
| What's in the state file? | Every resource Terraform manages: resource IDs, attributes, metadata. For an RDS instance: endpoint, identifier, password. |
| What does S3 versioning buy you? | If a bad apply corrupts state, roll back to the previous version. It's your undo for state. |
| What does KMS encryption buy you? | State files contain secrets. Encryption at rest stops an S3 bucket misconfiguration from leaking your database passwords. |
| What's the DynamoDB table's throughput requirement? | Almost nothing — only one write per apply, which happens infrequently. On-demand billing is fine. |
| Can the lock get stuck? | Yes, on a crash. Fix with `terraform force-unlock <lock-id>`. Always verify the previous apply is truly dead before doing this. |
| Is state itself encrypted in transit? | Yes — S3 HTTPS endpoint by default. Add `encrypt = true` in the backend config to also encrypt at rest with KMS. |

---

## Self-test (the killer question)

Out loud:

> **"Two engineers run `terraform apply` simultaneously on the same S3 + DynamoDB backend. What happens, in detail?"**

**Reference answer (intuitive version):**

"Engineer A's apply hits DynamoDB first and writes the lock item — a `LockID` row with her machine name and timestamp. Engineer B's apply tries to write the same `LockID` row — DynamoDB's conditional write rejects it because the item already exists. Terraform on B's machine prints a lock-info error with A's details and exits without modifying state or infrastructure.

When A's apply completes, Terraform deletes the lock item. If B re-runs now, she gets the lock and proceeds. If A's laptop crashes mid-apply, the lock stays in DynamoDB. B sees the stale lock, verifies A's apply is truly dead (check timestamps, ask Alice), then runs `terraform force-unlock <id>` to delete the DynamoDB item. Then re-runs apply — Terraform reconciles state with reality, skipping already-created resources."

That answer — naming the conditional write, the lock item contents, the error message, and the crash-recovery flow — is a senior-level response.

---

## Further reading / watching

- **Terraform docs: S3 backend** — search "Terraform S3 backend documentation". Official reference for all backend config options (`encrypt`, `kms_key_id`, `dynamodb_table`).
- **"How Terraform state works" — HashiCorp Learn** — search "hashicorp learn terraform state" for the official tutorial.
- **DynamoDB conditional writes** — AWS docs, "Condition Expressions" section. Understanding this explains why DynamoDB is chosen over other stores.

---

## Next: the deep-dive

When the safety-deposit-box analogy is obvious, jump to [`remote-state.md`](./remote-state.md). The deep-dive covers:

- The exact S3 backend HCL and what each option does
- DynamoDB lock mechanics at the API level (which API calls, what the error looks like)
- State file internals — what's actually in the JSON and why it contains secrets
- KMS encryption setup and cross-account key policies
- The mid-flight crash recovery flow, step by step
- Multi-region considerations
- Alternatives: Terraform Cloud, Atlantis-managed state, S3-compatible stores
- 4 self-test drills with reference answers
