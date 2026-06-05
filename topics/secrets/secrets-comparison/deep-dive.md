# Secrets management — when to pick each (the deep-dive)

> **Goal**: by the end you can answer — **"Your team needs to ship secrets via GitOps. Walk through the trade-offs of Sealed Secrets vs ESO."** — and extend that answer to cover SOPS and Vault, with Kyverno as a complementary enforcement layer.

> Start with the [simple version](./simple.md) if you haven't. The in-Git vs runtime-pull axis is the spine of the comparison.

---

## The senior framing — pick by trade-off, not hype

Mid-level: "We use ESO because it's better than Sealed Secrets."
Senior: "We use ESO because we have three clusters and rotation needs to happen without PRs. For a team with one cluster and strong GitOps discipline, Sealed Secrets would have been simpler."

The interview signal isn't knowing the right answer — it's knowing *why* the answer changes based on context. The decision framework below is what you verbalize in the interview.

---

## The full trade-off matrix

Across eight dimensions that actually matter:

| Dimension | Sealed Secrets | SOPS + age/KMS | ESO + Secrets Manager | Vault + ESO |
|---|---|---|---|---|
| **What's in Git** | Encrypted SealedSecret YAML | Encrypted YAML (values only) | ExternalSecret YAML (no values) | ExternalSecret YAML (no values) |
| **Multi-cluster** | Bad — unique keypair per cluster, must re-seal per cluster | OK — key is not cluster-specific | Good — one store, all clusters pull | Good — one Vault, all clusters pull |
| **Secret rotation** | Manual re-seal + PR | Manual re-encrypt + PR | Automatic (on refresh interval) | Automatic (+ dynamic creds) |
| **Audit trail** | None — no log of who decrypted what | KMS CloudTrail (if using KMS backend) | CloudTrail (Secrets Manager API calls) | Vault audit log (complete, queryable) |
| **Dynamic credentials** | No | No | No | Yes — DB engine, PKI, etc. |
| **Operational overhead** | Low — one controller in cluster | Low — no controller; plugin at apply time | Medium — ESO controller + external store | High — Vault HA + unsealing + policies |
| **External dependency at runtime** | None (cluster is self-contained) | None for sealed files; KMS at decrypt time | Secrets Manager API must be reachable | Vault must be reachable and unsealed |
| **Code review on secrets** | No — opaque ciphertext in Git | Yes — keys readable, only values encrypted | Yes — ExternalSecret YAML shows structure | Yes — ExternalSecret YAML shows structure |

---

## The decision tree

Walk this in order:

### 1. Do you have more than one cluster?

**Yes** → Sealed Secrets is disqualified (per-cluster keypair, must re-seal per cluster). Go to step 2.
**No** → Sealed Secrets is a valid option. Also consider SOPS if you want diff-able code review.

### 2. Do you need dynamic credentials (short-lived DB users, etc.)?

**Yes** → Vault + ESO is the answer. Nothing else provides this natively.
**No** → Go to step 3.

### 3. Do you have compliance/audit requirements that require a queryable access log?

**Yes** → Vault + ESO (Vault audit log) or ESO + Secrets Manager (CloudTrail). Vault is stronger if auditors require structured per-request logs.
**No** → Go to step 4.

### 4. Are you already on AWS with a managed service preference?

**Yes** → ESO + AWS Secrets Manager. Fully managed, IRSA auth, rotations via Lambda rotation functions, no Vault ops overhead.
**No** → Go to step 5.

### 5. Are you on-prem, multi-cloud, or self-hosted?

**Yes** → SOPS + KMS (if you want in-Git encrypted) or Vault + ESO (if you want a central store).
**No (small team, single cloud, no compliance requirements)** → SOPS + age is the simplest in-Git option with no runtime dependency.

---

## The Sealed Secrets vs ESO deep comparison

This is the specific question. Walk it as a structured argument:

### What's in Git

- **Sealed Secrets**: the `SealedSecret` YAML contains actual encrypted ciphertext. Anyone can look at it but can't read the values. Safe to store publicly.
- **ESO**: the `ExternalSecret` YAML contains only a reference — "fetch key `my-app/db/password` from store `aws-secrets-manager-prod`." The secret value is nowhere in Git, ever. For a strict "no sensitive material in Git" posture, ESO wins here.

### Multi-cluster (the dealbreaker)

Already covered: Sealed Secrets has a unique keypair per cluster. ESO is multi-cluster native.

In numbers: 50 microservices × 30 secrets each = 1,500 secrets. With 3 clusters, Sealed Secrets means 4,500 sealed YAML files (1,500 × 3) in Git, all needing to stay in sync. With ESO, you have 1,500 `ExternalSecret` YAMLs that work across all three clusters. The operational difference compounds with scale.

### Rotation

- **Sealed Secrets**: someone must run `kubeseal` with the new value, commit the new YAML, open a PR, get it merged, wait for ArgoCD sync. Every secret rotation is a code change. For 50 secrets, 50 PRs.
- **ESO**: update the value in Secrets Manager. Every cluster picks it up on the next `refreshInterval`. No PRs. No Git changes. Zero-friction rotation.

For a team with mandatory rotation policies (e.g., "all DB passwords rotate every 90 days"), ESO's rotation model is the difference between a sustainable process and a compliance nightmare.

### External dependency

- **Sealed Secrets**: the cluster is self-contained. Even if AWS goes down, existing `SealedSecret` resources continue to decrypt (the key is in the cluster). No runtime API calls to external services.
- **ESO**: requires the external store to be reachable. If AWS Secrets Manager has an outage (rare but it happens), ESO can't refresh secrets. Existing `Secrets` remain (last-good value) but can't be updated.

For air-gapped environments or environments with connectivity constraints, Sealed Secrets (or SOPS) is more resilient.

### The single-point-of-failure question

- **Sealed Secrets**: the controller's private key. If lost, every sealed secret is permanently unrecoverable. Requires out-of-band backup discipline.
- **ESO**: the external store. If AWS Secrets Manager is inaccessible, ESO stops updating. But the actual secret values are in a fully managed, multi-AZ service with their own backup — much less likely to result in data loss.

---

## Kyverno as a complementary control

This is the senior layer that separates a good answer from a great one.

The problem: any of the above tools can be bypassed. An engineer can `kubectl apply` a plain `Secret` manifest directly into production, completely bypassing ESO, Sealed Secrets, or SOPS. The GitOps tool won't know; it didn't create the resource.

Kyverno `ClusterPolicy` closes this gap:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-externalsecret-owner
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-secret-owner
      match:
        any:
          - resources:
              kinds:
                - Secret
              namespaceSelector:
                matchLabels:
                  environment: production   # only enforce in prod namespaces
      validate:
        message: "Secrets in production must be created by ExternalSecret or SealedSecret controllers. Direct Secret creation is not permitted."
        deny:
          conditions:
            any:
              - key: "{{ request.object.metadata.ownerReferences[].kind | contains(@, 'ExternalSecret') || request.object.metadata.ownerReferences[].kind | contains(@, 'SealedSecret') }}"
                operator: Equals
                value: false
```

What this enforces: any `Secret` created in a production namespace must have an `ownerReference` from an `ExternalSecret` or `SealedSecret`. Direct `kubectl apply` of a plain `Secret` is rejected at the admission level.

This is exactly the kind of defense-in-depth a senior with Wiz + Kyverno experience would propose. Wiz might surface "plain Secret in production without ownerReference" as a finding; the Kyverno policy prevents that finding from ever occurring.

---

## The Wiz + Kyverno production experience bridge

If you've used Wiz in production, you've seen findings for:
- Secrets in environment variables (container manifests)
- Overly permissive RBAC on `secrets` resources
- Unencrypted secrets in Git (if Wiz scans your repos)

The ESO/Sealed Secrets/SOPS story connects to each of these:
- **Wiz finding: secrets in env vars** → resolved by ESO syncing to Secrets Manager + app reading from K8s Secret with `valueFrom.secretKeyRef` instead of literal values
- **Wiz finding: broad RBAC on secrets** → ESO controller ServiceAccount needs read access; audit who else has `get`/`list` on `secrets` in production namespaces
- **Wiz finding: plaintext in Git** → SOPS or Sealed Secrets addresses the Git surface; ESO eliminates it entirely

The Kyverno enforcement policy is the answer to "how do you prevent a regression on any of these findings."

---

## SOPS in the comparison — where it fits

SOPS is often overlooked in the Sealed Secrets vs ESO debate, but it has a distinct niche:

**SOPS wins when**:
- You want diff-able, code-reviewable secrets (per-value encryption)
- You want in-Git encryption without a per-cluster keypair (unlike Sealed Secrets)
- You want no runtime store dependency (unlike ESO)
- Your team already uses Helm or Kustomize and wants a minimal-footprint solution

**SOPS loses when**:
- Your team doesn't want to manage key distribution (who has the age private key, who doesn't)
- Rotation happens frequently (every rotation is a re-encrypt + PR)
- You need dynamic credentials

SOPS + KMS is a strong choice for teams using ArgoCD with Kustomize, where each team manages its own secrets and the KMS key governance is already covered by IAM. The `helm-secrets` or KSOPS plugin at ArgoCD sync time is the integration point.

---

## The interview answer in 60 seconds

> "The core trade-off is: Sealed Secrets and SOPS put encrypted values in Git; ESO puts only a pointer in Git and fetches from an external store at runtime.
>
> For a single-cluster setup with a small team, Sealed Secrets is operationally the simplest — one controller, one keypair, no external store. The limitation is the keypair: unique per cluster, so multi-cluster means re-sealing per cluster and managing three separate encrypted copies of every secret.
>
> ESO is the right answer for multi-cluster: one source of truth (Secrets Manager or Vault), each cluster's ESO fetches independently. Rotation means updating Secrets Manager once; all clusters pick it up on the next refresh — no PRs needed for rotation.
>
> I'd add Kyverno as a complementary control: a policy that denies any Secret in production without an ownerReference from an ExternalSecret or SealedSecret. That closes the bypass — no one can `kubectl apply` a plain Secret in prod, which is exactly the kind of finding Wiz surfaces and that Kyverno prevents at admission.
>
> The decision tree: single cluster → Sealed Secrets or SOPS. Multi-cluster + AWS → ESO + Secrets Manager. Need dynamic DB creds or compliance audit log → Vault + ESO. Need per-value diffing in Git → SOPS anywhere."

---

## Self-test drills

### 1. Walk through the trade-offs of Sealed Secrets vs ESO for GitOps secrets.

**Reference answer:** In-Git encrypted ciphertext (Sealed Secrets) vs runtime-pull from external store (ESO). Sealed Secrets: single cluster, self-contained, no external dependency, but per-cluster keypair means multi-cluster is painful, rotation requires PRs, private key backup is the SPOF. ESO: multi-cluster native, rotation without PRs, external store dependency (resilient but required to be reachable). Add Kyverno enforcement as the complement to either.

### 2. Where does SOPS fit in the comparison?

**Reference answer:** SOPS is in-Git like Sealed Secrets but without the per-cluster keypair problem. Encrypts only values (diff-friendly). KMS key works across all clusters. No runtime dependency after the file is decrypted. Trade-off vs Sealed Secrets: key management is your responsibility (IAM or age key distribution). Trade-off vs ESO: rotation still requires re-encrypting and committing. Best for: diff-ability requirement, Helm/Kustomize workflows, no external store preference.

### 3. How does Kyverno complement your secrets management choice?

**Reference answer:** Any secrets management tool can be bypassed by `kubectl apply` of a plain Secret. Kyverno ClusterPolicy (Enforce mode) denies admission of any Secret in production that lacks an ownerReference from ExternalSecret or SealedSecret. This closes the bypass gap. In a Wiz-mature environment, this prevents the "plaintext Secret in production namespace" finding from occurring rather than just detecting it. The policy applies regardless of which tool you use — it's a floor, not a replacement.

### 4. When is Vault the right answer over Secrets Manager?

**Reference answer:** Three reasons Vault wins: (1) dynamic credentials — Vault's DB secrets engine creates short-lived users that Secrets Manager can't; (2) multi-cloud/on-prem — Vault runs anywhere, Secrets Manager is AWS-native; (3) compliance-grade audit log — Vault's structured per-request audit log is more complete than CloudTrail for some auditors. The cost: Vault HA is operationally heavier (unsealing, policy admin, HA cluster). For a team that doesn't need dynamic credentials and is fully on AWS, ESO + Secrets Manager is simpler and still very good.

---

## Further reading / watching

- **Each topic's own deep-dive** — read the four individual docs for the detailed mechanics. This doc is the comparison; the individual docs are the depth.
- **"GitOps Secrets: The Ultimate Comparison"** — search for this topic on the CNCF blog, Banzai Cloud, or Natan Yellin's blog. Several good recent comparisons exist.
- **ESO docs: "Comparison with Secrets Store CSI Driver"** — (`external-secrets.io`) — a complementary comparison for the "ESO vs CSI-based injection" question that often comes up alongside this one.
- **Kyverno policy library** (`kyverno.io/policies`) — examples for Secret governance policies. The library has ready-made policies for many of the Wiz-surface findings.
- **"HashiCorp vs AWS for secrets management" blog posts** — HashiCorp's own comparison is obviously biased, but the concrete feature matrix is accurate.

---

## The 4 dimensions (senior framing)

- **Tech**: in-Git (Sealed Secrets, SOPS) vs runtime-pull (ESO, Vault+ESO) is the primary axis. Multi-cluster and rotation frequency are the two dimensions that determine which side of the axis is right. Dynamic credentials only from Vault. No-external-dependency resilience only from in-Git tools.
- **People**: in-Git tools (Sealed Secrets, SOPS) require a sealing/encrypting workflow step for developers. ESO/Vault requires a SecretStore setup step once per cluster (platform team) and then simple ExternalSecret YAMLs for developers. Kyverno policy enforcement means developers get clear rejection messages if they try to bypass. Document "how to add a new secret" as a runbook per tool — don't assume people will read the docs.
- **CI/CD**: for all tools, the ExternalSecret or SealedSecret YAML is the artifact in the PR. For SOPS, the encrypted YAML is the PR artifact. Code review is possible in all cases (values are never readable in review). Kyverno admission policy is the CI gate — enforced at the Kubernetes API level, not just in the pipeline. Pipeline itself never needs to handle secret values.
- **Operations**: alert on ExternalSecret status being False for >15 min (ESO/Vault). Alert on Vault seal status changes (Vault+ESO). Backup Sealed Secrets controller keypairs out-of-band (Sealed Secrets). Quarterly review of `.sops.yaml` recipients and Vault policies. Annual key rotation exercise (test it before you need it in an incident).
