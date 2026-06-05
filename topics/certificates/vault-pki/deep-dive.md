# Vault PKI — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why might you use Vault PKI for internal mTLS certs instead of cert-manager + Let's Encrypt?"** — explaining the PKI secrets engine, root + intermediate CA setup, short-lived cert strategy, cert-manager and Vault Agent integration, revocation approaches, and the SPIFFE/SPIRE alternative as a senior reference.

> Start with [vault-pki-simple.md](./simple.md) if you haven't read it. The badge-office analogy is the spine of everything here.

---

## The senior framing — when Let's Encrypt is the wrong tool

Let's Encrypt is designed for one thing: domain-validated certs for public HTTPS services. It works brilliantly for that. It's not designed for:

- **Internal service-to-service mTLS** where the "domain" is `payment-service.cluster.local` — you can't prove you own that to a public CA.
- **Short-lived certs** — Let's Encrypt minimum is 60 days (practical minimum); it won't issue 1-hour certs.
- **Air-gapped environments** — Let's Encrypt requires outbound internet access for challenge verification.
- **Client certs for mTLS** — Let's Encrypt only issues server certs (domain validation), not client certs.

For internal PKI, you need to operate your own CA. Vault's PKI secrets engine is the standard way to do this in modern infrastructure.

---

## The PKI secrets engine — what it is

Vault's PKI secrets engine is a built-in CA. When enabled, it can:
- Generate and store a root CA (self-signed) or import an existing root
- Generate intermediate CAs and sign them with the root
- Issue leaf certificates on demand via API
- Set policies on what certificates can be issued (allowed domains, max TTL, key type)
- Optionally publish CRL (Certificate Revocation Lists) and OCSP endpoints

The key word is "on demand via API." Your service asks Vault: "give me a cert for `payment-service.internal`." Vault responds in milliseconds with a signed cert and private key. No human involvement, no queuing, no ACME challenge.

### Setup: root CA → intermediate CA → leaf certs

Best practice is a two-tier hierarchy:

```
Root CA (offline, airgapped)
└── Intermediate CA (online, Vault)
    ├── payment-service.internal (1h TTL)
    ├── order-service.internal (1h TTL)
    └── user-service.internal (1h TTL)
```

**Why two tiers?**
- The root CA's private key never needs to be online. It signs only the intermediate CA cert (a rare operation). If Vault is compromised, the attacker gets the intermediate — serious, but you can revoke it with a new intermediate signed by the (still-safe) root.
- Single-tier (root = online) means the root key is online. A Vault compromise means the attacker gets the root. Game over — you have to distribute a new root to all trust stores.

**Vault commands (simplified flow)**:
```bash
# 1. Enable the PKI engine
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki    # 10 years for root

# 2. Generate root CA (or import if you have an existing offline root)
vault write pki/root/generate/internal \
  common_name="Example Corp Root CA" \
  ttl=87600h

# 3. Enable intermediate PKI engine
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int  # 5 years for intermediate

# 4. Generate intermediate CSR and sign with root
vault write pki_int/intermediate/generate/internal \
  common_name="Example Corp Intermediate CA"
# Take the CSR, sign it with pki root
vault write pki/root/sign-intermediate \
  csr="<from above>" ttl=43800h
# Import signed cert back into pki_int
vault write pki_int/intermediate/set-signed certificate="<signed cert>"

# 5. Create a role defining what certs this engine can issue
vault write pki_int/roles/my-role \
  allowed_domains="internal,svc.cluster.local" \
  allow_subdomains=true \
  max_ttl=24h \
  generate_lease=true
```

### PKI roles — the policy layer

A role is the policy object that controls what certificates can be issued via a given path:

| Role field | What it controls |
|---|---|
| `allowed_domains` | Domains the cert may include as SAN/CN |
| `allow_subdomains` | Whether subdomains of `allowed_domains` are OK |
| `max_ttl` | Maximum TTL for certs issued via this role |
| `allow_wildcard_certificates` | Explicit opt-in for wildcard SANs |
| `require_cn` | Force a specific CN format |
| `key_type` / `key_bits` | RSA vs ECDSA and key size |
| `ou`, `organization` | Force specific OU/Org in the cert |

Roles make Vault PKI multi-tenant: different teams get different Vault policies that allow them to request certs from specific roles. The payment team can request `*.payment.internal`; the platform team can request any `*.internal`.

---

## Short-lived cert strategy — why it changes the security posture

Standard PKI thinking:
- Issue certs with long TTLs (1 year).
- Maintain a CRL (revocation list) for compromised certs.
- Publish CRL at a well-known URL; clients check it.

The problem: CRL maintenance is operational overhead. CRL fetching adds latency. CRL distribution is another thing that can break. And a 1-year cert that leaks is valid for up to a year.

Short-lived cert thinking:
- Issue certs with 1-hour TTL.
- Never maintain a CRL.
- Rotation is built into the workload lifecycle.

If a private key leaks with a 1-hour cert: the attacker has a cert that expires in < 60 minutes on average (assuming the cert was halfway through its life). By the time they understand what they have, it's expired.

**This isn't just theory.** Netflix's BLESS (internal SSH CA), Square's certificate tooling, and many large-scale service meshes use this approach. The assumption: "make the credential worthless by the time it's stolen, rather than trying to revoke it after the fact."

**The tradeoff**: workloads must auto-renew. Every service must be able to request a new cert, receive it, and reload (or continue serving) without downtime, every hour. This is non-trivial for legacy applications that do cert management at startup only.

Practical TTL guidance:
- Internal mTLS service certs: 24h–72h is a good starting point. 1h is aggressive but achievable with cert-manager.
- Short-lived SSH certs (BLESS pattern): 1h–8h is common.
- External-facing certs: still Let's Encrypt (90 days), don't use Vault for these.

---

## Integration pattern 1: cert-manager + Vault Issuer

cert-manager has a built-in Vault issuer. Your `Certificate` resources look exactly the same; the back-end is Vault instead of Let's Encrypt.

### Vault auth for cert-manager

cert-manager needs to authenticate to Vault to request certs. The Kubernetes auth method is the standard pattern in K8s:

```bash
# In Vault: enable Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl cluster-info | ...)" \
  kubernetes_ca_cert="@/path/to/ca.crt" \
  token_reviewer_jwt="$(kubectl get secret -n cert-manager ... -o json | ...)"

# Create a Vault policy allowing cert-manager to sign certs
vault policy write cert-manager - <<EOF
path "pki_int/sign/my-role" {
  capabilities = ["create", "update"]
}
EOF

# Create a Vault role for cert-manager's ServiceAccount
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=cert-manager \
  bound_service_account_namespaces=cert-manager \
  policies=cert-manager \
  ttl=1h
```

### The ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-internal-ca
spec:
  vault:
    server: https://vault.vault.svc.cluster.local:8200
    path: pki_int/sign/my-role
    caBundle: <base64-encoded-vault-ca-cert>   # if Vault uses a self-signed cert
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
        secretRef:                              # ServiceAccount token
          name: cert-manager-vault-token
          key: token
```

### The Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-service-tls
  namespace: payments
spec:
  secretName: payment-service-tls
  issuerRef:
    name: vault-internal-ca
    kind: ClusterIssuer
  dnsNames:
    - payment-service.payments.svc.cluster.local
  duration: 24h       # Vault PKI can do this; Let's Encrypt can't
  renewBefore: 8h     # Renew 8 hours before expiry
```

cert-manager handles renewal automatically using the same machinery as with Let's Encrypt. The short `duration` just means renewal happens more frequently.

---

## Integration pattern 2: Vault Agent Injector

Vault Agent is a sidecar that handles Vault authentication and secret injection into pods. For PKI, it:
1. Authenticates to Vault using the pod's ServiceAccount token.
2. Requests a cert from the PKI secrets engine.
3. Writes the cert and key to a file in a shared volume.
4. Watches for expiry and renews the cert automatically (writing updated files).

The application reads from a file path — no code changes, no cert-manager dependency.

**Annotation-driven injection**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "payment-service"
        vault.hashicorp.com/agent-inject-secret-tls.crt: "pki_int/issue/my-role"
        vault.hashicorp.com/agent-inject-template-tls.crt: |
          {{- with pkiCert "pki_int/issue/my-role" "common_name=payment-service.payments.svc.cluster.local" "ttl=24h" -}}
          {{ .Cert }}
          {{- end }}
        vault.hashicorp.com/agent-inject-secret-tls.key: "pki_int/issue/my-role"
        vault.hashicorp.com/agent-inject-template-tls.key: |
          {{- with pkiCert "pki_int/issue/my-role" "common_name=payment-service.payments.svc.cluster.local" "ttl=24h" -}}
          {{ .Key }}
          {{- end }}
```

Vault Agent writes the cert to `/vault/secrets/tls.crt` and the key to `/vault/secrets/tls.key`. Your application reads from there.

**The rotation problem with Agent Injector**: the Vault Agent sidecar renews the cert and rewrites the files. But your application won't automatically pick up the new cert files unless it re-reads them. Solutions:
- Use a library that watches the file path and reloads (many modern TLS libraries support this).
- Use a `SIGHUP` handler in your application to reload TLS config.
- Use short enough TTLs that a service restart (on deploy) is sufficient.

---

## Revocation strategies

### Option 1: CRL (Certificate Revocation List)

Vault publishes a CRL at `<vault-address>/v1/pki_int/crl`. Clients are configured to fetch this list and reject revoked certs.

**Problems**: CRL must be fetched periodically (adds latency to TLS handshakes); CRL must be reachable from clients; a client with a cached CRL won't know about new revocations until the cache expires; CRL grows over time.

### Option 2: OCSP (Online Certificate Status Protocol)

Vault can serve OCSP responses. Clients query the OCSP responder per-handshake to check if a specific cert is revoked.

**Problems**: adds a network round-trip to every TLS handshake (mitigated by OCSP stapling); OCSP responder is another service to operate and keep highly available.

### Option 3: Short-lived certs (the preferred approach)

Issue certs with TTLs short enough that revocation doesn't matter. A cert that expires in 1 hour doesn't need a CRL entry — it's already expired by the time anyone notices it was compromised.

This is the approach most teams adopt when they embrace Vault PKI seriously. It simplifies operations significantly: no CRL distribution infrastructure, no OCSP responder, no client configuration for CRL/OCSP endpoints.

**When CRL/OCSP is still needed**: if you have certs with long TTLs (you issued a 1-year cert to a device you can't rotate easily), short-lived strategy doesn't apply. In that case, Vault's CRL/OCSP functionality is important.

---

## SPIFFE/SPIRE — the senior-level alternative

SPIFFE (Secure Production Identity Framework For Everyone) and SPIRE (the reference implementation) solve workload identity at a layer above raw PKI.

**The concept**: instead of "cert for `payment-service.payments.svc.cluster.local`," SPIFFE issues identities called SVIDs (SPIFFE Verifiable Identity Documents) in the form `spiffe://trust-domain/path`. For example: `spiffe://example.com/ns/payments/sa/payment-service`.

SVIDs are encoded as X.509 certs (X.509 SVID) or JWT tokens. The key difference from vanilla PKI:
- Identity is tied to the **workload's runtime attributes** (namespace, ServiceAccount, node identity), not just a domain name.
- SPIRE's node attestation verifies the workload is running where it claims (checking the K8s API, AWS instance identity documents, etc.) — not just whether someone has a private key.
- Works **across clusters and clouds** — the same identity standard works on K8s, VMs, AWS, GCP.

**When to mention SPIFFE/SPIRE in an interview**:
- When asked about cross-cluster mTLS (Istio with SPIFFE-compatible identity, or multi-cluster meshes).
- When asked about zero-trust architectures where workload identity needs to be portable.
- As the "senior reference" that shows you know there's a layer above cert-manager.

**cert-manager and SPIFFE**: cert-manager can issue SPIFFE-format SVIDs via its SPIFFE URI SAN support. Combining cert-manager (for issuance + renewal) with SPIFFE URI identities gives you SPIFFE-compatible workload identity without a full SPIRE deployment.

```yaml
# Certificate with SPIFFE URI SAN
spec:
  uriSANs:
    - "spiffe://example.com/ns/payments/sa/payment-service"
```

---

## The interview answer in 60 seconds

> "Let's Encrypt is for public-internet certs — it needs domain validation and issues 90-day certs. For internal mTLS between services in a cluster, you don't want to go to a public CA: the 'domains' are internal hostnames, you want much shorter TTLs, and you want your own CA root.
>
> Vault's PKI secrets engine is your own internal CA. The standard setup is two-tier: an offline root CA signs an online intermediate CA hosted in Vault. Vault issues leaf certs on demand via API. TTLs can be hours or days — cert-manager with a Vault Issuer renews automatically, so short TTLs are fine.
>
> The short-lived cert strategy eliminates revocation complexity: a 1-hour cert that leaks is worthless by the time it's exploited. No CRL distribution infrastructure needed.
>
> Two K8s integration patterns: cert-manager with a Vault Issuer (looks like any other Issuer, Kubernetes-native, standard `Certificate` CRDs), or Vault Agent Injector (sidecar pushes cert files into pods via annotations, good for legacy apps). For auth from K8s, the Kubernetes auth method (JWT-based) is the standard.
>
> Senior reference: SPIFFE/SPIRE is the next level — workload identity tied to runtime attributes, portable across clouds, SPIFFE SVIDs instead of domain-name certs."

---

## Self-test drills

### 1. Why might you use Vault PKI for internal mTLS certs instead of cert-manager + Let's Encrypt?

**Reference answer:**
- Let's Encrypt requires public domain validation (HTTP-01 or DNS-01) — not applicable to internal hostnames like `payment-service.svc.cluster.local`.
- Let's Encrypt minimum cert duration is ~60 days; Vault can issue 1-hour certs for internal services.
- Let's Encrypt requires outbound internet access; Vault is air-gap-compatible.
- Let's Encrypt doesn't issue client certs for mTLS; Vault PKI can issue both client and server certs.
- Short-lived Vault certs eliminate the need for CRL/OCSP revocation infrastructure — a stolen cert expires before it can be misused.

### 2. Walk me through the two-tier CA setup in Vault.

**Reference answer:**
1. Enable PKI engine at `/pki`, generate a root CA (or import an offline root cert), set max TTL to 10 years.
2. Enable a second PKI engine at `/pki_int` for the intermediate.
3. Generate an intermediate CSR from `/pki_int`; sign it with the root CA at `/pki/root/sign-intermediate`.
4. Import the signed intermediate back into `/pki_int`.
5. Create a role at `/pki_int/roles/<name>` defining allowed domains, max TTL, key type.
6. Configure cert-manager ClusterIssuer or Vault Agent to call `/pki_int/sign/<role>` for leaf cert issuance.
7. Why two tiers: root CA can be kept offline. If the intermediate is compromised, revoke it and create a new one. Root CA key stays safe.

### 3. What are the two Kubernetes integration patterns for Vault PKI?

**Reference answer:**
- **cert-manager + Vault Issuer**: create a `ClusterIssuer` pointing at Vault's PKI sign endpoint. `Certificate` resources work as normal. cert-manager authenticates to Vault via the Kubernetes auth method (ServiceAccount JWT). Best for teams already using cert-manager; keeps cert management Kubernetes-native.
- **Vault Agent Injector**: annotate pods with Vault injection annotations. Vault's mutating webhook injects an init container (one-time fetch) and a sidecar (watches for expiry, renews). Cert files appear at `/vault/secrets/` in the pod. Best for legacy apps or when you want cert management entirely outside application code.

### 4. When would short-lived certs not be enough, and what revocation strategy would you use?

**Reference answer:**
- Short-lived certs don't work for **long-TTL certs** issued to devices you can't auto-rotate (IoT devices, hardware tokens, enterprise clients with manual cert processes).
- They also don't work when you need **immediate revocation** — e.g., an employee's client cert after they leave the company, if the cert has a 1-year TTL.
- In those cases: **OCSP stapling** is the preferred modern approach. The server fetches a signed OCSP response from Vault's OCSP responder and includes it in the TLS handshake. Clients don't need to query the OCSP endpoint per-request — the stapled response provides freshness. Avoids the privacy issues and performance overhead of client-side OCSP.
- CRL is the fallback for clients that don't support OCSP. Vault publishes a CRL; configure clients to fetch it on a schedule.

---

## Further reading / watching

- **Vault PKI secrets engine** — [developer.hashicorp.com/vault/docs/secrets/pki](https://developer.hashicorp.com/vault/docs/secrets/pki). The authoritative reference. Start with the "Getting Started" tutorial — it walks through the two-tier setup step by step.
- **cert-manager Vault issuer** — [cert-manager.io/docs/configuration/vault](https://cert-manager.io/docs/configuration/vault/). Shows ClusterIssuer config and Kubernetes auth setup.
- **Vault Agent Sidecar Injector** — [developer.hashicorp.com/vault/docs/platform/k8s/injector](https://developer.hashicorp.com/vault/docs/platform/k8s/injector). The annotation-based injection docs.
- **SPIFFE specification** — [spiffe.io/docs/latest/spiffe-about/overview](https://spiffe.io/docs/latest/spiffe-about/overview/). The workload identity concept and SVID format.
- **SPIRE** — [spiffe.io/docs/latest/deploying/getting_started_k8s](https://spiffe.io/docs/latest/deploying/getting_started_k8s/). SPIRE on Kubernetes getting started guide.
- **Netflix BLESS** — search "Netflix BLESS SSH CA" for the blog post describing short-lived SSH certificate approach (the same philosophy applied to SSH).

---

## The 4 dimensions (senior framing)

- **Tech**: two-tier PKI (offline root + online intermediate in Vault); PKI roles as the policy layer; short-lived certs (1h–24h) eliminate revocation complexity; cert-manager Vault Issuer vs Vault Agent Injector; OCSP stapling for long-lived cert revocation; SPIFFE/SPIRE for portable workload identity.
- **People**: Vault adds significant operational overhead — Vault HA setup, unsealing, audit logging, policy management. If the team doesn't already operate Vault, deploying it for PKI alone is a big investment. Evaluate: is a self-signed CA ClusterIssuer in cert-manager sufficient for your internal mTLS needs? Only bring in Vault when you need the policy granularity, short-TTL issuing, or multi-cluster/multi-cloud identity. Document which roles exist and what they allow — PKI roles are invisible until someone hits an allowed_domains rejection.
- **CI/CD**: Vault PKI roles and policies should be in Terraform (HashiCorp Vault provider). The two-tier CA hierarchy setup is a one-time Terraform apply. cert-manager ClusterIssuer pointing at Vault is in your cluster's GitOps repo. The root CA cert (for trust distribution) needs to be in your base container images or a trusted CA bundle Secret — manage this via Terraform + a CI pipeline that updates the bundle.
- **Operations**: monitor Vault's PKI engine: cert issuance rate (volume), failed sign requests, cert expiry distribution (are short-lived certs being rotated as expected?). Alert on: Vault PKI endpoint unreachable for > 5 minutes (cert rotation will start failing within the `renewBefore` window). Alert on: cert-manager `Certificate` NotReady for > 15 minutes when using Vault issuer. Runbook for "Vault PKI down": short-lived certs will start expiring; need to restore Vault or failover. This is why you want `renewBefore` > your expected Vault recovery time.
