# Certificates

TLS cert lifecycle on K8s. The rejection feedback specifically called out Let's Encrypt.

## Recommended order

1. **TLS / x.509 basics** — chain of trust, CA → intermediate → leaf
2. **Let's Encrypt + ACME protocol** ⚠️ rejection gap
3. **Challenge types** — HTTP-01, DNS-01, TLS-ALPN-01
4. **When DNS-01 is required** — wildcard certs
5. **cert-manager** — Issuer / ClusterIssuer / Certificate CRDs
6. **Rate limits + staging** — 50 certs / domain / week, always test against staging
7. **Vault as internal CA** — for service-to-service / mesh certs
8. **Cert rotation + monitoring expiry** — alerts before expiry, not after

## Files

(none yet — to be written)

## Why this matters

> "Significant gaps were identified regarding … Let's Encrypt."

Public-internet-facing service = needs valid TLS. cert-manager + Let's Encrypt is the default modern pattern. You should be able to set it up from scratch in a fresh cluster.

## Hands-on environment

- Local `kind` cluster + cert-manager via Helm
- A real domain you control (for DNS-01 testing)
- Cloudflare or Route 53 for DNS-01 (cert-manager has solver plugins for each)
- Always start with the Let's Encrypt **staging** issuer to avoid hitting rate limits
