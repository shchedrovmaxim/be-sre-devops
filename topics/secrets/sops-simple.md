# SOPS + age/KMS — the simple version (the redacted document)

> Read this first. Once the analogy clicks, the [deep-dive doc](./sops.md) fills in the mechanics.

This doc explains **one idea**:

> **SOPS encrypts only the values in a YAML file, not the whole file. The keys stay readable. That makes encrypted files diff-able and reviewable — a huge win for GitOps.**

That's the whole concept. Everything else — age vs KMS backends, `.sops.yaml` creation rules, ArgoCD integration — is just detail on top.

---

## The redacted document analogy

Imagine a classified government document. Two ways to handle it:

**Option A — seal the whole document in an envelope.** No one can read anything. You can't diff it, you can't review it, you can't see what changed without decrypting it first.

**Option B — publish the document with values redacted (black bars).** Everyone can see the structure, the field names, what the document is about — but the sensitive values are hidden. When it changes, you can see *which fields* changed, even if you can't read the new values.

SOPS is Option B. It encrypts the *values*, leaves the *keys* in the clear.

A plain Kubernetes Secret:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  username: cG9zdGdyZXM=
  password: c3VwZXJzZWNyZXQ=
```

After SOPS encryption:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  username: ENC[AES256_GCM,data:abc123...,tag:xyz,type:str]
  password: ENC[AES256_GCM,data:def456...,tag:uvw,type:str]
sops:
  age:
    - recipient: age1...
  lastmodified: "2026-06-05T10:00:00Z"
  mac: "..."
```

The structure, field names, and metadata are all readable. Only the values are ciphertext. A Git diff on this file shows you *which* secrets changed — without revealing the new values.

---

## The 2-3 sentence summary

SOPS encrypts individual values within YAML (or JSON, ENV, INI) files while leaving keys readable. You choose a backend for the encryption key: `age` for simplicity (generate a keypair, done), or KMS (AWS/GCP/Azure) for team-scale governance. The encrypted file goes in Git; decryption requires access to the key backend, either locally (for developers with the key) or at deploy time (for ArgoCD via a plugin like KSOPS).

---

## The 2 concepts that trip people up

### 1. age vs KMS — same outcome, different scale

`age` is a simple modern encryption tool. You generate a keypair with `age-keygen`, keep the private key somewhere safe, and share the public key in your `.sops.yaml`. Anyone with the public key can encrypt; only whoever has the private key can decrypt. Simple, no cloud dependency.

KMS (AWS, GCP, Azure) uses a cloud-managed key. The "private key" never leaves the KMS service — even SOPS doesn't see it. KMS makes a `Decrypt` API call on your behalf. Who can decrypt is controlled by IAM/KMS key policies, giving you fine-grained access control and audit logs.

For a single developer: `age`. For a team with cloud infrastructure: KMS. For a team with really sensitive secrets and compliance needs: KMS + Vault.

### 2. The "who can decrypt" question is the governance question

SOPS puts the *encryption key management* problem front and center. With Sealed Secrets, the cluster manages the key. With ESO, the external store manages the key. With SOPS, **you** manage the key — and "you" has to mean something concrete: who has the age private key, which IAM roles can use the KMS key, what happens when someone leaves the team?

This is actually a feature, not a bug. It forces the conversation. But it's operationally more demanding than Sealed Secrets.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What does SOPS encrypt? | Only the **values** in a YAML file — not the keys, not the structure |
| What's the benefit of value-only encryption? | Diffs are readable; code review shows which secrets changed |
| What's `age`? | A simple CLI tool for generating encryption keypairs. No cloud required. |
| What's KMS? | A cloud-managed encryption key. Decryption happens via API call; the key never leaves the cloud. |
| Where does the encrypted file go? | Git — it's ciphertext and safe to store publicly |
| How does ArgoCD decrypt it? | Via a plugin: KSOPS (for Kustomize) or helm-secrets (for Helm). Runs at sync time. |
| Can I use SOPS across multiple clusters? | Yes — unlike Sealed Secrets, SOPS keys are not cluster-specific. One key can work everywhere. |
| What's `.sops.yaml`? | A config file in your repo root that tells SOPS which key to use for which file paths |

---

## Self-test (the killer interview question)

Out loud:

> **"Why is SOPS better than just encrypting whole files for Kubernetes manifests?"**

**Reference answer (intuitive version):**

"Encrypting the whole file turns the YAML into a blob. You can't diff it, you can't review it in a PR, and you can't tell what changed without decrypting. If two engineers are editing secrets in a shared repo, every change looks like a completely different blob — merge conflicts become unresolvable.

SOPS encrypts only the values, leaving the keys in the clear. So in a Git diff you can see 'the `password` field changed' even though you can't see what the new value is. Code review becomes possible: a reviewer can confirm the right fields are present, the naming convention is correct, the structure is right — without needing to decrypt the values.

There's also a multi-cluster advantage over Sealed Secrets: a SOPS-encrypted file with a KMS key works everywhere that has access to that KMS key. You don't re-seal per cluster. The same encrypted YAML works for dev and prod clusters (with different IAM permissions controlling who can actually decrypt).

The trade-off is that SOPS adds a plugin layer at deploy time (KSOPS for ArgoCD) and requires key management discipline that Sealed Secrets doesn't. For a small team using ArgoCD + Kustomize, SOPS with age is a great fit. For a larger team that needs audit trails and per-team key governance, KMS or Vault is the better backend."

---

## Further reading / watching

- **mozilla/sops GitHub** (`github.com/getsops/sops`) — the canonical source; the README covers all backends and the `.sops.yaml` creation rules.
- **jkroepke/helm-secrets** (`github.com/jkroepke/helm-secrets`) — the Helm plugin for SOPS. Used to decrypt Helm values files at deploy time.
- **viaduct-ai/kustomize-sops (KSOPS)** (`github.com/viaduct-ai/kustomize-sops`) — the Kustomize plugin. Used with ArgoCD for GitOps workflows.
- **"age" encryption tool** (`github.com/FiloSottile/age`) — the simplest SOPS backend for getting started.

---

## Next: the deep-dive

When the "value-only encryption" insight and the age-vs-KMS split feel obvious, jump to [`sops.md`](./sops.md). The deep-dive covers:

- The exact encryption mechanics: how SOPS uses hybrid encryption (DEK + KEK pattern)
- `.sops.yaml` creation rules — path_regex, key groups, require any/all
- Key groups and threshold encryption (M-of-N decryption for high-security scenarios)
- The ArgoCD + KSOPS integration flow (what happens at sync time)
- The local dev workflow: decrypting, editing, re-encrypting
- The "who can decrypt" governance framework for teams
- Failure modes: losing the age key, KMS key policy change, plugin misconfiguration
