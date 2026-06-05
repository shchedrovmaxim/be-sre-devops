# CloudFront + Shield + WAF in front of EKS — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Design the front door for a production EKS web app: CloudFront, Shield, WAF, ALB. What does each layer do and what order?"** — naming each layer's job, the attack surface each closes, the origin protection pattern, caching behaviors, and the cost/trade-off at each tier.

> Start with the [simple version](./simple.md) first — the castle gate analogy is the mental model.

---

## The senior framing — defense in depth is a layering discipline

Mid-level engineers add WAF because "security team said so." Senior engineers understand each layer's **specific attack surface** and make conscious cost/benefit decisions about each.

The question is never "do we need WAF?" but:
- Which layer handles which threat?
- What is the cost at each layer ($ and operational)?
- What is the failure mode if a layer is bypassed?
- How do the layers compose without creating gaps?

The layered model is a principle (defense in depth), not a checkbox. A team that understands the layers can defend the trade-off in a cost review. A team that added WAF "because security said so" can't.

---

## The full layered model

```
                         [User]
                           |
                           ▼
              ┌────────────────────────┐
              │     CloudFront Edge    │  ← TLS termination (ACM cert)
              │   (600+ PoPs global)   │  ← CDN cache (static content, API cache)
              │                        │  ← TLS closest to user (fast handshake)
              └──────────┬─────────────┘
                         │
              ┌──────────▼─────────────┐
              │          WAF           │  ← Attached to CloudFront or ALB
              │   (HTTP/HTTPS layer)   │  ← Rate limit, geo block, OWASP rules
              │                        │  ← Evaluated before origin is called
              └──────────┬─────────────┘
                         │
              ┌──────────▼─────────────┐
              │    Shield (Standard    │  ← Automatic L3/L4 DDoS mitigation
              │    or Advanced)        │  ← Network/transport layer (SYN floods, etc.)
              │                        │  ← Advanced: near-RT visibility + SRT
              └──────────┬─────────────┘
                         │
              ┌──────────▼─────────────┐
              │          ALB           │  ← TLS termination from CloudFront
              │  (origin, in VPC)      │  ← Health checks, target group routing
              │                        │  ← MUST only accept traffic from CF IPs
              └──────────┬─────────────┘
                         │
              ┌──────────▼─────────────┐
              │  Ingress Controller    │  ← K8s-level routing
              │  (e.g., AWS LBC)       │
              └──────────┬─────────────┘
                         │
                   [Service → Pod]
```

Each layer is independently valuable. You can have CloudFront without WAF (caching + TLS acceleration, no filtering). You can have WAF on the ALB without CloudFront (filtering at origin, no edge caching). The full stack is the senior-level default for public-facing production.

---

## Layer 1: CloudFront

### What it does

CloudFront is a CDN (Content Delivery Network). When a user makes a request:

1. DNS resolves to the nearest CloudFront edge PoP (via Anycast routing).
2. If the response is in CloudFront's cache → served immediately from edge. Origin (ALB) never sees the request.
3. If cache miss → CloudFront forwards to the origin (ALB). Caches the response for future requests.

### Distributions, origins, behaviors

A **CloudFront distribution** has:
- One or more **origins** (where to forward cache misses — your ALB)
- One or more **cache behaviors** (rules matching request paths, controlling what's cached)

```hcl
# Terraform excerpt — CloudFront distribution (simplified)
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "alb-main"

    custom_header {
      name  = "X-Origin-Verify"
      value = var.origin_verify_secret   # the secret header for origin protection
    }

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"   # always TLS between CF and ALB
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "alb-main"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]

    cache_policy_id          = data.aws_cloudfront_cache_policy.caching_optimized.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.allviewer.id
  }

  ordered_cache_behavior {
    path_pattern     = "/api/*"
    target_origin_id = "alb-main"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]

    cache_policy_id        = data.aws_cloudfront_cache_policy.caching_disabled.id  # don't cache API responses
    viewer_protocol_policy = "redirect-to-https"
  }

  web_acl_id = aws_wafv2_web_acl.main.arn   # WAF attachment
}
```

**Cache behaviors** are evaluated in order. The first match wins. `default_cache_behavior` is the fallback.

### TLS — CloudFront + ACM

CloudFront requires a certificate in **ACM in us-east-1** (regardless of where your resources are). This is the one hard requirement: even if your EKS cluster is in `eu-west-1`, the CloudFront cert lives in `us-east-1`.

The ALB has its own cert (in the region the ALB is in) for the CloudFront-to-origin leg.

```
User ──TLS (ACM cert, us-east-1)──► CloudFront
CloudFront ──TLS (ACM cert, ALB region)──► ALB
```

Two TLS sessions, two certs, two terminations.

### Cache invalidation

When content changes, you need to invalidate CloudFront's cache. Options:
- **Invalidation API**: `aws cloudfront create-invalidation --paths "/static/*"` — takes 30-60 seconds; free up to 1,000 paths/month, then $0.005/path.
- **Versioned file names**: `app.abc123.js` → cache busted by changing the filename. Preferred for static assets. No invalidation API call needed.
- **Short TTL**: set `Cache-Control: max-age=60` on mutable content. CloudFront respects it.

---

## Layer 2: WAF (Web Application Firewall)

### What WAF does

WAF operates at the HTTP/HTTPS layer (Layer 7). It inspects the request before forwarding to the origin. Rules can match:
- IP address or IP set
- Geographic location
- HTTP headers, body, URI, query string
- Rate (requests per IP per time window)
- Managed rule group patterns (OWASP, bot control, known bad inputs)

### WAF rule types

**Managed rule groups** — pre-built by AWS or vendors, updated automatically:

| Rule group | What it blocks | Monthly cost |
|---|---|---|
| `AWSManagedRulesCommonRuleSet` | OWASP Top-10: SQLi, XSS, RFI, LFI | ~$1.50/million requests |
| `AWSManagedRulesKnownBadInputsRuleSet` | Log4Shell, ShellShock, common exploit payloads | ~$1.50/million |
| `AWSManagedRulesAmazonIpReputationList` | AWS Threat Intel — known bad IPs | ~$1.50/million |
| `AWSManagedRulesBotControlRuleSet` | Bot detection + CAPTCHA | ~$10/million (bot control is more expensive) |

**Custom rules** — you write the match conditions:

```json
{
  "Name": "BlockMissingOriginHeader",
  "Priority": 0,
  "Action": { "Block": {} },
  "Statement": {
    "NotStatement": {
      "Statement": {
        "ByteMatchStatement": {
          "FieldToMatch": { "SingleHeader": { "Name": "x-origin-verify" } },
          "SearchString": "your-secret-here",
          "PositionalConstraint": "EXACTLY",
          "TextTransformations": [{ "Priority": 0, "Type": "NONE" }]
        }
      }
    }
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockMissingOriginHeader"
  }
}
```

**Rate-based rules** — count requests from an IP over 5 minutes; block when threshold exceeded:

```json
{
  "Name": "RateLimitByIP",
  "Priority": 1,
  "Action": { "Block": {} },
  "Statement": {
    "RateBasedStatement": {
      "Limit": 1000,
      "AggregateKeyType": "IP"
    }
  }
}
```

### WAF pricing model

- **Web ACL**: $5/month
- **Rule**: $1/month per rule
- **Requests**: $0.60 per 1M requests (first 10M)
- **Managed rule groups**: additional $1-10/million depending on the group

For a service at 100M requests/month: WAF costs ~$60 in request fees plus managed rule group fees. Not a budget item — it's a baseline cost.

### WAF on CloudFront vs WAF on ALB

WAF can be attached to:
- **CloudFront distribution** — evaluated at the edge; blocked requests never reach the VPC. Cheapest (fewer requests hit origin) and most effective.
- **ALB** — evaluated at the origin; requests already crossed the internet and entered your VPC. More expensive but necessary if you don't have CloudFront.

Best practice: attach WAF to the CloudFront distribution as the primary filter. Optionally add a minimal WAF to the ALB as a secondary defense (e.g., just the "missing origin header" rule).

---

## Layer 3: Shield

### Shield Standard — what it actually does

Shield Standard is automatic for CloudFront, ALB, Elastic IP, and Route 53. It provides:
- **L3 protection**: IP-level attacks — protocol spoofing, fragmentation attacks
- **L4 protection**: TCP/UDP floods — SYN floods, UDP amplification (DNS reflection, NTP reflection)

Standard detects and mitigates automatically within seconds. No configuration needed. You don't "enable" it; it's always on.

### Shield Advanced — when it's worth it

Shield Advanced adds:
- **Near-real-time DDoS metrics** in CloudWatch: bytes/packet/request rates with attack detection context
- **AWS Shield Response Team (SRT)** 24/7 access — AWS engineers who engage during an active DDoS and write custom WAF rules in real time
- **Automatic application-layer protection** — AWS can automatically create WAF rules in response to detected L7 DDoS patterns (requires WAF to be attached)
- **Cost protection** — AWS credits your bill for data transfer and scaling costs caused by a DDoS. A large volumetric attack can inflate your bill by thousands of dollars; Shield Advanced makes AWS absorb that cost.

**Price**: $3,000/month base + data transfer out costs (post-Shield traffic). Per-account, not per-resource.

**When to use it**:
- You're a high-value target (fintech, healthcare, media, government)
- You've experienced DDoS before and the cost/downtime was material
- Your business continuity requires 24/7 SRT engagement
- The $3k/month is less than a 30-minute DDoS event costs you in lost revenue

**When to skip it**:
- Small/medium traffic, low DDoS risk
- Shield Standard + WAF rate limiting handles most threats at a fraction of the cost

---

## Origin protection — the critical gap

The most common CloudFront misconfiguration: the ALB is publicly accessible with an AWS-assigned `*.us-east-1.elb.amazonaws.com` hostname. Attackers can discover this hostname (via DNS history, certificate transparency logs, Shodan) and send requests directly to the ALB, bypassing CloudFront and WAF entirely.

### Method 1: Custom header (recommended)

CloudFront injects a secret header on every request to the origin:

```hcl
# Terraform: CloudFront origin custom header
origin {
  custom_header {
    name  = "X-Origin-Verify"
    value = random_password.origin_secret.result
  }
}
```

ALB has a WAF Web ACL with a rule that blocks requests missing this header:

```hcl
resource "aws_wafv2_web_acl" "alb" {
  scope = "REGIONAL"   # ALB-attached WAF is always REGIONAL (not CLOUDFRONT)

  rule {
    name     = "RequireOriginHeader"
    priority = 0
    action { block {} }
    statement {
      not_statement {
        statement {
          byte_match_statement {
            field_to_match { single_header { name = "x-origin-verify" } }
            search_string         = var.origin_verify_secret
            positional_constraint = "EXACTLY"
            text_transformation { priority = 0; type = "NONE" }
          }
        }
      }
    }
    visibility_config { sampled_requests_enabled = true; cloudwatch_metrics_enabled = true; metric_name = "RequireOriginHeader" }
  }
}
```

The secret should be stored in Secrets Manager and rotated periodically. CloudFront supports multiple origin secrets simultaneously during rotation — no downtime for key rotation.

### Method 2: CloudFront managed prefix list

AWS publishes a managed prefix list of CloudFront origin-facing IP ranges: `com.amazonaws.global.cloudfront.origin-facing`.

```hcl
resource "aws_security_group_rule" "allow_cloudfront" {
  security_group_id = aws_security_group.alb.id
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  prefix_list_ids   = ["pl-3b927c52"]   # CloudFront origin-facing managed prefix list
}
```

This blocks all traffic to the ALB that isn't from CloudFront's origin-facing IPs. No secret needed. But: the prefix list is large (~100 CIDRs) and counts against security group rule limits. AWS can also expand the list.

**Use Method 1 (custom header + WAF) as the default.** Simpler, no IP-range limit concerns, works even if CloudFront IPs change.

---

## Caching behaviors — practical patterns

### Static content (aggressive caching)

```hcl
ordered_cache_behavior {
  path_pattern           = "/static/*"
  cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # CachingOptimized
  viewer_protocol_policy = "redirect-to-https"
  compress               = true
}
```

TTL: 86400s (1 day) or longer if files are versioned. CloudFront serves from edge; origin never sees repeat requests.

### API responses (no caching, forward headers)

```hcl
ordered_cache_behavior {
  path_pattern             = "/api/*"
  cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled
  origin_request_policy_id = "b689b0a8-53d0-40ab-baf2-68738e2966ac"  # AllViewer
  viewer_protocol_policy   = "redirect-to-https"
  allowed_methods          = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
}
```

`AllViewer` origin request policy forwards all headers, cookies, and query strings to the origin — necessary for auth headers and session cookies.

### WebSockets

CloudFront supports WebSocket connections natively. No special configuration needed; the protocol upgrade happens transparently. Ensure the ALB target group has WebSocket support enabled.

---

## The interview answer in 60 seconds

> "The order is: user → CloudFront → WAF → Shield → ALB → EKS cluster.
>
> CloudFront is the edge layer: it terminates TLS closest to the user using an ACM cert in us-east-1, caches static content and eligible API responses at 600+ PoPs globally, and absorbs volumetric traffic before it reaches the VPC. For static assets, the origin never sees repeat requests.
>
> WAF is attached to the CloudFront distribution and evaluates HTTP rules before CloudFront calls the origin: rate limiting by IP, geo blocking, and AWS managed rule groups for OWASP Top-10 patterns. WAF runs at the edge — blocked requests never touch the VPC.
>
> Shield Standard is automatic at CloudFront and ALB — L3/L4 DDoS mitigation. If this is a high-value DDoS target, Shield Advanced adds the SRT and cost protection for $3k/month base.
>
> ALB is the origin. It must only accept traffic from CloudFront. The standard pattern is a custom header: CloudFront injects `X-Origin-Verify: <secret>`, and a regional WAF rule on the ALB blocks requests missing it. Without this, attackers bypass CloudFront by hitting the ALB hostname directly — found via cert transparency logs or DNS history.
>
> For the team: WAF rules go through code review and GitOps. New rules go into `Count` mode first, then `Block` after validating they only match malicious traffic. A bad WAF rule in `Block` mode accidentally matching legitimate requests will take down the site."

---

## Self-test drills

### 1. Design the front door for a production EKS web app: CloudFront, Shield, WAF, ALB. What does each layer do and what order?

**Reference answer:** user → CloudFront (edge TLS + cache, absorbs volume) → WAF (HTTP inspection, blocks before origin, edge-evaluated) → Shield (L3/L4 DDoS, automatic, Standard free / Advanced $3k/mo) → ALB (origin, TLS termination, restricted to CF IPs only) → ingress controller → service → pod.

### 2. What is origin protection and how do you implement it?

**Reference answer:**
- Origin protection: ensuring the ALB can only be reached via CloudFront, not directly from the internet. Without it, WAF is bypassable — the attacker hits the ALB hostname directly.
- Method 1 (recommended): CloudFront injects a secret custom header (`X-Origin-Verify`). A regional WAF rule on the ALB blocks requests missing the header. Secret stored in Secrets Manager, rotated periodically. CloudFront supports two origin secrets simultaneously during rotation so there's no downtime.
- Method 2: CloudFront origin-facing managed prefix list (`pl-3b927c52`) added to the ALB's security group inbound rules. Simpler but large prefix list (~100 CIDRs) counts against SG rule limits.
- Both can be layered. Method 1 is the default for its simplicity and immunity to IP range changes.

### 3. When would you recommend Shield Advanced over Shield Standard?

**Reference answer:**
- Shield Standard (free) handles L3/L4 attacks automatically — SYN floods, UDP amplification. It's always on for CloudFront, ALB, EIP, Route 53.
- Shield Advanced ($3k/month base) is justified when: (a) you're a high-value DDoS target where downtime has significant revenue impact; (b) you've been targeted before; (c) you need SRT 24/7 access for real-time expert assistance; (d) cost protection matters — AWS credits your bill for DDoS-caused traffic spikes.
- For most teams: Shield Standard + WAF rate limiting handles the practical threat model. Shield Advanced is a business decision, not a purely technical one.

### 4. What's the difference between attaching WAF to CloudFront vs attaching it to the ALB?

**Reference answer:**
- WAF on CloudFront (scope=`CLOUDFRONT`, must be created in us-east-1): evaluated at the edge. Blocked requests never enter your VPC. Lowest cost per request. Works with all 600+ PoPs.
- WAF on ALB (scope=`REGIONAL`, created in the ALB's region): evaluated at the origin. Traffic already crossed the internet and entered your VPC before WAF evaluates it. More expensive. Necessary if you don't use CloudFront, or as a secondary defense (just the origin-header check rule).
- Best practice: primary WAF on CloudFront; optional secondary WAF on ALB for origin protection. Don't duplicate all rules on both layers.

---

## Further reading / watching

- **AWS docs — CloudFront Developer Guide**: "Using AWS WAF to control access to your content" and "Restricting access to an Application Load Balancer"
- **AWS docs — Shield User Guide**: the Standard vs Advanced comparison table is authoritative on features and cost
- **AWS docs — WAFv2 Developer Guide**: "AWS Managed rule groups" — the list of available groups and pricing
- **AWS blog — "How to protect a web application against DDoS attacks by using Amazon CloudFront and AWS WAF"** — end-to-end walkthrough with screenshots
- **AWS re:Invent — "AWS Shield: Protect your applications from DDoS attacks"** (search YouTube) — explains the SRT engagement model clearly

---

## The 4 dimensions (senior framing)

- **Tech**: The layered order matters — user → CF (edge TLS, cache) → WAF (HTTP rules) → Shield (L3/L4 DDoS) → ALB (origin, CF-locked). Origin protection is the most commonly missed gap. WAF rules have priorities — lower number evaluated first; rate-based rules at priority 1, OWASP managed groups at 10-20, geo block at 30+. WAF on CloudFront must be in `us-east-1` (CloudFront scope = global); WAF on ALB is regional.

- **People**: WAF rules can accidentally block legitimate traffic (false positives). New rules should go through `Count` mode first (log but don't block), then `Block` after validating CloudWatch WAF metrics show only malicious traffic. The team needs a runbook: "unexplained 403s — check WAF sampled requests in the console, check CloudWatch WAF metrics by rule name." Without this, a WAF misconfiguration will be blamed on the app, not WAF.

- **CI/CD**: CloudFront distribution config in Terraform (versioned, reviewed). WAF Web ACL rules in Terraform checked into Git. Use separate Terraform state for `us-east-1` resources (CloudFront cert + WAF ACL) vs the ALB region. Validate WAF rule changes in a staging CloudFront distribution first — production WAF changes in `Block` mode are high-risk. Shield Advanced enrollment via `aws_shield_protection` Terraform resource.

- **Operations**: CloudWatch metrics for WAF: `BlockedRequests`, `AllowedRequests` per rule. Alert on a sudden `BlockedRequests` spike (active attack) or sudden drop to zero (rule accidentally disabled). CloudFront metrics: `5xxErrorRate`, `CacheHitRate` — low cache hit on static paths means caching is misconfigured. Shield Advanced provides `DDoSDetected` CloudWatch event — configure SNS alarm. Runbook for "under DDoS": check Shield console for detection, inspect WAF blocked-request samples, engage SRT if Advanced, consider emergency geo block of source region.
