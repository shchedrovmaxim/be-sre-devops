# Cert rotation + monitoring expiry — the simple version (the passport expiry problem at scale)

> Read this first. Once the passport analogy clicks, the [deep-dive doc](./cert-rotation.md) covers the exact tooling and the famous outage patterns.

One idea:

> **A cert expiry is a self-inflicted outage. You knew the cert would expire — the date is stamped on it. The problem is that with hundreds of services, no human remembers to renew them all. The engineering challenge is detecting expiry before it happens, and automating renewal so humans don't need to remember.**

---

## The passport problem at scale

Imagine a company with 500 employees, each needing a passport to travel. The HR team tracks expiry dates in a spreadsheet. It works when there are 10 people. At 500:

- Someone's passport expires on Friday and they find out at the airport on Monday.
- The HR team doesn't know which passports expire next week because the spreadsheet has 500 rows.
- Nobody remembered to check — everyone assumed someone else was watching.

Now imagine each employee has **three passports** (internal cert, client cert, CA intermediate cert). And each passport is only valid for **90 days** instead of 10 years.

That's what a microservices platform looks like without cert automation. The famous LinkedIn-style outage: a cert nobody was watching expired at 3am, causing HTTPS handshakes to fail, causing cascading service-to-service failures that looked like a database outage until someone noticed the date on the cert.

---

## The two problems to solve

### Problem 1: Renewal — don't let certs expire

cert-manager solves this. If you use cert-manager, renewal is automatic: it renews 30 days before expiry (default `renewBefore`). The only failure mode is if cert-manager itself is broken, or if the issuer (Let's Encrypt, Vault) is unreachable.

**What still needs human attention**: certs that aren't managed by cert-manager. Legacy systems, external load balancers, CDN edge certs, database TLS certs, wildcard certs managed by a third party. These are the ones that expire silently.

### Problem 2: Detection — know before users do

You want to know a cert is expiring **before** it causes an outage, not after. Two layers:

1. **Prometheus blackbox exporter** — connects to the live endpoint over TLS and measures the cert's expiry. Alerts at 30 days, 14 days, 7 days. This is end-to-end: it catches cert problems even if cert-manager doesn't know about them (e.g., a CDN cert).

2. **cert-manager controller metrics** — expose Prometheus metrics about `Certificate` objects: how many are Ready, how many are failing, when each is due for renewal. This catches cert-manager's own view — but only for certs it manages.

You need both: blackbox catches what cert-manager doesn't manage; cert-manager metrics catch cert-manager failures before they become blackbox failures.

---

## The renewal window that actually matters

cert-manager's default is to renew when 1/3 of the cert's lifetime remains:
- 90-day cert: renews at day 60 (30 days before expiry)
- 24-hour cert: renews at 16 hours (8 hours before expiry)

The 30-day renewal window for 90-day certs is intentional. Let's Encrypt can be down for days. Vault can have an outage. You want 30 days to notice and fix the problem before the cert expires.

Don't set `renewBefore: 24h` on a 90-day cert. If renewal fails, you have 24 hours to fix it — and you probably won't notice for a day.

---

## Intuition cheat-sheet

| Question | Answer |
|---|---|
| Why do cert expiry outages keep happening? | Manual processes at scale. Someone knew the cert would expire; nobody automated the renewal. |
| What is the blackbox exporter? | A Prometheus exporter that makes real TLS connections and reports cert expiry time. Catches any TLS endpoint, not just cert-manager-managed ones. |
| What does cert-manager expose in Prometheus? | Metrics on Certificate objects: Ready/NotReady count, expiry time, last renewal time. |
| When should an alert fire? | At 30 days for production certs (gives time to fix renewal failures). At 7 days for urgent action. At 1 day for P1. |
| What is the client-side pinning gotcha? | If a client pins your cert's fingerprint and you rotate it, the client rejects the new cert. You can't just rotate without coordinating. |
| How do you rotate mTLS client certs? | Dual-cert window: both old and new certs trusted simultaneously. Clients rotate to new cert. Then old cert removed. |
| What's the "soft expiry" pattern? | Treat cert-nearing-expiry as a degraded state (yellow), not a failed state (red). Alert early; don't wait for expiry to page. |
| What if cert-manager is the problem? | The blackbox exporter will catch it — it doesn't care how certs are managed, only whether TLS works. |

---

## Self-test (the killer question)

Out loud:

> **"How would you detect and prevent cert expiry outages across hundreds of services?"**

**Reference answer (intuitive version):**

"Two layers of defense: renewal automation and independent expiry monitoring.

For renewal: cert-manager with a `renewBefore` of 30 days on 90-day certs. That gives a 30-day window if renewal fails — enough time to notice and fix it without an outage. For short-lived internal certs (24h Vault certs), the renewal window is proportionally shorter and automated rotation is baked into the cert-manager or Vault Agent lifecycle.

For monitoring: Prometheus blackbox exporter hitting every HTTPS endpoint and reporting days until cert expiry. Alert at 30 days (ticket), 14 days (warning), 7 days (urgent), 1 day (P1 page). The blackbox exporter is the safety net — it catches certs that aren't managed by cert-manager (CDN edge, external LBs, database TLS).

Additionally, cert-manager exposes Prometheus metrics on Certificate objects — I'd alert on any Certificate that's NotReady or hasn't renewed within its expected window.

The senior gotcha: client-side pinning. If any clients pin the cert's fingerprint (common in mobile apps or internal services using certificate pinning instead of CA pinning), rotating the cert silently breaks them. The fix is a dual-cert rotation window: serve both the old and new cert simultaneously, let clients transition, then retire the old one. Or better: pin the CA, not the leaf cert."

---

## Further reading / watching

- **Prometheus blackbox exporter** — [github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter). The `ssl` module docs show how to configure TLS cert expiry probing.
- **cert-manager monitoring** — [cert-manager.io/docs/observability/metrics](https://cert-manager.io/docs/observability/metrics/). Lists every Prometheus metric exposed by the controller.
- **Let's Encrypt expiry bot** — [letsencrypt.org/docs/expiration-emails](https://letsencrypt.org/docs/expiration-emails/). Let's Encrypt sends expiry notification emails to the account. A fallback — don't rely on it as your primary alert.
- **The LinkedIn cert expiry post-mortem** — search "LinkedIn cert expiry incident" for community write-ups of the 2016 incident. The original is the canonical example of why this matters at scale.

---

## Next: the deep-dive

When the passport analogy is clear and you can name both monitoring layers without prompting, jump to [`cert-rotation.md`](./cert-rotation.md). The deep-dive covers:

- The LinkedIn-style outage pattern — exact failure cascade
- cert-manager renewal mechanics — the exact controller loop, what `cmctl renew` does
- Blackbox exporter configuration for TLS cert monitoring (YAML)
- cert-manager Prometheus metrics — the specific metric names and alert rules
- Manual rotation patterns — for when automation isn't available
- Client-side pinning gotchas and dual-cert rotation
- mTLS client cert rotation across many services
- The 4-dimensions senior framing
