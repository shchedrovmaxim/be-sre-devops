# Cosign / Sigstore (keyless OIDC) — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through keyless image signing with Cosign — what does 'keyless' actually mean, and what's the trust chain?"** — naming Fulcio, Rekor, the OIDC exchange, the verification flow, and how Kyverno enforces it at admission time.

> Start with [cosign-sigstore-simple.md](./simple.md) if you haven't. The wax-seal analogy frames everything here.

---

## The senior framing — supply-chain integrity at admission time

Mid-level: "we sign our images with Cosign."
Senior: "we enforce at admission time — Kyverno's `verifyImages` blocks any pod that tries to run an unsigned image from our registry, and the signing identity policy ensures only our CI pipeline (not a developer's laptop) can produce a valid signature."

The gap between signing and enforcement is where most teams fall short. Signing without enforcement is theater.

---

## Classic key-pair signing vs keyless

### Classic (key-pair) signing

```bash
# Generate a key pair (done once, guard private key)
cosign generate-key-pair
# Outputs cosign.key (private) + cosign.pub (public)

# Sign
cosign sign --key cosign.key $IMAGE_DIGEST

# Verify
cosign verify --key cosign.pub $IMAGE
```

Problems:
- The private key is a long-lived secret. You store it in a secret manager, audit access, rotate it periodically.
- Key rotation means updating every verifier policy.
- If a developer's machine has the key and they leave, you have a rotation event.
- Multi-cluster: every cluster needs the public key distributed to it.

### Keyless signing (Sigstore)

The private key exists for ~10 minutes, then is gone. The identity is proven by an OIDC token from your CI provider.

The full flow:

**Step 1 — CI job requests OIDC token**

GitHub Actions, GitLab CI, GCP Workload Identity, etc. issue OIDC tokens to their jobs. The token is a signed JWT containing:
- `iss` (issuer): `https://token.actions.githubusercontent.com`
- `sub` (subject): the job identity, e.g., `repo:myorg/myrepo:ref:refs/heads/main`
- `aud` (audience): set to Sigstore's Fulcio by Cosign

**Step 2 — Exchange with Fulcio**

Cosign sends the OIDC token to Fulcio (Sigstore's CA). Fulcio verifies the token signature (via the OIDC provider's JWKS endpoint) and issues a **short-lived X.509 certificate** (~10 minute lifetime) containing:
- The subject SAN (Subject Alternative Name) from the OIDC token: `https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main`
- The OIDC issuer embedded as a certificate extension

**Step 3 — Sign the image digest**

Cosign generates a temporary key pair, signs the image digest with the private key, and includes the Fulcio cert as the verification chain. The private key is discarded after signing.

**Step 4 — Publish to Rekor**

The signature bundle (signature + cert + image digest) is published to Rekor, the public transparency log. Rekor gives back an entry ID + inclusion proof (a signed timestamp called an SCT). This is stored in the OCI registry alongside the image as a Cosign signature artifact.

**Step 5 — Private key discarded**

The ephemeral private key no longer exists. The signature in Rekor is permanent. The Fulcio cert is embedded in the signature bundle.

---

## Verification flow

```bash
# Keyless verify — checks Rekor + validates cert chain
cosign verify \
  --certificate-identity-regexp="^https://github.com/myorg/myrepo/" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  $IMAGE
```

What Cosign does:
1. Fetches the Cosign signature artifact from the OCI registry (stored at `registry/image:sha256-<digest>.sig`).
2. Extracts the Fulcio cert from the signature bundle.
3. Verifies the cert chain: cert → Fulcio intermediate → Sigstore root CA.
4. Checks the cert's SAN against `--certificate-identity-regexp`.
5. Checks the cert's OIDC issuer extension against `--certificate-oidc-issuer`.
6. Verifies the signature against the image digest using the public key in the cert.
7. Verifies the Rekor inclusion proof (the signature was logged before the cert expired).

Steps 6 and 7 are both required: the signature must be cryptographically valid AND the cert must have been valid when it was signed (proven by the Rekor timestamp).

---

## Kubernetes admission integration — Kyverno verifyImages

You have Kyverno in production, so this is your home terrain.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["production", "staging"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/myorg/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

What this enforces:
- Any pod in `production` or `staging` using an image from `registry.example.com/myorg/*` must have a valid Cosign signature.
- The signature must have been created by the specific GitHub Actions workflow (`build.yaml`) in the specific repo.
- Verification goes through Rekor.
- Unsigned images or images signed by anything else → admission denied.

### SBOM attestation requirement (bonus)

```yaml
      verifyImages:
        - imageReferences:
            - "registry.example.com/myorg/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
          attestations:
            - type: https://cyclonedx.org/bom
              attestors:
                - entries:
                    - keyless:
                        subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
                        issuer: "https://token.actions.githubusercontent.com"
```

Now you require both: a Cosign image signature AND a Cosign-attested CycloneDX SBOM. Both must be from the trusted workflow.

---

## Sigstore Policy Controller — the alternative to Kyverno

If you're not using Kyverno, Sigstore ships its own K8s admission controller:

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: image-policy
spec:
  images:
    - glob: "registry.example.com/myorg/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "^https://github.com/myorg/.*"
```

Sigstore Policy Controller installs as a validating admission webhook. It's lighter than Kyverno (purpose-built for signing), but Kyverno's `verifyImages` integrates with your existing policy surface. For most teams already running Kyverno, `verifyImages` is the answer. Policy Controller is for teams who want signing enforcement without adopting a full policy engine.

---

## CI signing pipeline (complete)

```yaml
# .github/workflows/build.yaml
name: Build and Sign
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write   # REQUIRED: allows the job to request an OIDC token
  packages: write

jobs:
  build-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Build and push image
        id: build
        run: |
          docker build -t $IMAGE .
          docker push $IMAGE
          # Capture the digest for signing (signing by digest, not tag, is critical)
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' $IMAGE)
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT

      - name: Sign image (keyless)
        run: |
          cosign sign --yes ${{ steps.build.outputs.digest }}
        # --yes skips the interactive prompt
        # The job's OIDC token is used automatically by cosign

      - name: Generate and attest SBOM
        run: |
          syft ${{ steps.build.outputs.digest }} -o cyclonedx-json > sbom.cdx.json
          cosign attest --yes \
            --predicate sbom.cdx.json \
            --type cyclonedx \
            ${{ steps.build.outputs.digest }}
```

The `permissions: id-token: write` is required — without it, GitHub Actions won't issue an OIDC token to the job.

Signing the **digest** (not the tag) is critical. Tags are mutable; a `latest` tag can point to a different image tomorrow. The digest is immutable. Cosign's signature is tied to the digest, so it survives tag changes.

---

## The "no long-lived keys to rotate" senior win — fully argued

1. **No key rotation ceremony**: every long-lived key needs a rotation schedule, a notification when it expires, a plan for rotating it across all consumers. With keyless, there's nothing to rotate.

2. **No key compromise risk**: the ephemeral key exists for 10 minutes. Even if an attacker got it (extremely unlikely from a CI runner), it's useless before they could sign anything meaningful.

3. **Self-describing identity**: the signing cert embeds the exact identity — `repo:myorg/myrepo:ref:refs/heads/main`. The Kyverno policy checks this identity. You know *exactly* who can produce a valid signature without managing any key distribution.

4. **No "ex-employee has the key" risk**: the key was a CI job's OIDC identity. Former employees don't retain that.

5. **Public auditability**: Rekor is a public log. Anyone can verify that image X was signed by pipeline Y at time Z. This is the supply chain transparency story — open, auditable, no trust-me-we-signed-it assertions.

---

## The gotcha — signing by tag vs digest

Common mistake:

```bash
# BAD: signs the tag, not the digest
cosign sign registry.example.com/myapp:v1.2.3

# GOOD: sign by digest
cosign sign registry.example.com/myapp@sha256:abc123...
```

When you sign a tag, the signature is tied to whatever digest the tag points to at signing time. If the tag is later pushed to point to a different image (intentional or via a supply-chain attack), your signature still appears valid — because the signature lookup is by tag, not digest. Always sign by digest.

In practice: build → push → capture the digest from the registry → sign the digest. The example above shows this with `docker inspect --format='{{index .RepoDigests 0}}'`.

---

## The interview answer in 60 seconds

> "Keyless means there's no private key you manage. The CI job gets an OIDC token from GitHub Actions — a short-lived JWT proving 'this is job X in repo Y.' Cosign exchanges that with Fulcio, the Sigstore CA, which issues a short-lived signing cert embedding the job's identity. The image digest is signed with an ephemeral private key, and the signature bundle is recorded in Rekor, the public transparency log. The private key is then discarded.
>
> Verification is: did this image get signed by a cert from a trusted OIDC identity? Cosign checks Rekor for the signature, validates the cert chain back to Fulcio, checks the identity in the cert — 'github.com/myorg/myrepo, GitHub Actions issuer' — and verifies the cryptographic signature.
>
> In Kubernetes, I enforce this with Kyverno's `verifyImages`. Any pod using images from our registry must have a valid keyless Cosign signature from our CI pipeline. Unsigned images are denied at admission. The senior win is: no long-lived key to rotate, compromise, or distribute. The CI job's OIDC identity is self-describing and self-expiring."

---

## Self-test drills

### 1. What does "keyless" actually mean in Cosign, and what's the trust chain?

**Reference answer:**
- No long-lived private key. The CI job authenticates via an OIDC token from GitHub/GitLab/GCP.
- Cosign exchanges the OIDC token with Fulcio for a short-lived (~10 min) signing cert.
- An ephemeral key pair is generated, the image digest is signed, the cert + signature is recorded in Rekor.
- The ephemeral private key is discarded. The Rekor entry is permanent.
- Verification: check Rekor for the signature → validate cert chain to Fulcio root → check OIDC identity in cert → verify cryptographic signature.
- Trust anchor: OIDC issuer (GitHub Actions, etc.). No key secret to manage.

### 2. Why must you sign by digest, not by tag?

**Reference answer:**
- Tags are mutable. `myapp:v1.2.3` can be overwritten to point to a different image at any time.
- If you sign by tag and the tag is later updated (supply chain attack or accident), the Cosign signature still "verifies" — because the signature is stored under the tag, and lookups are by tag.
- Signing by digest ties the signature to the immutable image content. If the content changes, the digest changes, and the old signature no longer matches.
- Practical: capture digest after `docker push` with `docker inspect --format='{{index .RepoDigests 0}}'`, then pass the `registry/image@sha256:...` form to `cosign sign`.

### 3. How does Kyverno's verifyImages enforce signing at admission time?

**Reference answer:**
- `ClusterPolicy` with a `verifyImages` rule matches pods by `imageReferences` glob.
- The `attestors` block specifies the keyless identity: issuer + subject (or regexp).
- When a pod is admitted, Kyverno's webhook fetches the Cosign signature artifact from the registry, runs the same verification flow (cert chain, Rekor, identity check), and either allows or denies the pod.
- `validationFailureAction: Enforce` means denial. `Audit` means allow but log.
- You can additionally require SBOM attestations in the same `verifyImages` block — the pod is denied if the SBOM attestation is missing or not from the trusted identity.
- The Wiz integration: Kyverno enforces "only signed images"; Wiz enforces "only images that passed our posture scan." They're complementary gates.

---

## Further reading

- [Sigstore docs](https://docs.sigstore.dev/) — the canonical reference; keyless signing walkthrough
- [Cosign GitHub](https://github.com/sigstore/cosign) — CLI reference
- [Fulcio](https://github.com/sigstore/fulcio) — the CA; understand what certs it issues
- [Rekor](https://github.com/sigstore/rekor) — query it directly: `rekor-cli get --log-index N`
- [Kyverno verifyImages docs](https://kyverno.io/docs/writing-policies/verify-images/) — the admission integration
- [Sigstore Policy Controller](https://github.com/sigstore/policy-controller) — the alternative to Kyverno for signing enforcement

---

## The 4 dimensions (senior framing)

- **Tech**: keyless signing via OIDC → Fulcio cert → ephemeral key → Rekor transparency log. Sign by digest, not tag. Verification: `cosign verify --certificate-identity-regexp --certificate-oidc-issuer`. SBOM attestation via `cosign attest`. Kyverno `verifyImages` for admission enforcement.
- **People**: introduce signing incrementally. Start in `Audit` mode with Kyverno — log unsigned images but don't block. After 2 weeks you know the full blast radius. Flip to `Enforce` with a known-exceptions list. Don't flip Enforce on a Monday.
- **CI/CD**: `permissions: id-token: write` in GitHub Actions is easy to forget and gives a confusing error. Add it to the platform team's workflow template. Sign in the build job (not a separate job) so the digest is immediately available. Cache Cosign binary installation.
- **Operations**: Rekor is a public log — that's usually fine (it only records signatures, not image contents). If you're in an air-gapped environment, run your own Rekor + Fulcio stack (Sigstore supports self-hosted). Monitor for "image unsigned" Kyverno audit events as a security signal — any such event in production warrants investigation.
