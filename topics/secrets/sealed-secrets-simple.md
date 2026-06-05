# Sealed Secrets — the simple version (the wax-sealed envelope)

> Read this first. Once the analogy clicks, the [deep-dive doc](./sealed-secrets.md) fills in the mechanics.

This doc explains **one idea**:

> **A SealedSecret is an envelope you can only open inside the specific cluster it was sealed for. Anyone can seal it; only that cluster can open it.**

That's the whole concept. Everything else — `kubeseal`, the controller, key rotation, the multi-cluster problem — is just detail on top.

---

## The wax-sealed envelope

Imagine the old practice of sealing a letter in wax. The sender presses a specific signet ring into the wax. Only the person holding the matching ring can verify it — but critically, *anyone can seal a new envelope* by pressing the same wax stamp on their side.

Sealed Secrets works the same way:

| In the physical world | In the Sealed Secrets world |
|---|---|
| The wax stamp (publicly available) | The **cluster's public key** (fetchable with `kubeseal --fetch-cert`) |
| You press wax to seal the envelope | `kubeseal` encrypts your secret against that public key |
| The sealed envelope | The `SealedSecret` YAML — safe to commit to Git |
| The signet ring (private, only you hold it) | The **controller's private key** — lives only in the cluster |
| Breaking the wax seal to read the letter | The controller decrypts and creates a real `Secret` |

You check the sealed envelope (the `SealedSecret` YAML) into Git. It's ciphertext — no one can read it without the cluster's private key. The controller in the cluster holds the private key, opens the envelope, and creates a plain Kubernetes `Secret` from the decrypted contents.

---

## The 2-3 sentence summary

Sealed Secrets uses asymmetric encryption: you encrypt with the cluster's **public key** (available to everyone) and only the in-cluster controller can decrypt with the **private key** (never leaves the cluster). The resulting `SealedSecret` YAML is safe to store in Git — it's just ciphertext. The controller watches for `SealedSecret` resources and automatically creates the corresponding `Secret` in the right namespace.

---

## The 2 concepts that trip people up

### 1. Scope binding (namespace + name)

A `SealedSecret` is not generically encrypted. It's encrypted specifically for `name=my-secret, namespace=production`. If you try to rename it or move it to a different namespace, the controller **rejects it**.

This is intentional security hardening — a leaked ciphertext can't be replayed in another namespace to gain access there. But it's a gotcha: you can't just `cp` a SealedSecret between namespaces.

### 2. The single point of failure: the controller private key

The private key that lives in the cluster is a `Secret` called `sealed-secrets-key` in the `kube-system` namespace (by default). If that key is lost — cluster deleted, DR scenario, key rotation gone wrong — **you cannot decrypt any of your existing SealedSecrets**. Every secret has to be re-sealed.

This is the thing seniors notice immediately: the keys need to be backed up out-of-band, separately from the cluster, before anything goes wrong.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What encrypts a SealedSecret? | `kubeseal` CLI, using the cluster's **public** key |
| What decrypts it? | The Sealed Secrets controller, using its **private** key (never leaves the cluster) |
| Is the YAML safe in Git? | Yes — it's pure ciphertext |
| Can I use one SealedSecret in two namespaces? | No — it's scoped to a specific name + namespace at seal time |
| What's the single point of failure? | The controller's private key — if lost, every secret must be re-sealed |
| What's the multi-cluster problem? | You must seal separately per cluster (each has its own keypair). One SealedSecret ≠ works everywhere. |
| When is it a bad fit? | Multiple clusters, large number of secrets, or when you need rotation without re-sealing |

---

## Self-test (the killer interview question)

Out loud:

> **"Why might Sealed Secrets be a bad fit for a multi-cluster GitOps setup?"**

**Reference answer (intuitive version):**

"Because each cluster generates its own unique keypair. A `SealedSecret` encrypted for cluster A cannot be decrypted by cluster B — they have different private keys. So if you have three clusters, you need to seal every secret three separate times and store three different `SealedSecret` YAMLs for the same logical secret.

In a single-cluster world that's fine. In a multi-cluster GitOps world where you want one Git repo to be the source of truth across environments, this gets painful fast: every secret rotation means running `kubeseal` three times, opening three PRs, and hoping they all get merged before the old secret rotates.

There's also the backup problem: each cluster's private key has to be backed up separately. Lose one backup, lose the ability to recover that cluster's secrets.

The alternative that solves this properly is ESO (External Secrets Operator) — one central secrets store (AWS Secrets Manager, Vault), ESO sync agents in each cluster. The source of truth is the external store, not a cluster-specific encrypted blob."

---

## Further reading / watching

- **Bitnami Sealed Secrets GitHub** — `bitnami-labs/sealed-secrets`. The README covers the `kubeseal` workflow and key management. Look for the "Managing existing secrets" and "Key rotation" sections.
- **"Sealed Secrets with ArgoCD" by the ArgoCD docs team** — search "ArgoCD Sealed Secrets" on the ArgoCD docs site. Short integration guide.
- **"GitOps Secrets Management" — Salaboy (Mauricio Salatino)** — various blog posts on the topic; covers all three approaches (Sealed Secrets, ESO, SOPS) with opinions on when to use each.

---

## Next: the deep-dive

When the wax-seal analogy and the "scoped-per-cluster" limitation feel obvious, jump to [`sealed-secrets.md`](./sealed-secrets.md). The deep-dive covers:

- The full encryption mechanics (`kubeseal` → controller → `Secret`) with key sizes and algorithm details
- Scope modes: strict, namespace-wide, cluster-wide — and when you'd use each
- Key rotation: what happens, the overlap window, why you shouldn't wait too long
- Backup and DR: exact commands to back up and restore the controller key
- The multi-cluster dealbreaker: why this disqualifies Sealed Secrets from most GitOps setups at scale
- The senior comparison: Sealed Secrets vs ESO vs SOPS across the dimensions that matter
