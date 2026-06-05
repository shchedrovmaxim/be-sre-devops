# CloudFront + WAF + Shield — the simple version (the castle gate analogy)

> Read this first. Once the layers click as a castle gate, the [deep-dive doc](./deep-dive.md) will be obvious.

This doc only explains **one idea**:

> **CloudFront, WAF, Shield, and ALB are not redundant — they're layered. Each layer does exactly one job, and each job happens at a different point in the attack surface.**

That's it. Everything else (caching behaviors, Shield Advanced cost, WAF managed rule groups, origin protection) is precision on top.

---

## The castle gate analogy

Imagine a medieval castle. You don't protect it with one big door. You have layers:

| Castle layer | What it does | Equivalent in AWS |
|---|---|---|
| Moat and drawbridge far outside the city | Keeps attackers away from the city entirely | **CloudFront edge** — traffic enters at the nearest AWS edge PoP (Point of Presence), not your VPC |
| Guards at the city gate checking papers | Inspects everyone entering — blocks known criminals | **WAF** — inspects HTTP requests, blocks by rule (rate limit, geo, OWASP top-10) |
| Royal knights patrolling for siege weapons | Active defense against large-scale attacks | **Shield Advanced** — DDoS detection and response at the edge |
| The castle's own inner gate | Final checkpoint before the keep | **ALB** — terminates TLS, routes to your services |

An attacker has to get through **all four layers**. Each layer independently reduces the attack surface.

---

## What each layer actually does

### CloudFront — the global cache + TLS terminator

CloudFront is a CDN. Requests from users hit the nearest AWS edge location (there are 600+), not your EC2/EKS origin.

Two jobs:
1. **Caches static content** — images, CSS, JS, API responses with `Cache-Control` headers — so your origin never sees the request.
2. **Terminates TLS at the edge** — users get a fast, geographically close TLS handshake. Your origin only sees one TLS connection from CloudFront's IP range.

The side effect: DDoS traffic is absorbed at the edge. CloudFront's distributed network can absorb enormous volumes before anything reaches your VPC.

### Shield — DDoS protection

AWS Shield has two tiers:

- **Shield Standard** — free, automatic, always-on. Protects against L3/L4 (network/transport) attacks (SYN floods, UDP reflection). Included for all CloudFront and ALB resources automatically.
- **Shield Advanced** — paid ($3k/month base + data transfer fees). Adds: near-real-time visibility, AWS SRT (Shield Response Team) support, automatic application-layer (L7) DDoS detection, and **cost protection** (AWS credits your bill for traffic spike costs caused by a DDoS).

For most teams: Shield Standard is enough. Shield Advanced makes sense when the business cost of a prolonged DDoS is high enough to justify $3k/month plus the support premium.

### WAF — the HTTP firewall

WAF inspects individual HTTP requests and applies rules. Rules can:
- **Rate-limit** by IP (e.g., 1000 requests/5 min)
- **Block by geo** (block traffic from specific countries)
- **Match OWASP Top-10 patterns** (SQL injection, XSS, malformed requests) via AWS managed rule groups
- **Block by header/cookie/body pattern** (custom rules)

WAF rules run **before** the request reaches your ALB — so a blocked request never touches your cluster.

### ALB — the final routing layer

ALB terminates TLS (again — from CloudFront, not the user), applies target group routing, and forwards requests to your Ingress controller → Service → pod.

The ALB should **only accept traffic from CloudFront**. If it's open to the internet, attackers can bypass all your WAF and Shield layers by targeting the ALB IP directly.

---

## The layered model — left to right

```
User request
  → CloudFront edge (cache hit? serve from cache; miss: continue)
  → WAF rules evaluated (block? 403; pass: continue)
  → Shield monitoring (DDoS detected? mitigate; normal: continue)
  → ALB (only accepts traffic from CloudFront IPs)
  → Ingress controller
  → Service → Pod
```

If the user hits a cached response at CloudFront, they never touch WAF, ALB, or your cluster. That's the cost-saving win, not just the security win.

---

## The one gotcha you must know: origin protection

ALB has a public IP. If an attacker knows that IP (or finds it via DNS history), they can send traffic **directly to the ALB**, bypassing CloudFront and WAF entirely.

Fix: **only allow traffic from CloudFront IPs at the ALB**.

Two mechanisms:
1. **Custom header + WAF rule**: CloudFront adds a secret header (e.g., `X-Origin-Verify: <secret>`) to every request. An ALB WAF rule blocks requests missing that header. Simple and effective.
2. **CloudFront managed prefix list + security group**: AWS publishes a managed prefix list of CloudFront origin-facing IPs. Add it to the ALB security group's inbound rules — only CloudFront IPs can reach port 443.

Most teams use approach 1 (custom header) because it's simpler and doesn't require security group updates when CloudFront IP ranges change.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does CloudFront do? | Caches content at edge; terminates TLS close to user; absorbs DDoS volume |
| What does Shield Standard do? | L3/L4 DDoS protection — free, automatic |
| When do I need Shield Advanced? | High-value targets where DDoS downtime is very costly; adds SRT support + cost protection |
| What does WAF do? | HTTP-layer firewall: rate limit, geo block, OWASP rules, custom rules |
| What does ALB do? | TLS termination from CloudFront; K8s routing; should be locked to CloudFront IPs only |
| What's the order? | User → CloudFront → WAF → Shield (monitoring) → ALB → cluster |
| What's origin protection? | Ensuring the ALB only accepts requests from CloudFront, not direct internet traffic |

---

## Self-test (one question — the killer one)

Out loud:

> **"Design the front door for a production EKS web app: CloudFront, Shield, WAF, ALB. What does each layer do and what order?"**

**Reference answer (intuitive version):**

"The order is: user → CloudFront → WAF → ALB → EKS cluster. CloudFront sits at the edge, caches static content, terminates TLS nearest to the user, and absorbs large-scale volumetric attacks before they reach the VPC. WAF is attached to the CloudFront distribution (or ALB if you want redundancy) and applies rules — rate limiting, geo blocking, AWS managed rule groups for OWASP Top-10. Shield Standard is automatic at CloudFront and ALB; if we're a high-value DDoS target, Shield Advanced adds the SRT and cost protection. ALB routes to the ingress controller.

The critical detail: the ALB must only accept traffic from CloudFront. If it's open to the internet, attackers bypass WAF entirely. The standard pattern is a custom header: CloudFront injects `X-Origin-Verify: <secret>`, and a WAF rule on the ALB blocks anything missing that header."

If that came out cleanly, jump to the [deep-dive](./deep-dive.md) for caching behaviors, WAF rule group pricing, Shield Advanced trade-offs, and the CF + cert-manager TLS pattern.

---

## Further reading / watching

- **AWS docs — CloudFront + WAF integration**: search "AWS WAF CloudFront distribution" in the AWS docs — the setup guide is the fastest start
- **AWS docs — Shield Advanced**: the pricing and use-case comparison is in the Shield User Guide
- **AWS blog — "Protect your web application using AWS WAF managed rules"** — explains managed rule groups (the quickest WAF win)
- **AWS docs — CloudFront origin protection**: "Restricting access to an Application Load Balancer" in the CloudFront Developer Guide

---

## Next: the deep-dive

When the castle-gate layering feels obvious, jump to [`cloudfront-waf-shield.md`](./deep-dive.md). The deep-dive covers:

- CloudFront distributions: origins, behaviors, cache policies, invalidation
- WAF rule groups: managed vs custom, rate-based rules, the cost model
- Shield Advanced: what the SRT actually does, cost protection mechanics, when it's worth it
- Origin protection: the custom-header pattern step by step, managed prefix list alternative
- CloudFront + ACM TLS: how certificates work at the edge vs the origin
- The self-test drills and 4-dimensions framing
