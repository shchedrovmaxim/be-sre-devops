# External Secrets Operator — the deep-dive

> **Goal**: by the end you can answer — **"Walk through what happens when you create an ExternalSecret referencing AWS Secrets Manager."** — covering the SecretStore/ExternalSecret CRD split, the IRSA auth chain, the reconciliation loop, and what happens when things go wrong.

> Start with the [simple version](./simple.md) if you haven't. The personal-shopper analogy is the mental model.

---

## The senior framing — ESO decouples the secret's existence from its location in Git

Mid-level: "ESO syncs secrets from AWS into Kubernetes."
Senior: "ESO moves the secret out of the GitOps loop entirely. What lives in Git is a declaration of *where* and *how* to fetch a secret — not the secret itself. The actual value stays in the system of record (Secrets Manager, Vault), where it has audit trails, rotation policies, and access control. ESO is the bridge."

That distinction matters in conversations about security posture: you can show an auditor the `ExternalSecret` YAML (says nothing sensitive) and point to AWS Secrets Manager as the controlled, audited store. The Git repo is no longer a secret surface.

---

## The CRD model — three resources you need to know

### SecretStore

Namespace-scoped. Defines how ESO authenticates to an external backend for secrets in that namespace.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-production-sa   # annotated with IAM role ARN
```

### ClusterSecretStore

Cluster-scoped. Any namespace's `ExternalSecret` can reference it. Use when one IAM role can access all secrets and you don't need namespace isolation.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-global
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-sa
            namespace: external-secrets   # where ESO is installed
```

### ExternalSecret

The declaration of what to fetch and where to put it:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h                    # how often to poll
  secretStoreRef:
    name: aws-global
    kind: ClusterSecretStore
  target:
    name: db-credentials                 # name of the K8s Secret to create
    creationPolicy: Owner                # ESO owns the Secret lifecycle
    deletionPolicy: Retain               # if ExternalSecret is deleted, keep the Secret
  data:
    - secretKey: username                # key in the K8s Secret
      remoteRef:
        key: my-app/production/db        # AWS Secrets Manager secret name
        property: username               # JSON key inside the secret (optional)
    - secretKey: password
      remoteRef:
        key: my-app/production/db
        property: password
```

The produced Kubernetes `Secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
  ownerReferences:
    - apiVersion: external-secrets.io/v1beta1
      kind: ExternalSecret
      name: db-credentials
      # controller: true, blockOwnerDeletion: true
data:
  username: <base64 of fetched value>
  password: <base64 of fetched value>
```

---

## The IRSA auth chain — what actually happens when ESO calls AWS

This is where most engineers have gaps. "IRSA" gets said a lot; the actual exchange is less understood.

1. **ESO pod starts** with a projected `ServiceAccount` token (automounted at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`). This is a short-lived OIDC JWT signed by the EKS cluster's OIDC issuer.

2. **ESO calls AWS STS**: `AssumeRoleWithWebIdentity`, passing the OIDC JWT as the `WebIdentityToken` and the IAM role ARN (from the ServiceAccount annotation).

3. **AWS STS validates the JWT** against the EKS OIDC issuer URL (which EKS has registered in IAM). Checks: token not expired, audience is `sts.amazonaws.com`, issuer is trusted.

4. **STS returns temporary credentials** (access key, secret key, session token) with the permissions of the assumed role — typically `secretsmanager:GetSecretValue` on specific ARNs.

5. **ESO uses those temporary credentials** to call `secretsmanager:GetSecretValue` for the ARN referenced in the `ExternalSecret`.

The temporary credentials expire (typically 1 hour). ESO refreshes them automatically using the projected token, which Kubernetes rotates every ~24 hours (or on expiry).

What this means for you: if the IAM role's trust policy doesn't match the ESO ServiceAccount (`sub: system:serviceaccount:external-secrets:eso-sa`), step 3 fails with an access denied and ESO can't fetch anything. This is the most common misconfiguration.

---

## The reconciliation loop — what ESO does on every refresh

1. **Fetch** — ESO calls the backend API (Secrets Manager `GetSecretValue`), requesting the current version.
2. **Diff** — ESO compares the fetched value against what's currently in the Kubernetes `Secret`.
3. **Write** — if different, ESO patches the `Secret`. If the `Secret` doesn't exist yet, it creates it.
4. **Update status** — ESO updates the `ExternalSecret`'s `.status` field: `Ready: true/false`, `refreshTime`, `syncedResourceVersion`.

The status is the source of truth for "is this working":

```bash
kubectl get externalsecret db-credentials -n production
# NAME             STORE        REFRESH INTERVAL   STATUS    READY
# db-credentials   aws-global   1h                 SecretSynced   True

kubectl describe externalsecret db-credentials -n production
# Conditions:
#   Type: Ready
#   Status: True
#   Reason: SecretSynced
#   Last Transition: 2026-06-05T10:00:00Z
```

If status is `SecretSyncedError`, the `message` field tells you why (often: auth failure, secret not found, or rate limited).

---

## Auth patterns — a comparison

| Auth method | When to use | Risk |
|---|---|---|
| **IRSA** (IAM role via OIDC) | EKS clusters — the right default | Low; no long-lived credentials in the cluster |
| **EKS Pod Identity** | Newer EKS feature; similar to IRSA but managed by EKS directly | Low; even simpler OIDC setup than IRSA |
| **Static IAM credentials** | Local dev, small teams, non-EKS K8s | High; long-lived key must be rotated, stored as a K8s Secret (ironic) |
| **Vault Kubernetes auth** | On-prem, multi-cloud, or Vault as the backend | Medium; requires Vault to trust the cluster's ServiceAccount tokens |
| **GCP Workload Identity** | GKE clusters fetching from Secret Manager | Low; same concept as IRSA, GCP's implementation |

For EKS + AWS Secrets Manager, IRSA is the canonical answer. Static credentials are a last resort and a finding in any security review.

---

## ESO generators — the feature most people miss

ESO can also *generate* secrets, not just sync them. Useful cases:

**Random password generator**: create a random password, store it in Secrets Manager via a push policy, and sync it into a Kubernetes Secret. Used for bootstrapping DB passwords that need to be stored somewhere.

**GitHub token generator**: ESO can generate short-lived GitHub installation tokens from an app private key. Useful for CI/CD pipelines running in-cluster.

**Password generator**:

```yaml
apiVersion: generators.external-secrets.io/v1alpha1
kind: Password
metadata:
  name: db-initial-password
  namespace: production
spec:
  length: 32
  digits: 5
  symbols: 5
  symbolCharacters: "-_$@"
  noUpper: false
```

Referenced in an `ExternalSecret` via `sourceRef.generatorRef`. Niche but useful for bootstrapping.

---

## Failure modes — what can break and what the impact is

### 1. Auth failure (IRSA role misconfigured, token expired, role revoked)

ESO logs: `failed to fetch secret: AccessDeniedException`. Status flips to `SecretSyncedError`. The Kubernetes `Secret` retains its last-known-good value. ESO retries on the next refresh cycle.

Impact: if the application reads the `Secret` on startup (typical), it gets the last-good value until the pod restarts. If auth is broken for longer than the pod restart cycle, deployments that need the Secret may fail.

Detection: alert on ESO `ExternalSecret` status conditions being `False` for more than N minutes.

### 2. Source secret deleted in AWS Secrets Manager

If the secret is deleted in Secrets Manager and ESO tries to fetch it, it gets `ResourceNotFoundException`. Status flips to `SecretSyncedError`.

The `deletionPolicy` on the `ExternalSecret` controls what happens to the Kubernetes `Secret`:
- `Retain` (default): Kubernetes `Secret` is kept as-is. Safe but stale.
- `Delete`: Kubernetes `Secret` is deleted. Application pods that depend on it may crash-loop.
- `Merge`: only the keys fetched by this `ExternalSecret` are removed; other keys in the `Secret` remain.

In production, `Retain` is usually the right choice — a stale secret is better than a deleted one until the team can investigate.

### 3. AWS API rate limiting

AWS Secrets Manager has a quota of ~10,000 API calls per second per region (combined). With many `ExternalSecret` resources polling on short intervals, you can hit this. ESO respects rate limits and backs off, but in a cluster with hundreds of ExternalSecrets on 1m refresh intervals, you'll see throttling errors.

Mitigation: use longer refresh intervals (1h is fine for most secrets), or use ESO's `dataFrom` with JSON secrets (one API call fetches multiple keys at once from a single secret object in Secrets Manager).

### 4. The `creationPolicy: Merge` footgun

If you use `creationPolicy: Merge`, ESO merges its fetched keys into an existing `Secret`. If another process also writes to that `Secret`, you get a conflict loop. Stick to `creationPolicy: Owner` for ESO-managed secrets; create separate `Secrets` for things managed by other tools.

---

## Multi-cluster patterns

Since ESO pulls from an external store, multi-cluster is straightforward:

```
cluster-dev   → ESO → AWS Secrets Manager (dev secrets)
cluster-prod  → ESO → AWS Secrets Manager (prod secrets)
```

Each cluster has its own ESO installation, its own `ClusterSecretStore` with its own IAM role (separate IAM role per cluster + environment). The `ExternalSecret` YAMLs in Git are identical across clusters (they reference the same logical key in Secrets Manager). The IAM policies control which cluster can access which secrets.

The GitOps pattern: the `ExternalSecret` manifests are the same across environments, with only the `SecretStore` name or the secret path varying. Use Kustomize overlays or Helm values to parameterize per environment.

You can also run a single ESO installation in a "control cluster" and use ESO's `PushSecret` resource to push into remote clusters — but this is more complex and less common.

---

## Wiz / Kyverno integration — closing the loop

If you're using Kyverno to enforce "no plaintext Secrets in Git" (a common policy in shops that have done Wiz-level security reviews), ESO is the complementary mechanism:

- Kyverno policy: `deny` any `Secret` resource that was not created by ESO (check for `ownerReference` to an `ExternalSecret`).
- This means devs can't manually `kubectl apply` a raw `Secret` in production; every secret must flow through ESO.
- The `ExternalSecret` YAMLs in Git contain no secret values — they're safe to audit and review.

This gives you: Git as the source for "what secrets exist and where", Secrets Manager as the source for "what those secrets are worth", and ESO as the enforcement mechanism. Kyverno closes the loop by preventing bypass.

---

## The interview answer in 60 seconds

> "When I create an `ExternalSecret` referencing AWS Secrets Manager, the ESO controller picks it up and first resolves the auth. On EKS, this is IRSA: ESO's ServiceAccount has an IAM role annotation, so the controller exchanges its projected OIDC token with AWS STS for temporary credentials scoped to that role.
>
> With credentials in hand, ESO calls `secretsmanager:GetSecretValue` for the key named in the `ExternalSecret`. It decodes the value, creates (or patches) a Kubernetes `Secret` in the target namespace, and sets an `ownerReference` back to the `ExternalSecret` for lifecycle management.
>
> After that initial fetch, ESO re-polls on the `refreshInterval` (say, 1 hour). If the value in Secrets Manager changed, ESO patches the Kubernetes `Secret`. If auth breaks, ESO logs an error and the `ExternalSecret` status flips to `SecretSyncedError` — but the last-good Kubernetes `Secret` remains until the pod that uses it restarts.
>
> The key design win: nothing sensitive is in Git. The `ExternalSecret` YAML says 'fetch from here' but contains no secret value. The actual value lives in Secrets Manager, which has audit logs, rotation policies, and IAM-controlled access."

---

## Self-test drills

### 1. Walk through what happens when you create an ExternalSecret referencing AWS Secrets Manager.

**Reference answer**: see "The interview answer in 60 seconds" above. Key beats: SecretStore auth via IRSA (OIDC exchange with STS) → `GetSecretValue` call → Kubernetes `Secret` created with `ownerReference` → re-poll on `refreshInterval` → status updated on `ExternalSecret`.

### 2. What's the difference between SecretStore and ClusterSecretStore? When do you use each?

**Reference answer:**
- `SecretStore` is namespace-scoped — only `ExternalSecret` resources in the same namespace can reference it.
- `ClusterSecretStore` is cluster-scoped — any namespace can use it.
- Use `ClusterSecretStore` when one IAM role can read all secrets and you want to avoid duplicating the auth config per namespace.
- Use per-namespace `SecretStore` when teams have separate IAM roles and need namespace-level auth isolation (each team can only reach their own secrets).

### 3. What happens if the source secret in AWS Secrets Manager is deleted?

**Reference answer:**
- ESO gets `ResourceNotFoundException` on the next refresh.
- `ExternalSecret` status flips to `SecretSyncedError`.
- The Kubernetes `Secret` behavior depends on `deletionPolicy`: `Retain` (default) keeps the last-good Secret, `Delete` removes it (dangerous — pods may crash-loop).
- Use `Retain` for production; monitor for `SecretSyncedError` status conditions and alert.

### 4. How would you prevent engineers from bypassing ESO and manually creating plain Secrets?

**Reference answer:**
- Kyverno `ClusterPolicy`: deny admission of any `Secret` resource in production namespaces that doesn't have an `ownerReference` pointing to an `ExternalSecret`.
- This means every Secret must flow through ESO — the `ExternalSecret` YAML is the only path to creating a Secret in prod.
- Bonus: complement with a Wiz/Trivy scan that flags plain Secrets in Git manifests (in case someone tries to commit raw Secret YAMLs).

---

## Further reading / watching

- **external-secrets.io** — the official docs. The "Guides" section on AWS provider and IRSA setup is the most immediately useful.
- **"External Secrets Operator — From Zero to Production" talks** — search KubeCon/YouTube. Several good walkthroughs of the IRSA chain and multi-cluster patterns.
- **GitHub: external-secrets/external-secrets** — look at the `examples/` directory for real-world SecretStore and ExternalSecret YAML patterns.
- **"ESO vs Secrets Store CSI Driver"** — ESO creates native Secrets; CSI Driver mounts secrets as volumes. Blog post comparison worth reading; the choice matters when apps expect file-based secrets vs env vars.

---

## The 4 dimensions (senior framing)

- **Tech**: SecretStore + ExternalSecret CRDs; IRSA or Vault K8s auth; refresh polling model; `ownerReference`-based lifecycle; `deletionPolicy` governs recovery behavior; generators for bootstrapping. Multi-cluster is native — each cluster has its own ESO + auth config pulling from the same store.
- **People**: simpler for developers than Sealed Secrets — no `kubeseal` CLI required; just write an `ExternalSecret` YAML. The operational burden shifts to whoever manages the SecretStore config and IAM roles (platform team). Runbook: "ExternalSecret stuck in SecretSyncedError — check IAM role, check Secrets Manager ARN, check ESO pod logs."
- **CI/CD**: `ExternalSecret` YAMLs are safe to commit and review in PRs — no secret values. Kyverno policy in admission enforces that all Secrets in prod have ESO ownership. The pipeline itself doesn't need to handle secrets at all; ESO handles delivery.
- **Operations**: alert on `ExternalSecret` status conditions. High-severity: `Ready=False` for more than 15 minutes in production (auth broken, secret missing). Medium-severity: AWS rate limiting errors (solution: increase refresh interval, use `dataFrom` for batch fetching). Quarterly review: audit which ExternalSecrets exist, what IAM roles they use, whether old secrets should be rotated or removed.
