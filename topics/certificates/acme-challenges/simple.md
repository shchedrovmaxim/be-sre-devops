# ACME challenge types — the simple version (the two ways to prove you own a house)

> Read this first. Once the two-proof analogy clicks, the [deep-dive doc](./deep-dive.md) has the exact mechanics and infra requirements for each type.

One idea:

> **To prove you own a domain, Let's Encrypt can check either what's reachable at that domain over HTTP, or what DNS records you've set for it. Those are two completely different proofs with completely different infra requirements — and which one you need depends on where your cluster lives.**

---

## Proving you own a house: two approaches

A notary can verify you own a property in two ways:

| Method | Analogy | In ACME terms |
|---|---|---|
| **The front door test** — walk up to the house, knock, and confirm you can open the door | **HTTP-01** — Let's Encrypt fetches a URL over port 80 and checks a token is there | Needs port 80 reachable from the public internet |
| **The deed at the registry** — look up the official land registry and check your name is on it | **DNS-01** — Let's Encrypt queries a DNS TXT record and checks a token is there | Needs API access to your DNS provider; port 80 not needed |
| **The keyhole test** — slide a signed envelope through the TLS keyhole itself | **TLS-ALPN-01** — Let's Encrypt negotiates a special TLS handshake on port 443 | Needs port 443 reachable; less common; no DNS API needed |

The notary will use whichever proof works for the situation. If the house is in a gated community with no public access (private cluster, internal LB), only the registry approach works.

---

## The critical rule for wildcards

`*.example.com` requires **DNS-01, no exceptions.**

HTTP-01 can only prove you control one specific hostname. DNS-01 proves you control the *zone* — which is what a wildcard requires. If you want `*.example.com`, you need API access to your DNS provider.

This one rule drives a lot of infra decisions. cert-manager with DNS-01 needs credentials for Route 53 (ideally via IRSA in EKS), Cloudflare, or your provider of choice.

---

## The two you'll use in practice

### HTTP-01 — the easy one for public clusters

How it works:
1. cert-manager deploys a temporary HTTP server in your cluster.
2. It puts the challenge token at `http://<domain>/.well-known/acme-challenge/<token>`.
3. Let's Encrypt fetches that URL from the public internet.
4. Token matches → challenge passes.

Needs: port 80 accessible from the public internet. Ingress controller must be configured to forward `/.well-known/acme-challenge/` to cert-manager's solver pod.

Doesn't work:
- Private clusters with no public LB
- Wildcard certs

### DNS-01 — the one for private clusters and wildcards

How it works:
1. cert-manager calls your DNS provider's API (Route 53, Cloudflare, etc.) and creates a TXT record: `_acme-challenge.example.com = <token>`.
2. Let's Encrypt queries DNS for that TXT record (from public resolvers).
3. Token matches → challenge passes.
4. cert-manager cleans up the TXT record.

Needs: cert-manager must have credentials to create and delete DNS records. For Route 53 in EKS, this is IRSA. For Cloudflare, it's an API token stored in a Secret.

Works for:
- Private clusters (no public HTTP endpoint needed)
- Wildcard certs (`*.example.com`)
- Clusters behind corporate firewalls

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What is HTTP-01? | Let's Encrypt fetches a token over HTTP on port 80. Simple, needs no DNS API. |
| What is DNS-01? | Let's Encrypt checks a TXT record you set via DNS API. Works for private clusters + wildcards. |
| What is TLS-ALPN-01? | Let's Encrypt checks a special TLS handshake on port 443. Rarely used; needs port 443 exposed. |
| When is DNS-01 required? | Always for wildcard certs (`*.example.com`). Also for clusters where port 80 isn't public. |
| What does DNS-01 need in EKS? | An IAM role with Route 53 permissions, bound to cert-manager's ServiceAccount via IRSA. |
| Can you mix challenge types? | Yes — per-Issuer or per-Certificate. Wildcard uses DNS-01; single-hostname can use HTTP-01. |
| What's the HTTP-01 gotcha? | Let's Encrypt follows redirects, but must be able to reach port 80 initially — not 443 redirect only. |
| What's the DNS-01 gotcha? | DNS propagation delay. cert-manager waits for the TXT record to appear; if DNS is slow, the challenge times out. |

---

## Self-test (the killer question)

Out loud:

> **"When would you use DNS-01 instead of HTTP-01, and what infra setup does each need?"**

**Reference answer (intuitive version):**

"HTTP-01 is simpler — cert-manager spins up a temporary solver pod, places a token at `/.well-known/acme-challenge/`, and Let's Encrypt fetches it. You just need port 80 reachable from the public internet and your ingress forwarding that path. Works for any single-hostname cert on a public cluster.

DNS-01 is required in two situations: wildcard certs (`*.example.com`) and private clusters where port 80 isn't exposed. With DNS-01, cert-manager calls your DNS provider's API to drop a TXT record, Let's Encrypt queries that TXT record from public resolvers, then cert-manager cleans it up. The infra requirement is DNS API credentials — for Route 53 in EKS, that's an IRSA-annotated ServiceAccount with the right IAM policy; for Cloudflare, a scoped API token in a K8s Secret.

The senior nuance: DNS-01 has a propagation delay. If your DNS TTL is 5 minutes and you just created the TXT record, Let's Encrypt might query before it propagates and fail the challenge. cert-manager handles retries, but you need to account for this in time-sensitive scenarios."

---

## Further reading / watching

- **Let's Encrypt challenge types** — [letsencrypt.org/docs/challenge-types](https://letsencrypt.org/docs/challenge-types/). The official explanation of each type with diagrams.
- **cert-manager DNS-01 solvers** — [cert-manager.io/docs/configuration/acme/dns01](https://cert-manager.io/docs/configuration/acme/dns01/). Lists every supported DNS provider with config examples.
- **cert-manager Route 53 + IRSA tutorial** — search "cert-manager Route53 IRSA" on the cert-manager docs site. Shows the exact IAM policy and ServiceAccount annotation needed.

---

## Next: the deep-dive

When the front-door vs registry analogy is clear and you can name the infra requirements for each type without hesitation, jump to [`acme-challenges.md`](./deep-dive.md). The deep-dive covers:

- HTTP-01 exact mechanics — the solver pod, ingress rules, redirect behavior
- DNS-01 exact mechanics — the TXT record format, TTL considerations, split-horizon DNS gotcha
- TLS-ALPN-01 — when it's actually useful
- Route 53 + IRSA setup (the real YAML)
- Private cluster patterns — DNS-01 behind a VPN/firewall
- The 4-dimensions senior framing
