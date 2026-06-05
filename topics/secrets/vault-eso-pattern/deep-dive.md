# Vault as source of truth + ESO sync pattern — the deep-dive

> **Goal**: by the end you can answer — **"Design a secrets management story for 50 microservices across 3 clusters."** — covering Vault as the central store, ESO auth (K8s auth + IRSA), dynamic DB secrets, HA Vault operations, and the comparison to AWS Secrets Manager + ESO.

> Start with the [simple version](./simple.md) if you haven't. The bank vault + armored car analogy is the mental model.

---

## The senior framing — Vault wins on audit, dynamic credentials, and portability

ESO + AWS Secrets Manager is a strong pattern. For many teams it's the right answer. But Vault adds three things Secrets Manager can't match:

1. **Dynamic secrets** — Vault doesn't just store and return secrets; it can *generate* short-lived credentials on demand (DB users, AWS STS tokens, PKI certificates). Secrets Manager stores static values.
2. **Portable, unified store** — Vault runs anywhere (EKS, GKE, bare metal, on-prem). One API, one policy model, regardless of cloud. Secrets Manager is AWS-native; using it multi-cloud requires per-cloud stores with different APIs.
3. **Compliance-grade audit log** — every Vault API call is logged: who authenticated as what identity, what path they accessed, at what time. Secrets Manager has CloudTrail, which is close — but Vault's audit log is structured, queryable, and often what compliance auditors explicitly require.

The honest trade-off: Vault is operationally heavier than Secrets Manager. It has an HA setup to manage, unsealing mechanics, policy administration, and upgrade cycles. The question is whether your team has the maturity and the need.

---

## Vault HA architecture — the production setup

### Storage backend

Vault needs a storage backend for its encrypted data. Two options dominate:

**Raft (integrated storage)** — Vault nodes form a Raft consensus cluster. No external storage dependency. Recommended for new deployments. Run 3 or 5 nodes for quorum.

**Consul** — older pattern; Vault uses Consul KV for storage. More moving parts. Not recommended for new setups unless you're already running Consul.

### Node roles

In a Raft cluster:
- **Active node**: handles all read/write requests.
- **Standby nodes** (1-4): replicate data, forward requests to active, take over if active fails. Failover is automatic via Raft leader election.
- **Performance replication** (Vault Enterprise): read-only replica clusters in different regions. Local reads, writes forwarded to primary.

Kubernetes deployment:

```yaml
# Vault Helm chart (recommended install method)
# values.yaml snippet
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
  affinity: |
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
          topologyKey: kubernetes.io/hostname
```

Three Vault pods, hard anti-affinity to separate nodes. Typical production setup.

### Sealing and unsealing — the lifecycle you must explain

Vault starts in a **sealed** state. A sealed Vault cannot decrypt its own data — the encryption key is not in memory.

**Shamir secret sharing**: the master key is split into N shares. You need K of N shares to unseal. Example: 5 shares, threshold 3. Three people each hold one share; any three together can unseal.

In practice, manual unsealing with Shamir shares is operationally painful (needs 3 humans after every pod restart). The modern solution:

**Auto-unseal with cloud KMS**: Vault uses AWS KMS (or GCP/Azure equivalent) to automatically unseal on startup. The master key is stored encrypted by the KMS key. When Vault starts, it calls `kms:Decrypt` to recover the master key and unseal itself.

```hcl
# Vault server config
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abc-..."
}
```

With auto-unseal, Vault pod restarts (during upgrades, node replacement, etc.) are transparent. The pod comes back sealed, calls KMS, unseals in seconds. Operators are not paged.

**What "sealed" means for ESO**: if Vault is sealed (KMS unavailable, key deleted, or manual seal), ESO cannot fetch secrets. All `ExternalSecret` statuses flip to `SecretSyncedError`. The last-good Kubernetes `Secrets` remain, but no new values can be fetched. This is the most operationally significant failure mode of the Vault + ESO pattern — and why Vault HA + auto-unseal is non-negotiable in production.

---

## Authentication — solving "secret zero"

### K8s auth method (the right answer for in-cluster ESO)

Vault has a Kubernetes auth method that trusts the K8s API server. Setup:

1. **Enable the K8s auth method** in Vault: `vault auth enable kubernetes`
2. **Configure it** with the cluster's CA and API server URL (or Vault running inside the cluster uses the cluster's own JWT issuer).
3. **Create a Vault role** that maps a K8s ServiceAccount to a Vault policy:

```bash
vault write auth/kubernetes/role/eso-production \
  bound_service_account_names=eso-sa \
  bound_service_account_namespaces=external-secrets \
  policies=eso-production-policy \
  ttl=1h
```

4. **ESO authenticates**: when ESO needs a Vault token, it presents its ServiceAccount JWT to `vault auth kubernetes login`. Vault calls the K8s API server to verify the JWT. Vault returns a token valid for 1 hour with the permissions of `eso-production-policy`.

Why this solves secret zero: the ServiceAccount token is already present in the pod (Kubernetes automounts it). No pre-placed credential is needed. Vault's K8s auth bootstraps itself from the cluster's existing identity system.

### IRSA / AWS IAM auth (EKS with Vault on AWS)

If Vault is running on AWS (EC2, EKS) and ESO is on EKS:

```bash
vault write auth/aws/role/eso-production \
  auth_type=iam \
  bound_iam_principal_arn=arn:aws:iam::123456789012:role/eso-production-role \
  policies=eso-production-policy \
  ttl=1h
```

ESO's IRSA role authenticates to Vault via the AWS IAM auth method. Same OIDC exchange as with AWS Secrets Manager, but terminates at Vault instead. Useful when ESO is already using IRSA for other providers and you want a unified auth pattern.

### The `SecretStore` config for Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-production
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"               # KV secrets engine mount
      version: "v2"                # KV v2 (supports versioning)
      auth:
        kubernetes:
          mountPath: "kubernetes"  # the auth method path in Vault
          role: "eso-production"   # the Vault role
          serviceAccountRef:
            name: eso-sa
            namespace: external-secrets
```

---

## Dynamic secrets — the feature that separates Vault from everything else

### Database secrets engine

Vault can connect to a database (PostgreSQL, MySQL, MongoDB, etc.) and create short-lived credentials on demand.

Setup:

```bash
# Enable the DB secrets engine
vault secrets enable database

# Configure a PostgreSQL connection
vault write database/config/my-postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres.internal:5432/mydb" \
  allowed_roles="my-app-role" \
  username="vault-root-user" \
  password="root-password"

# Create a role that defines what credentials look like
vault write database/roles/my-app-role \
  db_name=my-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE my-app-group;" \
  default_ttl="1h" \
  max_ttl="24h"
```

When an app (or ESO) requests credentials from `database/creds/my-app-role`:
1. Vault connects to PostgreSQL.
2. Vault executes the `creation_statements` with a generated username and password.
3. Vault returns the credentials to the caller, along with a lease TTL (1 hour by default).
4. After 1 hour, Vault revokes the credentials (executes `REVOKE ROLE`). The PostgreSQL user no longer exists.

A leaked credential is useless after 1 hour. No manual rotation needed. The attack surface for credential theft is bounded by the TTL.

### ESO + dynamic secrets

ESO can fetch from the DB secrets engine:

```yaml
spec:
  data:
    - secretKey: username
      remoteRef:
        key: database/creds/my-app-role
        property: username
    - secretKey: password
      remoteRef:
        key: database/creds/my-app-role
        property: password
```

On each refresh, ESO fetches new credentials from Vault (Vault creates a new short-lived user). The Kubernetes `Secret` is updated with the new credentials. The app needs to handle credential rotation — typically by watching for `Secret` changes (using Reloader or similar) to restart/reconfigure.

The `refreshInterval` on the `ExternalSecret` should be shorter than the credential TTL. If TTL is 1h, refresh every 45 minutes. If the refresh fails, you have 15 minutes before the credential expires.

---

## Vault for 50 services across 3 clusters — the design

```
Vault HA cluster (3 nodes, Raft, auto-unseal via KMS)
  ├── KV secrets engine (v2): static secrets
  │    ├── secret/production/myapp-a/config   → ExternalSecret in cluster-prod
  │    ├── secret/staging/myapp-a/config      → ExternalSecret in cluster-staging
  │    └── secret/dev/myapp-a/config          → ExternalSecret in cluster-dev
  └── DB secrets engine: dynamic credentials
       └── database/creds/myapp-a-db-role     → ExternalSecret with short TTL

Per-cluster:
  cluster-prod:    ESO → K8s auth role "eso-prod"    → policy "production-reader"
  cluster-staging: ESO → K8s auth role "eso-staging" → policy "staging-reader"
  cluster-dev:     ESO → K8s auth role "eso-dev"     → policy "dev-reader"

Per-service Vault policy (example for myapp-a in production):
  path "secret/data/production/myapp-a/*" { capabilities = ["read", "list"] }
  path "database/creds/myapp-a-db-role"   { capabilities = ["read"] }
```

Each cluster's ESO authenticates with its own K8s auth role. The policy scopes what that cluster's ESO can read. A misconfigured or compromised cluster's ESO can only access the secrets that cluster's policy allows — blast radius is bounded.

Path namespacing (`production/`, `staging/`, `dev/`) is enforced by policy, not by convention. An ESO in the dev cluster physically cannot read production secrets even if the `ExternalSecret` specifies the wrong path — Vault denies it.

---

## Vault vs AWS Secrets Manager + ESO — when to pick each

| Dimension | Vault + ESO | AWS Secrets Manager + ESO |
|---|---|---|
| Dynamic secrets | Yes (DB, PKI, AWS, etc.) | No (static values only) |
| Audit log | Structured, queryable, per-request | CloudTrail (good but AWS-specific) |
| Multi-cloud / on-prem | Yes — Vault runs anywhere | AWS-native; cross-cloud means multiple stores |
| Operational overhead | High — HA cluster, unsealing, policy admin | Low — fully managed by AWS |
| Cost | Infrastructure + license (Vault Enterprise) / free OSS | Per secret + per API call |
| Access policy granularity | Fine-grained (path wildcards, entity aliases) | IAM policies (powerful but AWS syntax) |
| Secret versioning | KV v2 supports N versions with rollback | Secrets Manager supports versioning with staging labels |
| Time to production | Days (Vault setup) | Hours (ESO + IAM role) |

The honest senior answer: for most teams on EKS, AWS Secrets Manager + ESO is the right starting point. It's less operational overhead and fully managed. If you later need dynamic DB credentials, multi-cloud portability, or a Vault-style audit log, migrate.

---

## Vault Agent Injector — the alternative to ESO

Worth knowing exists. Instead of ESO creating Kubernetes `Secret` resources, the Vault Agent Injector uses a mutating webhook to inject a Vault Agent sidecar into pods. The sidecar fetches secrets from Vault and writes them to a shared volume that the main container reads.

Trade-off vs ESO:
- ESO: secrets are Kubernetes `Secret` resources (env vars, volume mounts). No sidecar overhead. Secret material passes through the K8s API.
- Vault Agent: secrets go directly from Vault to the pod's filesystem via the sidecar. Never create a Kubernetes `Secret` resource. Better for high-sensitivity secrets where you don't want them in etcd.

In practice, ESO is simpler and the Kubernetes `Secret` in etcd is acceptable for most organizations (etcd is encrypted at rest in EKS). Vault Agent Injector is useful if your compliance requirement is "secret values must never touch the K8s API."

---

## The interview answer in 60 seconds

> "For 50 services across 3 clusters, I'd use Vault as the central source of truth with ESO as the sync mechanism in each cluster.
>
> Vault gives you three things AWS Secrets Manager can't: dynamic short-lived credentials for databases (a Vault DB secrets engine role creates a PostgreSQL user that expires in an hour), a compliance-grade audit log of every secret access, and portability if you're multi-cloud.
>
> Each cluster has ESO installed with K8s auth — Vault trusts the cluster's API server, and ESO authenticates using its ServiceAccount token. No pre-placed credentials needed. Vault policies scope each cluster's ESO to only the paths it needs — staging ESO can't read production secrets, enforced by policy not convention.
>
> `ExternalSecret` resources in each cluster declare what to fetch. For static secrets (API keys, config) they pull from the KV engine. For DB credentials they pull from the DB secrets engine on a short refresh interval.
>
> The operational investment is real — Vault needs HA setup (3 Raft nodes), auto-unseal via cloud KMS, regular snapshot backups, and policy administration. For a team without that capacity, ESO + AWS Secrets Manager is simpler and still very strong. Vault's premium is justified when you need dynamic credentials or you're running multi-cloud."

---

## Self-test drills

### 1. Design a secrets management story for 50 microservices across 3 clusters.

**Reference answer:** Vault + ESO as described above. Key beats: Vault HA (Raft + auto-unseal), K8s auth per cluster, policy per cluster/service, ExternalSecrets for static and dynamic credentials, path namespacing enforced by policy.

### 2. What is the "secret zero" problem and how does K8s auth solve it?

**Reference answer:**
- Secret zero: ESO needs credentials to authenticate to Vault, but those credentials are themselves a secret — where do they come from?
- K8s auth: Vault is configured to trust the K8s API server. ESO presents its ServiceAccount JWT to Vault. Vault calls the K8s API to verify the token's authenticity and identity. No pre-placed credentials needed.
- The ServiceAccount token is already present in every pod (automounted by Kubernetes). K8s auth converts it into a Vault token with scoped permissions. Bootstrap is handled by Kubernetes' own identity system.

### 3. What happens to ESO-managed secrets when Vault is sealed?

**Reference answer:**
- Vault sealed = Vault cannot decrypt any data. All API calls fail with 503.
- ESO's refresh calls fail; `ExternalSecret` status flips to `SecretSyncedError`.
- Existing Kubernetes `Secrets` (last-good values) remain — pods currently running continue to work.
- New pods that mount a Secret that ESO needs to refresh may get stale credentials or fail if the Secret doesn't exist yet.
- Prevention: auto-unseal via cloud KMS means Vault re-unseals automatically after pod restarts. Manual seal should require explicit operator action.
- Runbook: check if Vault pods are up, check if KMS is accessible, check Vault seal status via `vault status`.

### 4. How do dynamic DB credentials work with Vault and ESO?

**Reference answer:**
- Vault DB secrets engine: Vault connects to the DB, creates a short-lived user on demand (running the `creation_statements` SQL).
- When ESO fetches from `database/creds/my-role`, Vault creates a new user and returns credentials with a TTL.
- ESO creates a Kubernetes `Secret` with those credentials. On the next refresh (before TTL expires), ESO fetches again — Vault creates another new user.
- The app must handle credential rotation (restart or re-read the secret). Tools like Reloader can restart pods on Secret changes.
- Security benefit: leaked credentials expire naturally. No manual rotation policy needed for DB passwords.

---

## Further reading / watching

- **HashiCorp Vault docs** (`developer.hashicorp.com/vault`) — Auth methods (Kubernetes, AWS IAM), DB secrets engine, HA Raft configuration.
- **"Seth Vargo — Understanding Vault" talks** — search YouTube for "Seth Vargo Vault." Excellent deep dives on auth, policies, and dynamic secrets.
- **ESO Vault provider docs** (`external-secrets.io/latest/provider/hashicorp-vault/`) — the ClusterSecretStore configuration reference for Vault.
- **Vault Helm chart docs** (`developer.hashicorp.com/vault/docs/platform/k8s/helm`) — production HA setup guide with Raft.
- **"Vault Agent Injector vs External Secrets Operator" comparisons** — several blog posts; Banzai Cloud and Natan Yellin have written good comparisons.

---

## The 4 dimensions (senior framing)

- **Tech**: Vault HA (Raft, 3 nodes), auto-unseal via KMS, K8s auth per cluster, ESO VaultProvider, KV v2 for static secrets, DB secrets engine for dynamic credentials, policy per service, path namespacing as blast-radius control.
- **People**: Vault has a steeper learning curve than AWS Secrets Manager. Policy administration is a dedicated responsibility — nominate a Vault admin on the platform team. New engineers need to understand Vault's path-based model to write ExternalSecret YAMLs. Document: "how to add a secret for a new service" as a runbook. The K8s auth setup is per cluster; document it.
- **CI/CD**: ExternalSecret YAMLs in Git are safe to review (no values). CI pipelines that need secrets (e.g., build-time tokens) can use Vault AppRole or K8s auth from the CI cluster. Policy controls what secrets the CI cluster can access. Kyverno policy: no Kubernetes Secret without an ownerReference to an ExternalSecret (same as the ESO pattern).
- **Operations**: Vault seal status is the critical monitor — alert on `vault.core.unsealed = false`. Snapshot Vault storage (Raft snapshots) on a schedule and test restore quarterly. Audit log — forward to SIEM; alert on `permission denied` spikes (could indicate misconfigured policies or probing). Vault upgrade path: rolling updates via Helm, one node at a time. Test the upgrade in staging; Vault upgrades are generally smooth but Raft requires care.
