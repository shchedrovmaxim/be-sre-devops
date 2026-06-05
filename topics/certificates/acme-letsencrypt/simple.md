# ACME + Let's Encrypt — the simple version (the notary stamp)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) has the full state machine and the gotchas that surface in production.

This doc explains one idea:

> **Let's Encrypt needs to verify that you actually control the domain before it'll sign your certificate. ACME is the protocol that automates that entire negotiation — account creation, challenge, proof, and issuance — without you clicking anything.**

That's the whole concept. Everything else (account keys, challenge types, rate limits) is precision on top.

---

## Your domain is a property that needs a notarised title

You're a property owner applying for a loan. The bank won't just take your word that you own the house — they require proof: the notary checks the public registry and puts a stamp on the title. Until that stamp is there, no loan.

| In the property world | In the ACME world |
|---|---|
| You own a house | You own a domain (`example.com`) |
| The bank that issues the loan | Let's Encrypt — the CA that issues the cert |
| The notary + public registry | The ACME challenge — a live proof that you control DNS or HTTP for the domain |
| The stamp on the title | The signed certificate Let's Encrypt returns |
| You can't fake the registry | You can't fake owning a domain you don't control |
| Bank keeps your property as collateral | Your cert expires in 90 days — proof must be renewed |

The notary doesn't trust a photocopy; they go to the source. ACME does the same: Let's Encrypt doesn't trust you saying you own `example.com` — it calls out to the real world (HTTP or DNS) to verify it live, then stamps your certificate.

---

## The two keys you need to understand

One gotcha that confuses everyone the first time:

**ACME uses two completely different keys:**

| Key | What it's for |
|---|---|
| **Account key** | Identifies *you* as an ACME account holder. Stored by your ACME client (cert-manager, Certbot). Used to sign all ACME requests. |
| **Certificate key** | The actual private key for your TLS certificate. Generated fresh for each cert. Never sent anywhere — the cert request (CSR) is signed with it, but the key itself stays local. |

Think of it like this: the account key is your notary ID badge. The certificate key is the lock on your specific house. The notary uses your ID badge to verify who you are, then stamps the title for the house. Two different things.

---

## The ACME flow in plain English

1. You tell your ACME client (cert-manager): "I want a cert for `example.com`."
2. The client registers an account with Let's Encrypt (or reuses an existing one). This account is identified by the account key.
3. Let's Encrypt says: "Place this specific token at `http://example.com/.well-known/acme-challenge/<token>`" (HTTP-01 challenge) or "Add this TXT record to `_acme-challenge.example.com`" (DNS-01). See [acme-challenges.md](../acme-challenges/deep-dive.md) for the other types.
4. You (or cert-manager) place the token.
5. Let's Encrypt goes and checks that the token is really there.
6. If yes, Let's Encrypt signs your certificate and hands it back.
7. cert-manager stores it in a Kubernetes Secret.

The whole thing takes seconds to minutes when automated. cert-manager handles all of it — you just write a `Certificate` resource.

---

## The rate limit you will definitely hit

Let's Encrypt **production** enforces:

| Limit | Value |
|---|---|
| Certificates per registered domain per week | 50 |
| Duplicate cert renewals per week | 5 |
| Failed validations per hour per account | 5 |
| Accounts per IP per 3 hours | 10 |

50/week sounds like a lot — until you have 50 microservices each with their own cert, and someone triggers a mass renewal. The duplicate limit (5/week) is the sneaky one: if you re-issue the same cert more than 5 times in a week (same domain, same key), Let's Encrypt blocks you.

**The fix**: always use the **staging** environment first. Let's Encrypt staging has much higher limits and issues real ACME certs (that browsers don't trust, but your automation can test against). The issuer URL is:

```
https://acme-staging-v02.api.letsencrypt.org/directory
```

Hit staging → confirm it works → switch to production. This is non-negotiable in CI pipelines.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| What is ACME? | A protocol for automating domain-verified cert issuance. Client ↔ CA negotiate challenge, proof, issuance. |
| What is Let's Encrypt? | A free, automated CA that speaks ACME. Certs valid 90 days. |
| What's the account key? | Your ACME identity. Signs all requests. Not the cert key. |
| What's the certificate key? | The private key for the TLS cert. Stays local; never sent to the CA. |
| Why 90-day certs? | Short expiry forces automation. A cert that expires in 90 days is renewed at day 60 — no human forgets it. |
| Why use staging? | Production has strict rate limits. Hit staging first; switch to prod once your automation is confirmed. |
| What if Let's Encrypt is down? | Your cert still works until expiry. The problem is renewal — start renewal early (at 60 days, not 89). |
| What's a CAA record? | A DNS record saying "only these CAs are allowed to issue for this domain." Add `letsencrypt.org` or you may get blocked. |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through how Let's Encrypt verifies you own a domain before issuing a cert."**

**Reference answer (intuitive version):**

"You submit an order to Let's Encrypt for `example.com`. Let's Encrypt responds with a challenge — typically it asks you to place a specific token at a well-known HTTP path or in a DNS TXT record. Your ACME client (cert-manager in K8s) places that token. Let's Encrypt then goes and fetches that URL or queries DNS. If the token matches, it considers the challenge passed, and it signs your certificate. The whole proof is that only someone who controls the domain can put content at that HTTP path or modify that DNS record.

The senior nuance: there are two keys involved. The account key identifies your ACME account and signs all communication with Let's Encrypt. The certificate key is a separate key generated for the cert itself — its public half goes into the CSR, but the private key never leaves your system. Let's Encrypt signs the CSR, returns the cert. The cert is then stored as a K8s Secret by cert-manager.

Also worth mentioning: production rate limits are real — 50 certs per registered domain per week. You always test against Let's Encrypt staging first, then switch the issuer URL to production."

---

## Further reading / watching

- **Let's Encrypt official docs** — [letsencrypt.org/docs](https://letsencrypt.org/docs/). Start with "How It Works." The rate limits page is essential.
- **RFC 8555** — the ACME specification. You don't need to read it all; the introduction and section 7 (the flow) are worth skimming to understand the protocol shape.
- **cert-manager docs** — [cert-manager.io/docs](https://cert-manager.io/docs/). The "Getting Started" section covers the ClusterIssuer + Certificate flow that most K8s engineers use day-to-day.
- **Let's Encrypt staging environment** — [letsencrypt.org/docs/staging-environment](https://letsencrypt.org/docs/staging-environment/). Bookmark this. You will need it.

---

## Next: the deep-dive

When the notary-stamp analogy feels obvious and you can explain the two-key distinction without looking it up, jump to [`acme-letsencrypt.md`](./deep-dive.md). The deep-dive covers:

- The full ACME state machine (newAccount → newOrder → authorization → challenge → finalize → download)
- Account key vs cert key — the exact JOSE/JWK signing mechanics
- Rate limits in detail — the ones that will bite you in CI
- Let's Encrypt's CA cert chain (ISRG Root X1, the cross-sign story, what happened to Android in 2021)
- Why 90-day certs are a security *feature*, not a limitation
- Self-test drills with reference answers
- The 4-dimensions senior framing
