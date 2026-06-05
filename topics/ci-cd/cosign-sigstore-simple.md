# Cosign / Sigstore — the simple version (the wax seal)

> Read this first. Once the analogy clicks, the [deep-dive](./cosign-sigstore.md) is easy.

This doc explains **one idea**:

> **Cosign lets you cryptographically prove "this image came from this trusted pipeline and hasn't been tampered with" — without managing long-lived signing keys.**

---

## The wax seal on a letter

Medieval documents were sealed with a wax stamp. The seal did two things:

1. **Proved identity** — only the person with the signet ring could make that exact seal.
2. **Proved integrity** — if anyone opened the letter and resealed it, the seal would look different.

Image signing is the same idea. The "wax seal" on a container image proves:
- Who built it (identity: the CI pipeline, the specific GitHub Actions job, the specific commit)
- That it hasn't been tampered with (integrity: the image contents match the signed digest)

Cosign is the tool that applies and verifies these seals for container images.

---

## The "keyless" part — why it's the modern approach

Old-school signing: you generate a key pair, guard the private key like it's gold, rotate it periodically, store it in a secret manager, audit who has access, freak out when someone leaves the team.

Keyless signing (Sigstore): there is no long-lived private key. Instead:

1. Your CI job has an **OIDC token** — a short-lived proof of identity issued by GitHub Actions / GitLab CI / GCP / etc.
2. You exchange that token with **Fulcio** (Sigstore's certificate authority) for a **short-lived signing certificate** (valid for ~10 minutes).
3. You sign the image with that cert.
4. The signature + cert is recorded in **Rekor**, a public transparency log (like a blockchain for software signatures).
5. The cert expires. The signature in Rekor is permanent.

Later, when you verify: "was this image signed by a GitHub Actions job from my org's repo?" — you check Rekor. No private key to lose, rotate, or audit. The OIDC identity is the trust anchor.

---

## The trust chain in one picture

```
GitHub OIDC token (proves "this is job X in repo Y")
        ↓
Fulcio (Sigstore CA) issues short-lived cert
        ↓
cosign signs image digest with cert
        ↓
Rekor records {image digest + cert + signature}
        ↓
Verification: "image digest in Rekor, signed by github.com/myorg/myrepo?" → YES ✓
```

No long-lived key anywhere in this chain. The OIDC token is the identity.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does "signing" an image mean? | Cryptographically binding "this identity built this exact image digest." |
| What is keyless? | No long-lived key pair. Identity comes from an OIDC token (from GitHub/GitLab/GCP). |
| What is Fulcio? | The Sigstore CA. Exchanges your OIDC token for a short-lived signing cert. |
| What is Rekor? | The public transparency log. Records every signature permanently. Anyone can verify. |
| What is Cosign? | The CLI tool that talks to Fulcio + Rekor and writes signatures to the OCI registry. |
| How does K8s enforce this? | Kyverno's `verifyImages` rule checks the Cosign signature before allowing a pod to start. |
| Why is "no long-lived key" a win? | Nothing to rotate, nothing to leak, nothing to audit access to. The CI job's OIDC identity is self-describing. |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through keyless image signing with Cosign — what does 'keyless' actually mean, and what's the trust chain?"**

**Reference answer (intuitive version):**

"Keyless means there's no private key you manage. Instead, the CI job authenticates with an OIDC token — GitHub Actions issues one automatically for every job. Cosign exchanges that token with Fulcio, the Sigstore certificate authority, which issues a short-lived signing cert that embeds the job's identity (the repo, the workflow, the commit). Cosign signs the image digest with that cert. The signature plus cert is recorded in Rekor, the public transparency log. The cert expires in minutes. The entry in Rekor is permanent.

When someone wants to verify: 'was this image built by a trusted pipeline?' they run `cosign verify` with a policy — 'I trust images signed by GitHub Actions jobs from github.com/myorg/myrepo.' Cosign checks Rekor, finds the signature, verifies the cert chain back to Fulcio, and checks the identity in the cert. No private key to manage anywhere."

---

## Further reading

- [Sigstore docs](https://docs.sigstore.dev/) — the canonical reference
- [Cosign GitHub](https://github.com/sigstore/cosign) — the CLI tool
- [Fulcio](https://github.com/sigstore/fulcio) — the Sigstore CA
- [Rekor](https://github.com/sigstore/rekor) — the transparency log

---

## Next: the deep-dive

When the wax-seal + keyless trust chain feels obvious, jump to [`cosign-sigstore.md`](./cosign-sigstore.md). The deep-dive covers:

- Classic key-pair signing vs keyless in detail
- The OIDC issuer trust policy (what you actually configure)
- Verification flow with exact `cosign verify` commands
- Kyverno `verifyImages` admission integration
- Sigstore Policy Controller as an alternative
- The "no long-lived keys to rotate" senior win fully argued
- 3 self-test drills + 4-dimensions framing
