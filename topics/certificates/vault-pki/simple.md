# Vault PKI — the simple version (the company badge office vs the DMV)

> Read this first. Once the in-house badge office analogy clicks, the [deep-dive doc](./deep-dive.md) makes a lot more sense.

One idea:

> **Let's Encrypt is for public-internet certs. Vault PKI is for internal certs — the ones your services use to talk to each other. Using Vault instead of Let's Encrypt gives you certs that live for hours instead of 90 days, under your own CA, without going to the public internet.**

---

## Let's Encrypt is the DMV. Vault is your company's badge office.

| The DMV (Let's Encrypt) | The company badge office (Vault PKI) |
|---|---|
| Issues IDs for public citizens | Issues badges for your own employees |
| Anyone can apply (prove identity) | Only your own systems can request (internal CA) |
| ID is valid for years | Badge is reprinted daily (or hourly) |
| If a badge is stolen, it's valid for years | If a badge is stolen, it expires in hours |
| You go to the DMV, queue, get ID | Your service asks the badge office API, gets a cert in milliseconds |
| You can't operate your own DMV | You run your own Vault cluster |

The insight: for **internal service-to-service communication** (mTLS inside the cluster), you don't need a public CA like Let's Encrypt. You need your own CA that you control, that can issue certs in milliseconds, and that you can configure to expire in hours rather than months.

That's Vault PKI.

---

## Why short-lived certs change everything

A 90-day cert has a 90-day blast radius. If an attacker captures a private key:
- 90-day cert: they have 90 days to use it. You need to revoke it, which requires CRL or OCSP infrastructure, and you need to know you were compromised.
- 1-hour cert: it expires before they can do much. No revocation needed. The cert is essentially self-expiring proof.

This is the "make revocation irrelevant by making expiry fast" strategy. At 1-hour lifetime, a stolen cert is stale by the time an attacker figures out what to do with it.

**The tradeoff**: you need automated renewal. A service that can't auto-renew a 1-hour cert will break after an hour. This is fine — you build renewal into the lifecycle — but you can't issue 1-hour certs to legacy systems that do manual cert management.

---

## The two Vault PKI patterns in Kubernetes

### Pattern 1: cert-manager + Vault Issuer

cert-manager knows how to talk to Vault. You create a `ClusterIssuer` pointing at Vault instead of Let's Encrypt. Your `Certificate` resources look the same; the back-end is different.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-internal-ca
spec:
  vault:
    server: https://vault.internal:8200
    path: pki_int/sign/my-role   # Vault PKI role
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
```

cert-manager handles renewal using the same `renewBefore` logic. Certs can be issued with 24-hour or 8-hour lifetimes; cert-manager renews automatically.

### Pattern 2: Vault Agent Injector

Vault injects certs directly into pods as files via an init container + sidecar. The sidecar watches for expiry and refreshes the cert files. The application reads from the file path — no code changes, no cert-manager needed.

Best for: legacy applications, non-Kubernetes workloads, or when you want cert management completely outside the app code.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Why Vault PKI instead of Let's Encrypt for internal certs? | Let's Encrypt is for public domains. Vault is your own CA — faster, shorter-lived, no external dependency. |
| What is short-lived cert strategy? | Issue certs with 1h–24h lifetime. Stolen keys expire naturally; revocation infrastructure becomes optional. |
| What is the PKI secrets engine? | Vault's built-in CA — can generate root CAs, intermediate CAs, and leaf certs on demand. |
| Root CA vs intermediate CA? | Best practice: root CA is offline/airgapped. Intermediate CA is online and issues leaf certs. Vault hosts the intermediate. |
| What is a Vault PKI role? | A named policy in Vault: "certs issued via this role can only have these domains, max TTL 24h, CN must match this pattern." |
| cert-manager + Vault vs Vault Agent? | cert-manager: works like any other issuer, Kubernetes-native. Agent: sidecar injection, better for legacy apps. |
| What is SPIFFE/SPIRE? | An alternative standard for workload identity (X.509 SVIDs). Senior reference — relevant when you need portable identity across clouds. |
| When would you NOT use Vault PKI? | When you only need public internet certs (use cert-manager + Let's Encrypt). Vault adds operational overhead. |

---

## Self-test (the killer question)

Out loud:

> **"Why might you use Vault PKI for internal mTLS certs instead of cert-manager + Let's Encrypt?"**

**Reference answer (intuitive version):**

"Let's Encrypt is for public-internet domains — it requires proving you own the domain via HTTP or DNS, and it issues 90-day certs. For internal service-to-service mTLS, those constraints don't make sense. You don't want to go to the public internet for internal certs, and 90 days is a long revocation blast radius.

Vault PKI is your own internal CA. You issue certs in milliseconds via the Vault API, you control the CA root, and you can issue certs with 1-hour or 24-hour lifetimes. Short-lived certs make revocation essentially unnecessary — a stolen private key expires before it can be exploited. That simplifies your PKI enormously: no CRL distribution point, no OCSP responder to operate.

The integration pattern: cert-manager has a Vault issuer. You write a `ClusterIssuer` pointing at Vault's PKI secrets engine. Your `Certificate` resources stay the same; cert-manager handles renewal using Vault's signing API. Alternatively, the Vault Agent Injector can push cert files directly into pods as a sidecar.

The senior caveat: Vault adds operational overhead. If you don't already run Vault, deploying and operating it for PKI alone is a significant investment. For small teams or simple use cases, a self-signed ClusterIssuer in cert-manager might be the simpler starting point."

---

## Further reading / watching

- **Vault PKI secrets engine** — [developer.hashicorp.com/vault/docs/secrets/pki](https://developer.hashicorp.com/vault/docs/secrets/pki). The official docs. Start with the "Getting Started" tutorial.
- **cert-manager Vault issuer** — [cert-manager.io/docs/configuration/vault](https://cert-manager.io/docs/configuration/vault/). Shows the exact `ClusterIssuer` configuration and Vault auth methods.
- **SPIFFE/SPIRE** — [spiffe.io](https://spiffe.io/). The workload identity standard. Relevant when you need portable identity across multiple cloud providers or clusters. Senior reference.
- **"PKI Best Practices"** — search "Vault PKI best practices HashiCorp." The "Root CA offline, intermediate online" pattern is the key recommendation.

---

## Next: the deep-dive

When the badge-office analogy is clear and you can name the two K8s integration patterns without hesitation, jump to [`vault-pki.md`](./deep-dive.md). The deep-dive covers:

- The PKI secrets engine setup: root CA → intermediate CA → leaf cert flow
- Vault PKI roles — the exact policy levers (allowed_domains, max_ttl, require_cn, etc.)
- cert-manager Vault Issuer — the exact YAML and Vault auth configuration
- Vault Agent Injector — the annotation-based injection pattern
- Revocation strategy: CRL vs OCSP vs "short-lived is enough"
- SPIFFE/SPIRE as the senior-level alternative
- The 4-dimensions senior framing
