# cert-manager CRDs — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk through every cert-manager CRD created when I request a Certificate. What does each one do?"** — naming all five CRDs, their relationships, the controller that handles each, the resulting Secret, renewal mechanics, and how to debug a stuck cert in under 5 minutes.

> Start with [cert-manager-simple.md](./simple.md) if you haven't read it. The paperwork assembly-line analogy is the spine of everything here.

---

## The senior framing — CRDs as an audit trail

cert-manager's design choice: one Kubernetes object per step in the cert lifecycle. This looks like over-engineering until your cert is stuck. Then you can `kubectl describe` each object to see exactly which step failed and why.

The alternative (opaque "give me a cert" controller with no intermediate objects) would be faster to write and simpler to understand until it breaks. cert-manager chose observability over simplicity, which is the right call for infrastructure that will be debugged at 2 AM.

---

## The five CRDs and the controller dance

### 1. `ClusterIssuer` / `Issuer`

**What it represents**: a configured connection to a CA. "When a cert is requested, use these credentials and this protocol to get it signed."

**Namespace scope**: `Issuer` is namespace-scoped (can only sign certs in the same namespace). `ClusterIssuer` works across all namespaces. In practice, almost every platform team uses `ClusterIssuer` — it avoids duplicating Issuer config in every namespace.

**What's inside it**:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key  # the ACME account key; not the cert key
    solvers:
      - selector: {}   # applies to all Certificate requests
        http01:
          ingress:
            class: nginx
```

The `privateKeySecretRef` stores the ACME account key. This is not the certificate's private key. It's the key that identifies this cert-manager instance as an ACME account holder. cert-manager generates this automatically on first use if the Secret doesn't exist.

**Other issuer types** (the same CRD, different `spec`):
- `selfSigned`: generates a self-signed cert. Used for bootstrapping CA certs (not for end-user certs).
- `ca`: signs certs using a CA cert stored in a K8s Secret. Good for internal CAs that don't have an ACME API.
- `vault`: signs via Vault PKI secrets engine. See [vault-pki.md](../vault-pki/deep-dive.md).
- `venafi`: for enterprise CA integration.

**Controller**: the issuer controller watches `Issuer` and `ClusterIssuer` objects and validates that the configuration is reachable. It sets a `Ready` condition.

### 2. `Certificate`

**What it represents**: your request. "I want a cert with these SANs, signed by this issuer, stored in this Secret, renewed this far before expiry."

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: production
spec:
  secretName: api-tls          # where the cert + key will be stored
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
    - www.api.example.com
  duration: 2160h              # 90 days (Let's Encrypt's enforced maximum)
  renewBefore: 720h            # renew 30 days before expiry
  privateKey:
    algorithm: ECDSA           # default RSA; ECDSA gives smaller certs + faster handshakes
    size: 256
  usages:
    - server auth
    - client auth              # include only if you need this cert for mTLS client auth
```

**Controller**: the certificate controller watches `Certificate` objects. When a `Certificate` is Created/Updated, the controller:
1. Checks if a valid cert already exists in `secretName` with sufficient remaining lifetime.
2. If not (new cert, expired cert, or within `renewBefore` window): creates a `CertificateRequest`.

The certificate controller also watches the `Secret` named in `secretName`. If the Secret is deleted externally, the controller re-triggers issuance.

**Renewal trigger**: cert-manager evaluates `renewBefore` continuously. When `now + renewBefore >= cert.notAfter`, renewal starts. Default `renewBefore` if not set: 1/3 of `duration`. For a 90-day cert with no `renewBefore` set, renewal starts at day 60.

### 3. `CertificateRequest`

**What it represents**: an internal request object containing the CSR (Certificate Signing Request). Rarely written by hand — cert-manager creates it from a `Certificate`. But you do interact with it during debugging.

**What's inside it**:

```yaml
apiVersion: cert-manager.io/v1
kind: CertificateRequest
metadata:
  name: api-tls-abc123         # auto-generated name
  namespace: production
spec:
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  request: <base64-DER-encoded-CSR>   # the actual CSR
  duration: 2160h
```

The CSR contains the public key (from the freshly generated cert keypair) and the requested SANs. The private key is generated by cert-manager and immediately stored in the `secretName` Secret — it never appears in the `CertificateRequest` object.

**Approval workflow** (cert-manager v1.3+): by default, cert-manager auto-approves its own `CertificateRequest` objects. Enterprises use the approval API to route requests through human or policy review (e.g., require approval for wildcard certs or long-duration certs). Controllers like [approver-policy](https://github.com/cert-manager/approver-policy) plug into this.

**Controller**: the ACME issuer controller (or CA issuer controller, or Vault issuer controller) watches `CertificateRequest` objects and attempts to fulfill them by contacting the CA.

### 4. `Order` (ACME only)

**What it represents**: a Let's Encrypt ACME order. Created by the ACME issuer controller when it processes an ACME `CertificateRequest`. Not created for non-ACME issuers (Vault, CA, self-signed).

**State machine**:
```
pending → ready → processing → valid → (cert downloaded)
                             → invalid → (order failed)
```

An `Order` in `pending` state means Let's Encrypt hasn't confirmed domain control yet. `ready` means all challenges passed and the CSR can be submitted. `valid` means the cert was issued. `invalid` means a challenge failed — look at the associated `Challenge` objects for why.

**One order per `CertificateRequest`.** One order covers all SANs in the cert (not one order per SAN). But one `Challenge` per SAN.

### 5. `Challenge` (ACME only)

**What it represents**: the actual domain verification proof — one per SAN in the cert. This is where certs get stuck most often.

```yaml
# Example stuck Challenge
apiVersion: acme.cert-manager.io/v1
kind: Challenge
metadata:
  name: api-tls-abc123-123456789
  namespace: production
status:
  type: http-01
  token: "SxVzN8aKQCqzqoaLMl9a7..."
  key: "SxVzN8aKQCqzqoaLMl9a7....<account-key-thumbprint>"
  state: pending        # stuck here = Let's Encrypt can't reach the token URL
  reason: "Waiting for HTTP-01 challenge to complete"
```

**States**:
- `pending` → waiting for the solver to set up the token
- `processing` → cert-manager is actively placing the token; Let's Encrypt is checking
- `valid` → Let's Encrypt confirmed the token; challenge passed
- `invalid` → Let's Encrypt checked but didn't find the token; challenge failed

**What cert-manager does for HTTP-01**:
1. Creates a temporary `Pod` (the solver pod) and a `Service` in the cert's namespace.
2. Creates a temporary `Ingress` rule routing `/.well-known/acme-challenge/<token>` to that pod.
3. Notifies Let's Encrypt to check.
4. After the check (valid or invalid), cleans up the pod, service, and ingress.

**What cert-manager does for DNS-01**:
1. Calls the DNS provider's API to create `_acme-challenge.<domain> TXT <key>`.
2. Optionally waits for DNS propagation (the `dnsPropagationPolicy` field: `Sync` waits; `None` doesn't).
3. Notifies Let's Encrypt.
4. Cleans up the TXT record after the check.

---

## The resulting Secret

After a successful issuance, cert-manager stores three items in the Secret named in `Certificate.spec.secretName`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-tls
  namespace: production
  annotations:
    cert-manager.io/certificate-name: api-tls
    cert-manager.io/issuer-name: letsencrypt-prod
    cert-manager.io/issuer-kind: ClusterIssuer
type: kubernetes.io/tls
data:
  tls.crt: <base64 PEM chain: leaf cert + intermediate CA cert>
  tls.key: <base64 PEM private key>
  ca.crt:  <base64 PEM of the issuing CA cert>  # not always present; depends on issuer type
```

**How pods consume it** (Nginx example):
```yaml
volumes:
  - name: tls
    secret:
      secretName: api-tls
containers:
  - name: nginx
    volumeMounts:
      - name: tls
        mountPath: /etc/nginx/ssl
        readOnly: true
```

Nginx reads `tls.crt` and `tls.key` from the mount path. When cert-manager rotates the cert, the Secret is updated. Pods that mount the Secret as a volume get the new cert files (kubelet syncs volume contents) — but Nginx won't pick them up until it reloads. The common pattern:

```yaml
# annotation on the Deployment to restart on cert renewal
cert-manager.io/inject-ca-from: ...   # not for this, but shows the annotation pattern
```

**Better pattern**: use `reloader` (Stakater Reloader or similar) — watches Secrets for changes and triggers `Deployment` rollouts automatically.

---

## Renewal mechanics in detail

cert-manager has a renewal loop that runs every 10 minutes (configurable). It evaluates every `Certificate` object:

```
for each Certificate:
  cert = getCertFromSecret(secretName)
  if cert doesn't exist OR cert is expired:
    trigger immediate issuance
  elif now + renewBefore >= cert.notAfter:
    trigger renewal
  else:
    log "cert is healthy, next check in N minutes"
```

During renewal:
1. cert-manager creates a new `CertificateRequest` (not a new `Certificate` — the `Certificate` resource is reused).
2. If ACME: creates a new `Order` and `Challenges`.
3. When the new cert arrives, the old `secretName` Secret is updated atomically.
4. The old cert remains valid until its `notAfter` date — there's no revocation on renewal.

**What `cmctl renew` does**: forces immediate creation of a new `CertificateRequest` regardless of the renewal window. Useful when you want to rotate a cert ahead of schedule (e.g., after a key compromise). Does not delete the old cert.

**What happens on renewal failure**: cert-manager retries with exponential backoff (starting at 1 minute, capping at 32 minutes by default). The old cert continues serving until it actually expires. Renewal failure is not an immediate outage — you have the `renewBefore` window (30 days for a 90-day cert) to fix the problem.

---

## Debugging a stuck cert

This is the 5-minute drill. In order:

```bash
# Step 1: top-level status — where is it stuck?
cmctl status certificate api-tls -n production
# Output shows: Ready: False, reason: "Issuing", points to CertificateRequest

# Step 2: Certificate events — any obvious errors?
kubectl describe certificate api-tls -n production
# Look at Events section

# Step 3: CertificateRequest — what did the ACME issuer say?
kubectl get certificaterequest -n production
kubectl describe certificaterequest api-tls-<hash> -n production
# Look for "Certificate request has been denied" or other errors

# Step 4: Order — what did Let's Encrypt say?
kubectl get order -n production
kubectl describe order -n production
# Look for "Waiting for http-01 challenge propagation" or "order is in 'invalid' state"

# Step 5: Challenge — the actual error
kubectl get challenge -n production
kubectl describe challenge -n production
# This is almost always where the real error is:
# "Error getting HTTP validation: Get 'http://api.example.com/.well-known/acme-challenge/...': dial tcp: connection refused"

# Step 6: cert-manager controller logs (if object events aren't enough)
kubectl logs -n cert-manager deploy/cert-manager --since=10m
```

**Common Challenge failure causes and fixes**:

| Error in Challenge events | Cause | Fix |
|---|---|---|
| `connection refused` / `connection timed out` | Port 80 not reachable from Let's Encrypt | Check ingress controller is listening on port 80; check LB security groups |
| `NXDOMAIN` / DNS-01 TXT not found | DNS propagation lag or wrong credentials | Check DNS provider API keys; increase DNS TTL waiting; check `dnsPropagationPolicy` |
| `403` on `/.well-known/acme-challenge/` | Ingress is blocking or redirecting before challenge can be served | Check ingress annotations; some ingress configs redirect HTTP → HTTPS before cert-manager can serve the token |
| `too many certificates already issued` | Rate limit hit | Switch to staging; wait for the week to reset; request rate limit increase |
| `CAA record does not allow` | DNS CAA record blocks Let's Encrypt | Add `letsencrypt.org` to your CAA record: `example.com CAA 0 issue "letsencrypt.org"` |

---

## The interview answer in 60 seconds

> "When you create a `Certificate` resource, cert-manager's certificate controller sees it and checks whether the Secret named in `secretName` already has a valid cert. If not, it creates a `CertificateRequest` containing the CSR — cert-manager generates a fresh private key and puts only the public key in the CSR.
>
> If the issuer is ACME (Let's Encrypt), the ACME issuer controller picks up the `CertificateRequest` and creates an `Order` with Let's Encrypt — one per cert request. The order spawns one `Challenge` per domain name. cert-manager then places the challenge token (HTTP-01: temp pod + ingress; DNS-01: TXT record via provider API). Let's Encrypt checks it. When all challenges are valid, the order is finalized, Let's Encrypt signs the cert, and cert-manager stores it in the `secretName` Secret.
>
> Renewal is automatic: cert-manager polls every 10 minutes; when remaining lifetime hits `renewBefore`, it creates a new `CertificateRequest` and repeats the flow. The old cert stays valid until expiry.
>
> For debugging: `cmctl status certificate` first, then drill down to `Order`, then `Challenge` events. 90% of failures are at the `Challenge` level — wrong ingress config or wrong DNS credentials."

---

## Self-test drills

### 1. Walk through every cert-manager CRD created when I request a Certificate.

**Reference answer:**
- `Certificate` — you create this. Declares what you want.
- `CertificateRequest` — cert-manager creates this. Contains the CSR; triggers the issuer.
- `Order` (ACME only) — cert-manager creates this. The Let's Encrypt ACME order negotiation.
- `Challenge` (ACME only) — one per domain name in the cert. The actual domain verification proof.
- Result: a `Secret` with `tls.crt` (PEM chain), `tls.key` (private key), and `ca.crt`.

### 2. What's the difference between `Issuer` and `ClusterIssuer`?

**Reference answer:**
- Both configure how to get certs. The difference is namespace scope.
- `Issuer` is namespace-scoped: can only sign `Certificate` objects in the same namespace.
- `ClusterIssuer` is cluster-scoped: can sign `Certificate` objects in any namespace.
- Production pattern: one `ClusterIssuer` for the whole cluster. Namespace-scoped `Issuer` is useful for teams that want full control over their own cert config (e.g., different DNS providers per team).

### 3. How does cert-manager handle renewal, and what happens if renewal fails?

**Reference answer:**
- cert-manager polls every 10 minutes. When remaining lifetime < `renewBefore` (default 1/3 of duration), it creates a new `CertificateRequest`.
- The existing cert in the Secret remains valid throughout the renewal process. No downtime from renewal.
- If renewal fails (ACME challenge fails, Let's Encrypt is down), cert-manager retries with exponential backoff. The old cert continues to serve.
- If renewal fails for the entire `renewBefore` window (30 days for a 90-day cert), the cert will expire and cause an outage. This is why 30 days is the right `renewBefore` — 30 days to notice and fix.
- Alert on: `Certificate` NotReady for more than a few hours; `Order` stuck in `pending` state.

### 4. A cert is stuck. Walk me through your debugging process.

**Reference answer:**
1. `cmctl status certificate <name>` — get the high-level state and which object is failing.
2. `kubectl describe certificate <name>` — check Certificate events for any obvious message.
3. `kubectl describe certificaterequest <name>` — if the CertificateRequest was denied or has an error from the issuer.
4. `kubectl describe order <name>` — if it's ACME: what state is the order in, and why?
5. `kubectl describe challenge <name>` — almost always where the real error is: port 80 not reachable, DNS credentials wrong, CAA record blocking, rate limit.
6. `kubectl logs -n cert-manager deploy/cert-manager` — controller logs if the object events aren't enough.

---

## Further reading / watching

- **cert-manager architecture docs** — [cert-manager.io/docs/concepts](https://cert-manager.io/docs/concepts/). The "Certificate Lifecycle" diagram is the clearest visual of the controller dance.
- **`cmctl` CLI** — [cert-manager.io/docs/reference/cmctl](https://cert-manager.io/docs/reference/cmctl/). `cmctl status certificate` and `cmctl renew` are the two commands you'll use most.
- **cert-manager Prometheus metrics** — [cert-manager.io/docs/observability/metrics](https://cert-manager.io/docs/observability/metrics/). Key metrics: `certmanager_certificate_expiration_timestamp_seconds`, `certmanager_certificate_ready_status`, `certmanager_http_acme_client_request_count`.
- **approver-policy** — [github.com/cert-manager/approver-policy](https://github.com/cert-manager/approver-policy). If you need to gate certain cert requests on policy (e.g., wildcard certs require human approval).
- **Stakater Reloader** — [github.com/stakater/Reloader](https://github.com/stakater/Reloader). Watches Secrets/ConfigMaps for changes; triggers Deployment rollouts. The standard pattern for picking up cert rotations in running pods.

---

## The 4 dimensions (senior framing)

- **Tech**: five CRDs in the chain; the resulting Secret has `tls.crt` (chain), `tls.key`, `ca.crt`; ACME `Challenge` is the most common failure point; renewal starts at `renewBefore` (default 1/3 of duration); cert-manager retries on failure with exponential backoff.
- **People**: `ClusterIssuer` vs `Issuer` choice affects team autonomy. Platform teams generally own `ClusterIssuer`; application teams write `Certificate` objects. Document which issuers exist and when to use each in your internal runbook. New engineers need to know: "if the cert isn't working, run `cmctl status certificate`" — not "read 50 lines of controller logs."
- **CI/CD**: `ClusterIssuer` and `Certificate` resources should be in version control (Helm values or plain YAML). Issuers managed separately from application certs — platform team's responsibility. Use staging issuer in CI; switch to prod in production. GitOps: account key Secret (the ACME account key) should be provisioned via External Secrets Operator, not committed to Git.
- **Operations**: Prometheus metrics for cert expiry (`certmanager_certificate_expiration_timestamp_seconds`) and readiness (`certmanager_certificate_ready_status`). Alert on: cert expiry < 30 days (ticket), < 7 days (urgent), < 24 hours (P1). Alert on: any `Certificate` NotReady for > 30 minutes. Alert on: `Challenge` objects in `pending` state for > 15 minutes. Runbook: "cert stuck" → `cmctl status` → fix the challenge → verify with `cmctl status` again.
