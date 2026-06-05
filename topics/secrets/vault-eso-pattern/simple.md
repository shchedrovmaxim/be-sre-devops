# Vault + ESO sync pattern — the simple version (the bank vault)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) fills in the mechanics.

This doc explains **one idea**:

> **Vault is the bank vault: one highly controlled room where all secrets live, with a complete audit log of every access. ESO is the armored car service: it picks up what your app needs and delivers it to the right cluster.**

That's the whole pattern. Everything else — K8s auth, dynamic secrets, the secret zero problem — is just detail on top.

---

## The bank vault + armored car analogy

Imagine a bank vault:
- **Only credentialed parties** can enter (Vault policies + auth methods).
- **Every access is logged** — who entered, when, what they took.
- **Different keys for different rooms** — dev team can enter Room A, prod team Room B.
- **The vault can also mint temporary keys** — a DB vault employee who only works for one hour.

The armored car (ESO):
- You tell it "fetch what's in Room A, locker 3, and deliver it to my cluster."
- It authenticates to the bank, picks up the contents, delivers them.
- Every hour it checks if the contents changed and delivers an updated copy.

| In the bank analogy | In the Vault + ESO world |
|---|---|
| The bank vault building | HashiCorp Vault (or HCP Vault) |
| Vault policies (who can enter which room) | Vault policies (`path "secret/myapp/*" { capabilities = ["read"] }`) |
| The vault's entry log | Vault audit log (every API call, who made it, what path) |
| The temporary one-hour key | Dynamic secrets (Vault creates a short-lived DB user) |
| The armored car + driver | ESO controller in the K8s cluster |
| The car's credentials to enter the bank | K8s auth or IRSA auth on the ESO ServiceAccount |
| The delivery (stocking your fridge) | The Kubernetes `Secret` ESO creates |

---

## The 2-3 sentence summary

HashiCorp Vault is a central secrets store with strong access control, audit logging, and the ability to generate dynamic short-lived credentials (e.g., temporary DB users). ESO syncs from Vault into Kubernetes `Secret` resources using the same `SecretStore` + `ExternalSecret` model it uses for AWS Secrets Manager — but with Vault's `VaultProvider`. The big win over a direct AWS Secrets Manager setup is the audit trail, the dynamic secrets capability, and the ability to work across clouds.

---

## The 2 concepts that trip people up

### 1. The "secret zero" bootstrap problem

ESO needs credentials to authenticate to Vault. But those credentials are themselves a secret. Where do they come from? This is the "secret zero" problem — you need one secret to get all the other secrets.

The practical solutions:
- **K8s auth** (most common on self-hosted Vault): Vault trusts the Kubernetes API server; ESO's ServiceAccount token is the proof of identity. No pre-placed secret needed — the ServiceAccount token is already there.
- **IRSA / AWS IAM** (on EKS with HCP Vault or a Vault on AWS): ESO proves its identity via OIDC, just like with AWS Secrets Manager.
- **AppRole** (older pattern): a role ID + secret ID pre-placed in the cluster. The secret ID is the "secret zero" — it needs to be bootstrapped somehow.

The K8s auth method elegantly solves secret zero for on-cluster ESO: the ServiceAccount token is already there, and Vault is configured to trust the K8s API server.

### 2. Dynamic secrets — the thing that makes Vault uniquely powerful

Vault can create secrets on demand and expire them automatically. The canonical example: instead of giving your app a static database password, Vault creates a **temporary PostgreSQL user** that expires in 1 hour. The app gets a real username + password that works, but they're worthless after they expire.

This means a leaked credential is useless after an hour. You don't need to rotate a static password — Vault just stops renewing the lease. This is fundamentally stronger than any rotation policy on a static secret.

ESO can surface this via the `ExternalSecret` refresh model, but Vault handles the actual create-and-expire lifecycle.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Why Vault instead of AWS Secrets Manager? | Vault adds audit logs, dynamic secrets, fine-grained policies, multi-cloud, and works on-prem |
| What's the "secret zero" problem? | ESO needs credentials to reach Vault — where do those come from? K8s auth solves it by using the ServiceAccount token. |
| What's K8s auth? | Vault trusts the K8s API server. ESO proves its identity using its ServiceAccount token — no pre-placed secret. |
| What's a dynamic secret? | Vault creates a short-lived credential on demand (e.g., a DB user that expires in 1h). Much stronger than static rotation. |
| Is Vault operationally heavy? | Yes. HA setup, unsealing, snapshots, policy management, upgrades — significant platform team investment. |
| When does Vault + ESO beat just ESO + AWS SM? | When you need: on-prem secrets, dynamic DB credentials, multi-cloud, or compliance-grade audit logs. |
| What if Vault is sealed? | All secret reads fail. ESO can't fetch anything. Unsealing is manual (or automated with cloud KMS auto-unseal). |

---

## Self-test (the killer interview question)

Out loud:

> **"Design a secrets management story for 50 microservices across 3 clusters."**

**Reference answer (intuitive version):**

"The pattern I'd reach for is Vault as the central source of truth with ESO as the sync mechanism in each cluster.

Vault handles the 'what secrets exist and who can access them' question. You'd structure Vault paths per service and per environment — something like `secret/production/myapp/db`. Vault policies control which ESO auth identity (K8s auth role or IRSA role) can read which paths. With 50 services you'd probably create one policy per service or per team, not one per secret.

Each cluster has its own ESO installation. Each ESO authenticates to Vault using K8s auth — Vault trusts that cluster's API server, and ESO's ServiceAccount token is the proof of identity. There's no pre-placed credential needed; the ServiceAccount token is the 'secret zero' answer.

In each cluster, you have `ExternalSecret` resources (one per secret per namespace) that say 'fetch from Vault path X and create Kubernetes Secret Y.' ESO keeps them in sync on a refresh interval.

For database credentials specifically, you'd want to use Vault's DB secrets engine: Vault creates short-lived DB users on demand, and ESO fetches a fresh one every refresh interval. No static DB passwords in Secrets Manager at all.

The operational investment is real — Vault itself needs HA setup, auto-unseal, and regular snapshot backups. For a team that already has cloud Secrets Manager, that overhead is only justified if you need dynamic secrets, on-prem deployment, or compliance-grade audit logs. Otherwise, ESO + AWS Secrets Manager is simpler."

---

## Further reading / watching

- **HashiCorp Vault docs** (`developer.hashicorp.com/vault`) — "Getting Started" and the "Kubernetes" auth method are the most relevant sections.
- **"Vault + External Secrets Operator" guide** — search the ESO docs (`external-secrets.io`) for "Vault provider." Has a full setup guide.
- **"Vault dynamic secrets for PostgreSQL"** — HashiCorp's own guide on the DB secrets engine. The hands-on example makes dynamic secrets concrete.
- **"The Secret Zero Problem" by Seth Vargo** (HashiCorp) — blog post or talk. Seth was a Vault core contributor; his explanation of the bootstrap problem and K8s auth solution is the best one.

---

## Next: the deep-dive

When the "Vault as bank vault, ESO as armored car" pattern and the secret-zero concept feel intuitive, jump to [`vault-eso-pattern.md`](./deep-dive.md). The deep-dive covers:

- Vault HA architecture: storage backends (Raft vs Consul), active/standby/perf-replication nodes
- The unsealing lifecycle: Shamir secret sharing, auto-unseal with cloud KMS
- K8s auth mechanics: the full flow from ESO's JWT to a Vault token with a policy
- IRSA auth for EKS (comparing K8s auth vs AWS auth method)
- Dynamic secrets: DB secrets engine — how short-lived users are created and expired
- ESO VaultProvider config with path templating for 50 services
- The comparison to AWS Secrets Manager + ESO: when Vault wins and when it loses
- Vault Agent Injector as an alternative to ESO for in-pod secret injection
