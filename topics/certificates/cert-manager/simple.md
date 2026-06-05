# cert-manager — the simple version (the paperwork assembly line)

> Read this first. Once the assembly-line analogy clicks, the [deep-dive doc](./deep-dive.md) is a lot easier to follow.

One idea:

> **cert-manager is a Kubernetes controller that turns "I want a TLS cert" into a series of automated steps — each step represented by a CRD — so you can see exactly where the cert is in the pipeline at any time.**

---

## Your cert request is a paperwork assembly line

Think of a busy government office where your cert application moves through several desks before you get your passport:

| Desk in the office | cert-manager CRD |
|---|---|
| You fill in the application | `Certificate` — you declare what you want |
| The desk clerk creates a formal request form | `CertificateRequest` — the clerk's internal form |
| The clerk contacts the right issuing authority | `Order` — the negotiation with Let's Encrypt (or Vault) |
| The authority sets a test ("prove you live at this address") | `Challenge` — the HTTP-01 or DNS-01 proof |
| You pass the test; authority stamps the passport | The signed cert arrives back |
| The clerk files the passport in the office vault | `Secret` — the cert and private key, stored in K8s |

Each desk is a separate Kubernetes object you can inspect. If your cert is stuck, you find which desk it's sitting at and look at the events on that object. That's why there are so many CRDs — it's observability, not bureaucracy.

---

## The four actors

| Actor | What it does |
|---|---|
| `Issuer` / `ClusterIssuer` | Defines *how* to get a cert. Points at Let's Encrypt, Vault, or a self-signed CA. `ClusterIssuer` works across all namespaces; `Issuer` is namespace-scoped. |
| `Certificate` | Your request: "I want a cert for `api.example.com`, renewed 30 days before expiry, stored in Secret `api-tls`." |
| `CertificateRequest` | Created automatically by cert-manager when it processes your `Certificate`. Contains the CSR. You rarely write these. |
| `Order` + `Challenge` | Created automatically for ACME issuers. `Order` is the Let's Encrypt negotiation; `Challenge` is the HTTP-01 or DNS-01 proof step. |

You write `Issuer` (or `ClusterIssuer`) and `Certificate`. cert-manager creates the rest.

---

## What a Certificate resource actually looks like

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: production
spec:
  secretName: api-tls        # cert-manager writes the cert here
  issuerRef:
    name: letsencrypt-prod   # which ClusterIssuer to use
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
  renewBefore: 720h          # renew 30 days before expiry (default: 1/3 of duration)
  duration: 2160h            # 90 days; Let's Encrypt forces this
```

Once you `kubectl apply` this, cert-manager picks it up and starts the assembly line automatically.

---

## How renewal works

cert-manager checks cert expiry continuously. When the remaining lifetime hits the `renewBefore` threshold (default: 1/3 of cert duration), it creates a new `CertificateRequest` and kicks off a fresh ACME flow. The old cert stays valid until the new one is ready. When the new cert arrives, cert-manager updates the Secret — no downtime if your pods are watching for Secret updates (via a `Deployment` rollout or `cert-manager`'s auto-restart annotation).

The default `renewBefore` for a 90-day Let's Encrypt cert is 30 days — renewal at day 60. This gives you a 30-day window to fix problems if renewal fails.

---

## Debugging cheat-sheet

When a cert is stuck, check in this order:

```bash
# 1. Where is it stuck?
cmctl status certificate api-tls -n production

# 2. Look at the Certificate object events
kubectl describe certificate api-tls -n production

# 3. Look at the Order (if ACME)
kubectl describe order -n production

# 4. Look at the Challenge — this is usually where it stalls
kubectl describe challenge -n production

# 5. Check cert-manager controller logs
kubectl logs -n cert-manager deploy/cert-manager
```

99% of the time the problem is in `Challenge` events: the HTTP-01 endpoint isn't reachable, or the DNS-01 provider credentials are wrong.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What is cert-manager? | A K8s controller that automates cert issuance and renewal via CRDs. |
| Issuer vs ClusterIssuer? | Both define how to get a cert. `ClusterIssuer` works across all namespaces; `Issuer` is namespace-scoped. |
| What does a `Certificate` do? | Declares what you want. cert-manager reads it and creates the rest automatically. |
| What is a `CertificateRequest`? | cert-manager's internal form — the CSR sent to the issuer. You rarely write these manually. |
| What is an `Order`? | The ACME negotiation object — exists only with ACME issuers (Let's Encrypt). |
| What is a `Challenge`? | The ACME proof step — HTTP-01 or DNS-01. Where things usually go wrong. |
| Where does the cert live? | In a Kubernetes `Secret`, named by `secretName` in your `Certificate` spec. |
| How does renewal work? | cert-manager auto-renews when remaining lifetime < `renewBefore`. Old cert stays valid during the process. |
| Best debugging starting point? | `cmctl status certificate <name>` then work down to `Order` → `Challenge` events. |

---

## Self-test (the killer question)

Out loud:

> **"Walk through every cert-manager CRD created when I request a Certificate. What does each one do?"**

**Reference answer (intuitive version):**

"You write a `Certificate` resource saying what you want — domain names, duration, secret name, which issuer to use. cert-manager's controller sees it and creates a `CertificateRequest`, which is the internal representation of the CSR sent to the CA.

If the issuer is ACME (Let's Encrypt), cert-manager creates an `Order` — that's the negotiation with Let's Encrypt. The `Order` spawns one `Challenge` per domain name in the cert. The `Challenge` is the actual proof step — cert-manager spins up a temporary HTTP server (HTTP-01) or calls the DNS provider API (DNS-01) to place the token. Let's Encrypt checks the token. Once every Challenge is valid, the Order is finalized, Let's Encrypt signs the cert, and cert-manager stores the cert and private key in the `Secret` named in your `Certificate` spec.

The senior debugging move: when a cert is stuck, start with `cmctl status certificate`, then drill into the `Order`, then the `Challenge` events. The challenge is where 90% of failures happen — wrong ingress config for HTTP-01, wrong API credentials for DNS-01."

---

## Further reading / watching

- **cert-manager documentation** — [cert-manager.io/docs](https://cert-manager.io/docs/). The "Concepts" section has clean diagrams of every CRD.
- **`cmctl` reference** — [cert-manager.io/docs/reference/cmctl](https://cert-manager.io/docs/reference/cmctl/). `cmctl status certificate` is the fastest debugging entrypoint.
- **cert-manager GitHub** — [github.com/cert-manager/cert-manager](https://github.com/cert-manager/cert-manager). The `CHANGELOG.md` is worth scanning before upgrades — webhook changes have caused outages.

---

## Next: the deep-dive

When you can describe the CRD chain without looking it up, jump to [`cert-manager.md`](./deep-dive.md). The deep-dive covers:

- The controller dance in detail — which controller watches which object
- What's actually inside a `CertificateRequest` (the CSR format, the approval flow)
- Renewal mechanics — the exact timing, what happens on failure, what `cmctl renew` does
- The resulting `Secret` — the fields, how pods consume it, auto-restart strategies
- Common production failure modes and how to debug each one
- The 4-dimensions senior framing
