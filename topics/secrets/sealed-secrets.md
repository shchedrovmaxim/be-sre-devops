# Sealed Secrets — the deep-dive

> **Goal**: by the end you can answer — **"Why might Sealed Secrets be a bad fit for a multi-cluster GitOps setup?"** — covering asymmetric encryption mechanics, scope binding, the private-key backup risk, and why multi-cluster forces you to re-seal per cluster.

> Start with the [simple version](./sealed-secrets-simple.md) if you haven't. The wax-seal analogy is the mental model.

---

## The senior framing — Sealed Secrets solves Git safety, not secrets management

There's a subtle distinction worth making upfront.

**Sealed Secrets solves one problem**: how do you store a Kubernetes `Secret` in a Git repo without the secret being readable? It does not solve: secret rotation, multi-cluster distribution, audit trails, or access policy. It's encryption-at-rest for Git.

Mid-level: "We use Sealed Secrets so secrets are safe in Git."
Senior: "We use Sealed Secrets because it solved our single-cluster GitOps secret problem with minimal operational overhead. In a multi-cluster setup I'd reach for ESO instead, because Sealed Secrets requires a distinct keypair per cluster and re-sealing on every rotation."

---

## How it works — the full mechanics

### Step 1: the controller generates an RSA keypair

When you install the Sealed Secrets controller (typically via Helm into `kube-system`), it generates an **RSA-4096 keypair** on first startup. The private key is stored as a `Secret` named `sealed-secrets-key` in `kube-system`. The public key is fetchable by any `kubeseal` client.

```bash
# Fetch the public certificate for sealing (safe to share)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > my-cluster-public-cert.pem
```

The public cert can be committed to Git. It's not sensitive.

### Step 2: `kubeseal` encrypts per-value

You have a plain `Secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: cG9zdGdyZXM=   # base64("postgres")
  password: c3VwZXJzZWNyZXQ= # base64("supersecret")
```

Run `kubeseal`:

```bash
kubeseal \
  --cert my-cluster-public-cert.pem \
  --format yaml \
  < db-credentials-plain.yaml \
  > db-credentials-sealed.yaml
```

The output is a `SealedSecret` resource where **each value is independently encrypted**:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBy8hX...   # independently encrypted ciphertext
    password: AgCi3Kz...   # independently encrypted ciphertext
  template:
    metadata:
      name: db-credentials
      namespace: production
```

This `SealedSecret` YAML is safe to commit to Git. The values are AES-256-CBC encrypted (RSA is used to encrypt the symmetric key, not the data directly — hybrid encryption). The controller's private key is the only thing that can decrypt them.

### Step 3: the controller reconciles

The Sealed Secrets controller watches for `SealedSecret` resources cluster-wide. When it sees one:

1. It reads the `name` and `namespace` from the metadata (part of the integrity check).
2. It decrypts the `encryptedData` values using the private RSA key.
3. It creates (or updates) a plain `Secret` with `ownerReference` pointing to the `SealedSecret`.
4. If decryption fails for any reason, the controller logs an error and does not create the `Secret`.

The resulting plain `Secret` has an `ownerReference` — if you delete the `SealedSecret`, the `Secret` is garbage-collected.

---

## Scope binding — the intentional security constraint

When `kubeseal` encrypts a value, it embeds the **name and namespace** of the target resource as part of the plaintext that's authenticated by the encryption scheme (AEAD). This means:

- The ciphertext for `name=db-credentials, namespace=production` cannot be used to create `name=db-credentials, namespace=staging`.
- If you rename the `SealedSecret` or move it to a different namespace, the controller **rejects decryption** with an integrity error.

Three scope modes (set with `--scope` flag on `kubeseal`):

| Scope | Behavior | Use case |
|---|---|---|
| `strict` (default) | Bound to exact name **and** namespace | Production secrets — most secure |
| `namespace-wide` | Bound to namespace only; name can change | When you want to rename secrets without re-sealing |
| `cluster-wide` | No binding — any name, any namespace | Rare; reduces security guarantees significantly |

The default (`strict`) is the right answer for production. Don't reach for `cluster-wide` unless you have a specific reason.

---

## Key rotation — the lifecycle you must understand

The controller supports multiple keypairs simultaneously. Key rotation works like this:

1. The controller generates a new keypair after a configurable `--key-renew-period` (default: **30 days**).
2. Old keypairs are **not deleted** — they're kept with a label `sealedsecrets.bitnami.com/sealed-secrets-key=active` / `=expired`. The controller can still decrypt secrets sealed with old keys.
3. **Re-sealing existing secrets is optional but recommended**. Secrets sealed with an old key still decrypt until you explicitly delete the old keypair.

The practical implication: rotation of the controller key does not force immediate re-sealing of all secrets. You have a window. But if you're in DR and trying to restore only the latest key, you can't decrypt secrets sealed with old keys.

**Key rotation does not rotate the underlying application secret.** If `password=supersecret` is sealed, rotating the controller key still serves `password=supersecret`. To rotate the application secret, you re-seal new content.

### The backup imperative

```bash
# Back up ALL active keypairs (run this before any cluster operation)
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-key-backup.yaml

# Restore into a new cluster
kubectl apply -f sealed-secrets-key-backup.yaml
kubectl rollout restart deployment/sealed-secrets-controller -n kube-system
```

Store this backup in a separate secrets store (Vault, AWS Secrets Manager, 1Password). Not in the cluster. Not in Git. If you lose the cluster and you haven't backed up the keypair, every `SealedSecret` in Git is dead — they're permanently undecryptable.

---

## The multi-cluster problem — the dealbreaker

This is the question's answer. Walk through this carefully in an interview.

Each cluster generates its **own independent RSA keypair**. There is no mechanism to share a keypair across clusters (and sharing it would be a security problem — if cluster-A's private key is compromised, every cluster would be affected).

Consequence: a `SealedSecret` sealed for cluster-A **cannot be decrypted by cluster-B**.

In a single-cluster world: you seal once, commit to Git, controller decrypts. Fine.

In a multi-cluster GitOps world with three environments:

```
dev-cluster  (keypair A)
staging-cluster  (keypair B)
prod-cluster  (keypair C)
```

For every application secret, you need **three separate SealedSecret YAMLs** in Git — one per cluster. Three sealing operations, three PRs (or a more complex pipeline), three things that can drift.

When the underlying application secret rotates (e.g., a DB password changes), you must:
1. Re-seal for dev-cluster → commit `sealed-secret-dev.yaml`
2. Re-seal for staging-cluster → commit `sealed-secret-staging.yaml`
3. Re-seal for prod-cluster → commit `sealed-secret-prod.yaml`

For 50 services across 3 clusters, this is operationally painful and error-prone. ESO (External Secrets Operator) solves this by having a single source-of-truth store (AWS Secrets Manager, Vault) that ESO in each cluster pulls from independently — one rotation, all clusters pick up the new value on the next refresh cycle.

### The gotcha that trips seniors

Sometimes people try to solve the multi-cluster problem by sharing the public cert (exporting cluster-A's cert and using it to seal for cluster-B). **Don't do this.** The cluster-B controller has a different private key and will reject decryption. The only way to share a keypair is to export the private key itself — which you now have to protect, store, and rotate separately, at which point you've reinvented a simpler secrets manager.

---

## Failure modes

| Scenario | What happens | Recovery |
|---|---|---|
| Controller pod down | Existing `Secret` resources remain (GC only on explicit delete). No new `SealedSecret` reconciled. | Restart controller. |
| Private key lost (no backup) | All existing `SealedSecret` resources are permanently unrecoverable. | Re-seal everything from a live/known state of the original secrets. Painful. |
| Wrong namespace in `SealedSecret` | Controller logs integrity error, does not create `Secret`. | Re-seal with correct namespace. |
| Secret Store API secret rotated at source | Nothing — Sealed Secrets doesn't poll anything. The `SealedSecret` in Git still encrypts the old value. | Manually re-seal new value. |
| Controller key rotation without re-sealing | Old `SealedSecrets` still decrypt (old keys are kept). Fine until you delete old keypairs. | Re-seal before deleting old keys. |

---

## ArgoCD integration — the real-world pattern

Sealed Secrets is GitOps-native: ArgoCD syncs `SealedSecret` resources from Git the same way it syncs `Deployment` or `Service` resources. The controller picks them up and creates `Secrets` behind the scenes.

```yaml
# In your ArgoCD Application spec — no special config needed
# Just ensure the Sealed Secrets controller is running in the cluster
# ArgoCD treats SealedSecret as any other CRD resource
```

One nuance: ArgoCD may show `SealedSecret` resources as out-of-sync if it doesn't know the CRD. Solution: install the Sealed Secrets CRD in ArgoCD's known resource list (happens automatically when you add the Helm chart to ArgoCD).

A common pattern is to use `ignoreDifferences` in the ArgoCD `Application` for the `status` field of `SealedSecrets`, since the controller updates the status field and ArgoCD would otherwise constantly flag it as drifted.

---

## The interview answer in 60 seconds

> "Sealed Secrets uses asymmetric encryption: `kubeseal` encrypts each secret value against the cluster's public key; only the in-cluster controller's private key can decrypt it. The resulting `SealedSecret` YAML is safe to commit to Git — it's ciphertext.
>
> The limitation that makes it a bad fit for multi-cluster GitOps is that every cluster has its own unique keypair. A secret sealed for cluster-A is unreadable by cluster-B. So with three environments, you're maintaining three separate encrypted files for every logical secret, and re-sealing three times on every rotation.
>
> There's also a single point of failure: the controller's private key. If you lose it — cluster DR, accidental deletion — none of your existing SealedSecrets can be decrypted. You have to back up the key separately, which adds operational overhead.
>
> For a single-cluster setup with a small team, it's simple and effective. For multi-cluster GitOps at scale, ESO backed by a central store (AWS Secrets Manager or Vault) is the better answer — one rotation propagates everywhere automatically."

---

## Self-test drills

### 1. Why is Sealed Secrets a bad fit for a multi-cluster GitOps setup?

**Reference answer:**
- Each cluster generates its own independent RSA keypair.
- A SealedSecret sealed for one cluster is unreadable by another — different private keys.
- In a three-cluster setup, every secret needs three separate sealed files, three sealing operations, three PRs per rotation.
- Operationally painful at scale; ESO + central store solves this by decoupling encryption from the cluster.
- Bonus: the private-key backup problem is also per-cluster — three clusters, three separate backups required.

### 2. What happens if you lose the Sealed Secrets controller private key?

**Reference answer:**
- Every existing `SealedSecret` resource in Git is permanently undecryptable.
- The encrypted data is RSA+AES encrypted; without the private key, there's no way to recover.
- Recovery requires knowing the plaintext of every secret from some other source and re-sealing from scratch.
- Prevention: regularly back up the keypair with `kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml`, stored in an out-of-cluster secrets store.

### 3. What is scope binding and why does it matter?

**Reference answer:**
- The name and namespace of the target `Secret` are embedded in the authenticated data at seal time.
- A ciphertext sealed for `name=db-creds, namespace=production` cannot be applied to `namespace=staging` — the controller rejects it with an integrity error.
- Security benefit: a leaked SealedSecret YAML cannot be replayed in a different namespace to gain access.
- Practical gotcha: you can't rename or move a SealedSecret without re-sealing. Plan your naming conventions before you seal.

### 4. How does Sealed Secrets compare to ESO for the "secrets in GitOps" problem?

**Reference answer:**
- Sealed Secrets: secrets are encrypted blobs IN Git. Self-contained — no external dependency at runtime beyond the controller. Works offline. Single cluster only.
- ESO: secrets are NOT in Git. ESO syncs from an external store (AWS Secrets Manager, Vault). Multi-cluster native — each cluster's ESO pulls independently. Requires an external store (operational dependency).
- The axis: GitOps-in-Git (Sealed Secrets) vs GitOps-runtime-pull (ESO). For a single cluster where the team wants minimal moving parts, Sealed Secrets. For multi-cluster or where audit trails / central management matter, ESO.
- Bonus: SOPS is a third option — also in-Git, but not cluster-specific; the decryption key can be shared across clusters via KMS.

---

## Further reading / watching

- **bitnami-labs/sealed-secrets GitHub** (`github.com/bitnami-labs/sealed-secrets`) — the README and the `docs/` folder. Read the "Managing existing secrets," "Key rotation," and "Scopes" sections specifically.
- **"Sealed Secrets deep dive"** — search for Bitnami's blog posts on Sealed Secrets. They cover the encryption internals (the hybrid RSA+AES scheme) in detail.
- **ArgoCD docs: "Secret Management"** — `argo-cd.readthedocs.io`. Covers integrating Sealed Secrets with ArgoCD including the `ignoreDifferences` pattern.
- **comparison talk: "GitOps Secrets: Sealed Secrets vs ESO vs SOPS"** — search YouTube for this topic; several good KubeCon talks compare all three.

---

## The 4 dimensions (senior framing)

- **Tech**: RSA-4096 + AES-256 hybrid encryption; `kubeseal` encrypts, controller decrypts; strict scope binding by default; per-cluster keypair means no cross-cluster reuse.
- **People**: simple workflow for small teams — `kubeseal` is one CLI command; no external dependency to learn. But the key backup discipline has to be explicit; it won't happen naturally without policy. Document the backup runbook and test it.
- **CI/CD**: excellent GitOps fit for single-cluster — `SealedSecret` resources sync via ArgoCD like any other manifest. Multi-cluster CI gets expensive: need to seal per cluster in the pipeline, meaning the CI system needs each cluster's public cert and a process for rotating them.
- **Operations**: the single point of failure (controller private key) must be treated like a root secret — backed up, tested-for-restore quarterly, stored in a different system than the cluster. Add a runbook: "how to recover if the Sealed Secrets controller key is lost." Test it before you need it.
