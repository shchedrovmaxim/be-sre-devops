# External Secrets Operator — the simple version (the personal shopper)

> Read this first. Once the analogy clicks, the [deep-dive doc](./external-secrets-operator.md) fills in the mechanics.

This doc explains **one idea**:

> **ESO is a personal shopper that fetches the real thing from a store on your behalf. You tell it what you want and where to find it; it keeps your fridge stocked automatically.**

That's the whole concept. Everything else — SecretStore, ExternalSecret, refresh intervals, IRSA auth — is just detail on top.

---

## The personal shopper analogy

Imagine you hire a personal shopper. You hand them a list:

> "Get me milk from Store A on the corner. Keep the fridge stocked. Check every hour. If they're out of milk, tell me."

You don't go to the store yourself. You don't hold the cash. You just give instructions and receive the stocked fridge.

External Secrets Operator works the same way:

| In the shopping world | In the ESO world |
|---|---|
| The grocery store | AWS Secrets Manager / Vault / GCP Secret Manager |
| Your shopping list | The `ExternalSecret` resource in Kubernetes |
| The store loyalty card (how to get in) | The `SecretStore` or `ClusterSecretStore` (auth config) |
| The personal shopper | The ESO controller pod |
| Your fridge (always stocked) | The Kubernetes `Secret` ESO creates and keeps fresh |
| "Check every hour" | The `refreshInterval` on the ExternalSecret |

ESO runs as a controller in your cluster. It reads `ExternalSecret` resources that say "fetch secret X from store Y every N minutes." It authenticates to the external store (using IRSA on AWS, for example), fetches the value, and creates or updates a Kubernetes `Secret`. Every `refreshInterval`, it checks whether the value has changed and updates the `Secret` if needed.

---

## The 2-3 sentence summary

ESO is a Kubernetes operator that syncs secrets from external stores (AWS Secrets Manager, Vault, GCP Secret Manager, etc.) into Kubernetes `Secret` resources. You configure a `SecretStore` once with how to authenticate, then create `ExternalSecret` resources that say "fetch this key from that store." ESO keeps the Kubernetes `Secret` in sync automatically, on a configurable refresh interval.

---

## The 2 concepts that trip people up

### 1. SecretStore vs ClusterSecretStore — what's the difference?

A `SecretStore` is namespace-scoped: only `ExternalSecret` resources in the same namespace can use it. A `ClusterSecretStore` is cluster-scoped: any namespace can reference it.

In practice: for a shared AWS account where the ESO auth (IRSA role) can read from any Secrets Manager secret, use a `ClusterSecretStore`. For namespace-isolated auth (each team has its own IAM role and their own secrets), use per-namespace `SecretStore`.

### 2. Rotation is automatic — but "automatic" means "on the next refresh"

If the underlying secret in AWS Secrets Manager is rotated, ESO doesn't get a webhook or event. It just polls. The `refreshInterval` is the lag: if set to `1h`, the Kubernetes `Secret` can be up to 1 hour stale after the source rotates.

For most production secrets this is fine. For security-sensitive rotation (compromised credential, forced rotation), you can trigger an immediate refresh by adding an annotation or deleting the `ExternalSecret` and letting ESO recreate it.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Where do the actual secret values live? | In the external store (AWS Secrets Manager, Vault, etc.) — NOT in Git |
| What's in Git? | The `ExternalSecret` YAML — which describes *where* to fetch, not the secret value itself |
| What does ESO create in the cluster? | A plain Kubernetes `Secret` — the same as you'd create manually |
| How does ESO authenticate to AWS? | Most commonly via **IRSA** (an IAM role bound to the ESO ServiceAccount) |
| What if auth breaks? | ESO logs an error, stops updating the Secret, but the last-known-good Secret remains |
| What's refreshInterval? | How often ESO checks the external store for changes. Default is 1h. |
| What if the source secret is deleted? | Configurable: ESO can delete the Kubernetes Secret, or leave it as-is. Set `deletionPolicy`. |

---

## Self-test (the killer interview question)

Out loud:

> **"Walk through what happens when you create an ExternalSecret referencing AWS Secrets Manager."**

**Reference answer (intuitive version):**

"First, there's a `SecretStore` or `ClusterSecretStore` that tells ESO how to authenticate to AWS — on EKS, this is typically done via IRSA: the ESO controller's `ServiceAccount` has an annotation pointing to an IAM role that has `secretsmanager:GetSecretValue` permission for the specific secret ARN.

When I create an `ExternalSecret` that says 'fetch `my-app/db-password` from AWS Secrets Manager and put it in a Kubernetes Secret called `db-credentials` in namespace `production`', the ESO controller picks it up immediately and does the first fetch. It calls AWS Secrets Manager via the IRSA token — an OIDC exchange that trades the pod's projected ServiceAccount token for temporary AWS credentials — fetches the secret value, and creates the `Secret` resource.

After that, ESO re-fetches on every `refreshInterval` (say, every hour). If the value in Secrets Manager changed, ESO updates the Kubernetes Secret. If there's an auth error, ESO logs it and stops updating — but the last-good Secret stays in place.

The produced Kubernetes Secret has an `ownerReference` back to the `ExternalSecret`, so it's lifecycle-managed. If you delete the `ExternalSecret`, the Secret can be cleaned up too, depending on the `deletionPolicy`."

---

## Further reading / watching

- **External Secrets Operator docs** (`external-secrets.io`) — "Getting Started" and "Guides → AWS Secrets Manager" are the most useful sections.
- **"ESO with IRSA on EKS"** — search the ESO docs for "AWS" provider. Covers the ServiceAccount annotation + IRSA setup in detail.
- **"ESO vs Secrets Store CSI Driver"** — a common comparison; ESO creates native Kubernetes Secrets, CSI Driver mounts secrets as volumes. Blog posts comparing both are easy to find.

---

## Next: the deep-dive

When the personal-shopper analogy and the SecretStore/ExternalSecret split feel obvious, jump to [`external-secrets-operator.md`](./external-secrets-operator.md). The deep-dive covers:

- The full reconciliation loop — what ESO does on every refresh, how it diffs, ownerReferences
- All the auth patterns: IRSA, pod identity, Vault K8s auth, static credentials (and why static is bad)
- ESO generators: creating random passwords, rotating GitHub tokens, templating secrets
- Failure modes in detail: auth loss, source secret deleted, rate limiting from AWS
- The `deletionPolicy` and `managedSecretStore` options and when they bite you
- Multi-cluster patterns: ClusterSecretStore vs per-namespace; one ESO vs one per cluster
- The comparison to Sealed Secrets and SOPS on the "what lives in Git" axis
