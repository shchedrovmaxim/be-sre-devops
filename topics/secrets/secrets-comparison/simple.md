# Secrets management — which one to pick (the simple version)

> Read this first. Once the mental model clicks, the [deep-dive doc](./deep-dive.md) has the full trade-off matrix.

This doc explains **one idea**:

> **There's one axis that determines most of the trade-off: are secrets in Git (encrypted), or does Git only contain a pointer to where secrets live?**

Everything else — team size, operational maturity, audit requirements — is secondary to understanding this axis first.

---

## The one axis that matters: in-Git vs runtime-pull

Think of it like two different approaches to a safe:

**Option A — the portable safe (in-Git)**: you carry an encrypted container everywhere. It travels with your code. Anyone can see the container; only someone with the key can open it. Sealed Secrets and SOPS are this.

**Option B — the bank safety deposit box (runtime-pull)**: you have a card (an `ExternalSecret` manifest) that says "my valuables are at box 423 at First National." Anyone can read the card — it says nothing sensitive. But only you (ESO with the right auth) can access the box. ESO + AWS Secrets Manager and Vault + ESO are this.

| Tool | Axis | What lives in Git |
|---|---|---|
| Sealed Secrets | In-Git | Encrypted `SealedSecret` YAML |
| SOPS | In-Git | Encrypted YAML (values only) |
| ESO + AWS Secrets Manager | Runtime-pull | `ExternalSecret` YAML (no values) |
| Vault + ESO | Runtime-pull | `ExternalSecret` YAML (no values) |

Neither axis is universally better. The choice depends on your constraints.

---

## The 2-3 sentence summary

Sealed Secrets and SOPS encrypt the secret value and store the ciphertext in Git — the cluster or developer's key is needed to decrypt. ESO (with any backend) stores only a reference in Git and fetches the actual value from an external store at runtime. For single-cluster simple setups, the in-Git tools are easier; for multi-cluster, rotation-heavy, or compliance-grade environments, runtime-pull is the right answer.

---

## The 2 most confusing trade-offs

### 1. Multi-cluster is the dealbreaker for Sealed Secrets

Sealed Secrets generates a unique keypair per cluster. A secret sealed for cluster-A is permanently unreadable by cluster-B. In a three-cluster setup you maintain three separate sealed copies of every secret.

SOPS doesn't have this problem — a KMS key or age key is not cluster-specific. You seal once and every cluster that has access to the decryption key can use it.

ESO doesn't have this problem either — the `ExternalSecret` YAML is identical across clusters; each cluster's ESO fetches independently from the same store.

**If you have more than one cluster: Sealed Secrets is the wrong choice.**

### 2. Rotation works differently in each tool

- **Sealed Secrets**: rotation means re-sealing (run `kubeseal` again, commit the new YAML, let ArgoCD sync).
- **SOPS**: rotation means updating the decrypted value, re-encrypting with SOPS, committing the new YAML.
- **ESO + Secrets Manager**: rotation happens in Secrets Manager; ESO picks it up on the next `refreshInterval`. No Git commit needed. This is the huge operational win.
- **Vault + ESO**: same as ESO + Secrets Manager, plus dynamic secrets where Vault rotates credentials automatically.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Single cluster, small team, simple GitOps | **Sealed Secrets** or **SOPS + age** |
| Multi-cluster GitOps | **ESO** (with Secrets Manager or Vault). Sealed Secrets is disqualified. |
| Want zero values in Git at all | **ESO** (only pointer manifests in Git) |
| Need audit trail of every secret access | **Vault + ESO** |
| Need dynamic short-lived DB credentials | **Vault + ESO** — nothing else does this |
| Team has AWS, wants managed service | **ESO + AWS Secrets Manager** |
| On-prem or multi-cloud | **Vault + ESO** |
| Want diff-able encrypted YAML | **SOPS** |
| Minimal operational overhead | **Sealed Secrets** (single cluster) or **ESO + Secrets Manager** |
| Compliance-grade audit requirement | **Vault + ESO** |

---

## Self-test (the killer interview question)

Out loud:

> **"Your team needs to ship secrets via GitOps. Walk through the trade-offs of Sealed Secrets vs ESO."**

**Reference answer (intuitive version):**

"The core difference is where the actual secret lives. With Sealed Secrets, the encrypted value is in Git — the cluster's private key decrypts it at sync time. With ESO, the value is never in Git; only a pointer `ExternalSecret` manifest lives in Git, and ESO fetches the real value from an external store like AWS Secrets Manager.

For a single-cluster setup, Sealed Secrets is simpler: no external store to manage, no controller polling an API, just a `kubeseal` command and a commit. The limitation is the per-cluster keypair — sealed once for this cluster, can't be read by any other cluster. So if you have three environments on three clusters, you're sealing three times per rotation.

ESO is the right answer for multi-cluster. One source of truth in Secrets Manager, each cluster's ESO fetches independently. Rotation means updating Secrets Manager once, and all clusters pick up the new value on the next refresh interval — no PRs needed for rotation.

The Sealed Secrets gotcha I'd flag specifically is the private-key backup problem: the controller's private key is the single point of failure. If the cluster dies and you haven't backed up the key out-of-band, every sealed secret in Git is permanently unrecoverable.

I'd complement either approach with a Kyverno policy that denies any `Secret` resource in production that wasn't created by a known controller — ESO or Sealed Secrets — so no one can bypass the system by manually `kubectl apply`-ing a plain Secret."

---

## Further reading / watching

The best way to internalize the trade-offs is to read each tool's dedicated doc in this folder:

- [`sealed-secrets-simple.md`](../sealed-secrets/simple.md) — wax-seal analogy
- [`external-secrets-operator-simple.md`](../external-secrets-operator/simple.md) — personal shopper analogy
- [`sops-simple.md`](../sops/simple.md) — redacted document analogy
- [`vault-eso-pattern-simple.md`](../vault-eso-pattern/simple.md) — bank vault analogy

For the comparison from a team that has evaluated all options: search "GitOps secrets management comparison 2024/2025" — several good blog posts from Natan Yellin, Banzai Cloud, and the ESO maintainers compare the tools with concrete examples.

---

## Next: the deep-dive

When the in-Git vs runtime-pull axis is clear and you can explain the multi-cluster dealbreaker for Sealed Secrets, jump to [`secrets-comparison.md`](./deep-dive.md). The deep-dive covers:

- The full trade-off matrix across 8 dimensions
- Where Kyverno fits as a complementary control (enforcing "no plain Secrets in manifests")
- The decision tree for picking a tool based on your constraints
- The audit requirement question — when Vault is the only answer
- How your Wiz experience informs the "defense in depth" framing
