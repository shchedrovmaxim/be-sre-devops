# SOPS + age/KMS — the deep-dive

> **Goal**: by the end you can answer — **"Why is SOPS better than just encrypting whole files for Kubernetes manifests?"** — covering per-value encryption and diff-readability, the backend options (age vs KMS), the ArgoCD integration story, and the governance tradeoffs.

> Start with the [simple version](./sops-simple.md) if you haven't. The redacted-document analogy is the mental model.

---

## The senior framing — SOPS trades operational simplicity for code-review hygiene

Sealed Secrets is cluster-centric (the cluster holds the key). ESO is store-centric (the external store holds the truth). SOPS is repo-centric — the encrypted file is the artifact, and decryption happens at the point of use: locally for devs, at sync time for ArgoCD, at apply time for CI pipelines.

What makes SOPS senior-notable:

1. **Diff-friendliness** — the thing that makes code review on secrets possible at all.
2. **No runtime dependency** — unlike ESO, there's no controller polling an external store. ArgoCD decrypts at sync time using a plugin; after that the manifest is a plain K8s resource.
3. **Key portability** — one encryption key can cover multiple clusters (unlike Sealed Secrets, where each cluster has its own keypair).
4. **The governance question is explicit** — "who can decrypt" must be answered via IAM policy or key distribution, which forces the conversation.

---

## The encryption mechanics — DEK + KEK hybrid pattern

SOPS uses hybrid encryption. You need to understand this to explain it in an interview.

For each file SOPS encrypts:

1. **SOPS generates a random Data Encryption Key (DEK)**: 256-bit AES-GCM key, unique per file.
2. **SOPS encrypts each value** in the YAML using the DEK (AES-256-GCM with authentication).
3. **SOPS encrypts the DEK** using the Key Encryption Key (KEK) — your `age` keypair or KMS CMK.
4. **The encrypted DEK is stored in the `sops` metadata block** in the file alongside the encrypted values.

To decrypt:
1. SOPS reads the encrypted DEK from the `sops` block.
2. SOPS decrypts the DEK using the KEK (age private key, or KMS Decrypt API call).
3. SOPS uses the DEK to decrypt each value.

Why this design matters:
- **Re-encryption without full decryption**: adding a new recipient (a new `age` public key) means re-encrypting the DEK for that recipient — the values themselves don't need to be touched.
- **KMS never sees your data**: when using AWS KMS, the `Decrypt` API call only decrypts the small DEK, not the actual secret values. The values never leave your machine.
- **Multiple recipients**: SOPS supports key groups — you can encrypt the DEK for multiple recipients. Any one of them can decrypt.

---

## Backends — age vs KMS, with concrete trade-offs

### age

```bash
# Generate a keypair
age-keygen -o key.txt
# Public key:  age1abc123...
# Private key: AGE-SECRET-KEY-1...

# Encrypt a file
sops --encrypt --age age1abc123... secret.yaml > secret.enc.yaml

# Decrypt
SOPS_AGE_KEY_FILE=./key.txt sops --decrypt secret.enc.yaml
```

Pros: zero cloud dependency, zero IAM config, works offline, simple.
Cons: key distribution is manual — the private key must be shared out-of-band (e.g., 1Password or a team vault), and if a team member leaves, you must re-encrypt all files with a new key (or use key groups where you can remove one recipient).

Best for: solo developers, small teams, self-hosted environments, local dev.

### AWS KMS

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/production/.*
    kms: arn:aws:kms:us-east-1:123456789012:key/abc123-...
  - path_regex: secrets/dev/.*
    kms: arn:aws:kms:us-east-1:123456789012:key/def456-...
```

```bash
# Encrypt (requires AWS credentials with kms:Encrypt on the key)
sops --encrypt secret.yaml > secret.enc.yaml

# Decrypt (requires kms:Decrypt on the key)
sops --decrypt secret.enc.yaml
```

Pros: "who can decrypt" is governed by IAM + KMS key policies. Audit trail in CloudTrail for every `Decrypt` call. Key never leaves KMS. Rotation is managed by KMS.
Cons: requires AWS credentials; doesn't work offline; latency on every decrypt (~50-200ms for the KMS API call).

Best for: teams with AWS infrastructure; situations where audit trails and centralized key management are required.

### age + KMS together (key groups)

```yaml
creation_rules:
  - path_regex: .*
    key_groups:
      - age:
          - age1abc123...   # dev 1
          - age1def456...   # dev 2
        kms:
          - arn:aws:kms:us-east-1:...:key/...  # AWS KMS for CI/CD
```

Key groups allow M-of-N threshold decryption. In the above: the DEK is encrypted separately for both age recipients and the KMS key. Any single key is sufficient to decrypt (threshold = 1). You can require all keys (threshold = N) for high-security scenarios.

This is the pattern for a team where devs use age keys locally and CI/CD uses KMS.

---

## `.sops.yaml` — creation rules

`.sops.yaml` sits at the repo root and tells SOPS which key to use for files matching path patterns.

```yaml
creation_rules:
  - path_regex: .*/production/.*\.(yaml|yml)$
    kms: arn:aws:kms:us-east-1:123456789012:key/prod-key-id
    encrypted_regex: ^(data|stringData)$   # only encrypt these YAML keys

  - path_regex: .*/staging/.*\.(yaml|yml)$
    kms: arn:aws:kms:us-east-1:123456789012:key/staging-key-id
    encrypted_regex: ^(data|stringData)$

  - path_regex: .*/dev/.*\.(yaml|yml)$
    age: age1abc123...,age1def456...       # comma-separated age recipients
    encrypted_regex: ^(data|stringData)$
```

`encrypted_regex` is important for Kubernetes manifests: you only want to encrypt `data` and `stringData` fields, not `metadata`, `apiVersion`, `kind`, etc. Without this, SOPS would encrypt the entire YAML tree, breaking the diff-readability benefit.

---

## ArgoCD + KSOPS integration

This is the most common production setup. How it works:

1. ArgoCD is configured with a custom plugin: KSOPS (Kustomize + SOPS). KSOPS is a Kustomize plugin that calls SOPS at manifest generation time.
2. The ArgoCD `Application` uses `plugin: ksops` in the source config.
3. When ArgoCD syncs, it runs KSOPS, which:
   a. Reads the `kustomization.yaml` in your repo.
   b. Finds references to SOPS-encrypted files.
   c. Calls `sops --decrypt` for each, using the credentials available to the ArgoCD pod (typically IRSA for AWS KMS, or an age key mounted as a Secret).
   d. Passes the decrypted manifests to Kustomize.
   e. ArgoCD applies the resulting plaintext manifests to the cluster.

```yaml
# kustomization.yaml (in your GitOps repo)
generators:
  - ksops.yaml   # KSOPS generator config

# ksops.yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: ksops-generator
files:
  - ./secrets/production/db-credentials.enc.yaml
  - ./secrets/production/api-keys.enc.yaml
```

The decrypted values never touch Git. ArgoCD decrypts in-memory at sync time and applies the plain Kubernetes `Secret` directly to the cluster.

**The auth setup for ArgoCD + KMS**: annotate the ArgoCD repo-server `ServiceAccount` with an IRSA role that has `kms:Decrypt` on the relevant KMS keys. The same OIDC exchange as ESO.

**For age**: mount the age private key as a Kubernetes `Secret` in the ArgoCD namespace and set `SOPS_AGE_KEY_FILE` env var on the repo-server. The age key itself needs to be protected — ironic, but you're bootstrapping a small number of keys to protect many.

---

## The local dev workflow

Day-to-day developer experience:

```bash
# First-time setup: get the .sops.yaml from the repo
# For age: get the private key from 1Password / team vault
# For KMS: ensure AWS credentials are configured (profile, envvar, or IRSA)

# Decrypt a file for editing (in-place, original overwritten)
sops --decrypt --in-place secrets/dev/db-credentials.enc.yaml

# OR open the editor directly (SOPS handles encrypt/decrypt around the edit)
sops secrets/dev/db-credentials.enc.yaml
# Opens your $EDITOR with decrypted content; on save, SOPS re-encrypts

# Re-encrypt after manual edit
sops --encrypt --in-place secrets/dev/db-credentials.yaml

# Add a new recipient (e.g., new team member's age key) — no full re-encrypt needed
sops updatekeys secrets/dev/db-credentials.enc.yaml
# SOPS re-encrypts just the DEK for the new recipient set from .sops.yaml
```

The `sops <file>` pattern (opens in editor) is the ergonomic workflow. You get a decrypted view, edit, and SOPS re-encrypts on save. The file on disk is always encrypted.

---

## Failure modes

| Scenario | What happens | Recovery |
|---|---|---|
| age private key lost | All files encrypted for that key only are permanently unrecoverable | Use key groups (multiple recipients) so one lost key doesn't lose everything |
| KMS key deleted (not disabled — deleted) | 7-30 day waiting period in AWS before deletion. Files recoverable during that window. | Enable key deletion protection; use key aliases so you can swap the underlying key |
| KMS key policy changed (IAM revoked) | Next decrypt attempt fails. ArgoCD sync fails if KSOPS can't decrypt. | Fix IAM policy. ArgoCD will retry on next sync. |
| ArgoCD KSOPS plugin misconfigured | ArgoCD can't generate manifests; sync fails with plugin error | Check ArgoCD repo-server logs; verify KSOPS binary is in the image |
| `.sops.yaml` creation rule mismatch | SOPS uses the wrong key for a file | Always use `sops --encrypt` on new files and verify the `sops` metadata block shows the right key ARN/recipient |
| Team member leaves, has age private key | They can still decrypt files they encrypted. | Use KMS for shared team secrets; for age, rotate (re-encrypt all files with a new key, remove old recipient) |

---

## The "who can decrypt" governance framework

This is the senior question. For a team using SOPS, decide upfront:

| Scenario | Recommendation |
|---|---|
| Solo developer | age keypair, stored in 1Password. Simple. |
| Small team (2-5) | age keypair per person in `.sops.yaml` key_groups + one KMS key for CI/CD |
| Growing team (10+) | AWS KMS exclusively; access controlled via IAM roles and groups. CloudTrail audit on every decrypt. |
| Multi-team, compliance needs | KMS + Vault as the KEK; SOPS for in-Git storage of encrypted secrets, Vault for access policy. |
| Someone leaves | For age: remove their key from `.sops.yaml`, run `sops updatekeys` on all files; for KMS: revoke IAM access. |

The senior insight: the governance answer should match the team's operational maturity. A small team forced to use KMS will constantly fight IAM issues and lose the simplicity win. A large team using age keys will struggle with onboarding, key rotation, and audit trail.

---

## The interview answer in 60 seconds

> "Encrypting whole files turns them into blobs. You lose the ability to diff them in Git — every change looks like a completely different ciphertext, even if you only changed one value. You can't do meaningful code review on a blob.
>
> SOPS encrypts only the values, leaving the keys in the clear. In a Git diff you can see 'the password field changed' without seeing what the new value is. That makes secrets-in-Git reviewable — which is the whole point of GitOps.
>
> For the backend: age is the simple choice (one keypair, no cloud dependency), good for single developers or small teams. KMS gives you IAM-controlled access, CloudTrail audit trails, and centralized key rotation — right for teams with compliance requirements. For ArgoCD, the KSOPS plugin runs SOPS at sync time to decrypt before applying manifests, so ArgoCD itself never stores plaintext.
>
> The trade-off versus Sealed Secrets: SOPS keys are not cluster-specific, so one key can decrypt across all clusters. But key management is your responsibility — there's no controller to hold the key. That 'who can decrypt' governance question is explicitly yours to answer."

---

## Self-test drills

### 1. Why is SOPS better than encrypting whole YAML files?

**Reference answer:**
- Per-value encryption leaves keys readable. Git diffs show which fields changed.
- Code review becomes possible: reviewers can check naming conventions, structure, presence of required fields.
- Merge conflict resolution is possible on the structural level.
- Whole-file encryption produces an opaque blob — a change to one value looks identical to a change to all values.

### 2. Explain the DEK + KEK pattern SOPS uses.

**Reference answer:**
- SOPS generates a random 256-bit DEK (Data Encryption Key) per file.
- Values are encrypted with the DEK (AES-256-GCM).
- The DEK itself is encrypted with the KEK (Key Encryption Key) — the age private key or KMS CMK.
- The encrypted DEK is stored in the `sops` metadata block in the file.
- Benefit: the KEK (KMS) never sees your actual secret data. You can add new recipients by re-encrypting just the DEK. Rotating the KEK doesn't require re-encrypting values.

### 3. How does ArgoCD decrypt SOPS-encrypted secrets at sync time?

**Reference answer:**
- KSOPS plugin (for Kustomize) or helm-secrets (for Helm) runs as a plugin in the ArgoCD repo-server.
- At sync time, the plugin calls `sops --decrypt` on encrypted files.
- For KMS: the ArgoCD repo-server's ServiceAccount has an IRSA role with `kms:Decrypt` permission.
- For age: the age private key is mounted as a Kubernetes Secret and referenced via env var.
- The decrypted manifest is passed to Kustomize/Helm and applied. Plaintext never persists in the ArgoCD pod.

### 4. What's the failure mode if a developer's age key is the only key in `.sops.yaml` and they leave the team?

**Reference answer:**
- If the key group only contains their age key, those files are unrecoverable without their private key.
- Best practice: always include a KMS key (or a shared team age key in 1Password) alongside individual keys in `.sops.yaml`.
- On offboarding: remove their key from `.sops.yaml`, run `sops updatekeys` on all encrypted files (this re-encrypts the DEK for the remaining recipients), and rotate any secrets they had access to decrypt.
- The `updatekeys` operation is the correct mitigation — no need to change the actual secret values unless the departing person was malicious.

---

## Further reading / watching

- **getsops/sops GitHub** (`github.com/getsops/sops`) — the authoritative README. Covers backends, key groups, creation rules, and the CLI in detail.
- **"SOPS: Secrets OPerationS" blog post by Mozilla** — the original motivation for building SOPS. Explains the DEK+KEK design decision.
- **viaduct-ai/kustomize-sops (KSOPS)** — the ArgoCD integration. The README has a full setup guide for ArgoCD + KSOPS + IRSA.
- **jkroepke/helm-secrets** — the Helm equivalent. If your GitOps uses Helm charts, this is the plugin.
- **FiloSottile/age** (`github.com/FiloSottile/age`) — the `age` encryption tool docs. The alternative to GPG — simpler, no keyring, modern crypto.

---

## The 4 dimensions (senior framing)

- **Tech**: DEK+KEK hybrid encryption; per-value encryption for diff-friendliness; age (simple, portable) vs KMS (IAM-governed, audited); `.sops.yaml` creation rules to enforce correct key per path; KSOPS/helm-secrets for ArgoCD sync-time decryption; key groups for M-of-N threshold decryption.
- **People**: developers need one workflow — `sops <file>` opens in editor, re-encrypts on save. Key distribution is the onboarding question: for age, share private key via 1Password; for KMS, grant IAM access. Offboarding: `sops updatekeys` after removing from `.sops.yaml`. Write a team runbook for both scenarios; don't assume people will figure it out.
- **CI/CD**: CI pipelines decrypt using KMS + IRSA (or a service account with KMS access). KSOPS plugin in ArgoCD handles sync. No secrets in the pipeline runner's environment variables — decryption happens in the operator, not the build system. New secrets follow the same PR review process as code changes (the encrypted file is reviewed, not the value).
- **Operations**: CloudTrail logs on every KMS Decrypt call — use this for audit trails ("who decrypted the DB password and when"). Alert on ArgoCD sync failures that include KSOPS errors (usually an IAM or KMS policy change). Quarterly: review `.sops.yaml` recipients; remove departed team members; test key recovery (can you decrypt files if the primary key is unavailable?).
