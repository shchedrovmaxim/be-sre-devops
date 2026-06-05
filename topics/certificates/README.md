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

| File | Type | Topic | Date |
|---|---|---|---|
| [acme-letsencrypt-simple.md](./acme-letsencrypt/simple.md) | Simple companion | ACME + Let's Encrypt | 2026-06-05 |
| [acme-letsencrypt.md](./acme-letsencrypt/deep-dive.md) | Deep-dive | ACME + Let's Encrypt | 2026-06-05 |
| [cert-manager-simple.md](./cert-manager/simple.md) | Simple companion | cert-manager CRDs | 2026-06-05 |
| [cert-manager.md](./cert-manager/deep-dive.md) | Deep-dive | cert-manager CRDs | 2026-06-05 |
| [acme-challenges-simple.md](./acme-challenges/simple.md) | Simple companion | HTTP-01 vs DNS-01 vs TLS-ALPN-01 | 2026-06-05 |
| [acme-challenges.md](./acme-challenges/deep-dive.md) | Deep-dive | HTTP-01 vs DNS-01 vs TLS-ALPN-01 | 2026-06-05 |
| [vault-pki-simple.md](./vault-pki/simple.md) | Simple companion | Vault as internal CA | 2026-06-05 |
| [vault-pki.md](./vault-pki/deep-dive.md) | Deep-dive | Vault as internal CA | 2026-06-05 |
| [cert-rotation-simple.md](./cert-rotation/simple.md) | Simple companion | Cert rotation + monitoring expiry | 2026-06-05 |
| [cert-rotation.md](./cert-rotation/deep-dive.md) | Deep-dive | Cert rotation + monitoring expiry | 2026-06-05 |

## Why this matters

> "Significant gaps were identified regarding … Let's Encrypt."

Public-internet-facing service = needs valid TLS. cert-manager + Let's Encrypt is the default modern pattern. You should be able to set it up from scratch in a fresh cluster.

## Hands-on environment

- Local `kind` cluster + cert-manager via Helm
- A real domain you control (for DNS-01 testing)
- Cloudflare or Route 53 for DNS-01 (cert-manager has solver plugins for each)
- Always start with the Let's Encrypt **staging** issuer to avoid hitting rate limits
