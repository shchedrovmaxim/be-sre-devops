# ACME challenge types — the deep-dive

> **Goal**: by the end you can answer the killer question — **"When would you use DNS-01 instead of HTTP-01, and what infra setup does each need?"** — naming the exact mechanics of each challenge type, the wildcard cert forcing condition, the Route 53 + IRSA setup, and the failure modes that catch teams off guard.

> Start with [acme-challenges-simple.md](./acme-challenges-simple.md) if you haven't read it. The two-proof (front-door vs registry) analogy is the spine of everything here.

---

## The senior framing — challenge type determines your infra dependencies

Most teams start with HTTP-01 because it's the path of least resistance: no DNS API credentials, no IAM policies to set up, just "make port 80 reachable." It works fine for public clusters.

The problem surfaces when:
- You need a wildcard cert (`*.example.com`). HTTP-01 is not allowed by spec.
- You move to a private cluster with no public ingress. HTTP-01 physically can't work.
- You're issuing internal mTLS certs for services that aren't public-facing. HTTP-01 is the wrong tool entirely.

Understanding which challenge type to use — and what infra each needs — is the kind of question that distinguishes someone who has actually operated cert-manager from someone who followed a tutorial.

---

## HTTP-01 — exact mechanics

### What happens

1. cert-manager's ACME controller creates a temporary `Service` and `Pod` (the "solver pod") in the namespace of the `Certificate`.
2. It creates a temporary `Ingress` (or `HTTPRoute` in newer setups) routing requests for `/.well-known/acme-challenge/<token>` to the solver pod.
3. cert-manager notifies Let's Encrypt that the challenge is ready.
4. Let's Encrypt sends HTTP GET requests to `http://<domain>/.well-known/acme-challenge/<token>` from **multiple vantage points** (geographically distributed — this is a BGP hijacking mitigation).
5. The solver pod returns the `keyAuthorization` string.
6. Let's Encrypt confirms. cert-manager tears down the solver pod, service, and ingress.

### What you need

- **Port 80 reachable from Let's Encrypt's servers.** Not port 443 only. Let's Encrypt's HTTP-01 explicitly uses HTTP, not HTTPS. Many teams have an HTTPS-only ingress that redirects 80 → 443; this breaks HTTP-01. The redirect happens before the challenge can be served.
- **Your ingress controller must forward `.well-known/acme-challenge/` correctly.** The solver pod path must not be caught by other ingress rules. cert-manager creates the ingress rule automatically, but if you have a catch-all ingress rule, it can intercept the challenge before the solver.
- **The solver ingress class must match.** The `http01.ingress.class` field in your `ClusterIssuer` must match the ingress controller class in your cluster.

### Common failure modes

**The HTTP → HTTPS redirect trap**:
```yaml
# This annotation breaks HTTP-01
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```
cert-manager creates its challenge ingress without this annotation, but your ingress controller may have a global redirect policy. Let's Encrypt sends HTTP; gets a 301 redirect to HTTPS; follows it (it does follow redirects); but the new HTTPS URL requires a valid cert... which you don't have yet. Deadlock.

Fix: configure your ingress controller to serve `.well-known/acme-challenge/` over HTTP before redirecting. nginx-ingress supports per-path annotation overrides.

**Firewall blocks Let's Encrypt's source IPs**:
Let's Encrypt uses multiple vantage points from various ISP ranges. You can't whitelist by IP — the vantage points change. If you have a WAF or security group that only allows traffic from known sources, HTTP-01 won't work. Use DNS-01.

**Multiple solver pods (too many certs at once)**:
cert-manager creates solver pods per-challenge. If you're issuing 20 certs simultaneously (new cluster, CI environment), you'll have 20 solver pods and 20 ingress rules simultaneously. This is generally fine, but can hit pod-per-namespace limits in tightly constrained clusters.

### cert-manager ClusterIssuer configuration for HTTP-01

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
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx    # must match your ingress controller class
            # ingressTemplate allows overriding ingress annotations per-challenge
            ingressTemplate:
              metadata:
                annotations:
                  nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

---

## DNS-01 — exact mechanics

### What happens

1. cert-manager calls your DNS provider's API to create a TXT record:
   ```
   _acme-challenge.<domain>  TXT  "<keyAuthorization-sha256>"
   ```
   The value is the SHA-256 of the key authorization, base64url-encoded.
2. cert-manager optionally waits for DNS propagation (the `dnsPropagationPolicy` field).
3. cert-manager notifies Let's Encrypt.
4. Let's Encrypt queries DNS resolvers for the TXT record. It uses **multiple vantage points** (different resolvers) — same BGP hijacking mitigation.
5. If the TXT record matches, challenge valid.
6. cert-manager deletes the TXT record.

### What you need

- **API access to your DNS provider.** Credentials (API token or IAM role) that allow creating and deleting TXT records on the challenge domain.
- **No requirement for port 80 or 443.** The only network path is from Let's Encrypt's resolvers to your DNS zone — which is always public for any internet-accessible domain.
- **DNS propagation patience.** The TXT record must propagate to the resolver Let's Encrypt uses before it checks. If your DNS TTL is high (300s+) and the record just changed, Let's Encrypt might query before it propagates.

### Route 53 + IRSA — the EKS pattern

In EKS, the best practice is IRSA (IAM Roles for Service Accounts) — cert-manager's ServiceAccount gets an IAM role annotation; the pod gets a projected token; AWS exchanges it for temporary credentials. No long-lived API keys in Secrets.

**Step 1 — IAM policy** (least privilege for Route 53 DNS-01):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/<HOSTED_ZONE_ID>"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
```

**Step 2 — ServiceAccount annotation** (cert-manager's SA):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-manager
  namespace: cert-manager
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/cert-manager-route53
```

Or via Helm values when installing cert-manager:
```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/cert-manager-route53
```

**Step 3 — ClusterIssuer with Route 53**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: Z1234567890ABC   # optional; speeds up lookup
```

cert-manager will use the IRSA credentials automatically (the projected token is mounted into the pod by EKS's admission webhook).

### Cloudflare pattern (simpler setup, commonly used for small teams)

```yaml
# 1. Create Cloudflare API token Secret
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
stringData:
  api-token: "your-cloudflare-api-token"   # scoped to: Zone:DNS:Edit for the specific zone

---
# 2. Reference in ClusterIssuer
spec:
  acme:
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

### DNS propagation gotcha

cert-manager's default `dnsPropagationPolicy` is `Sync` — it polls the authoritative nameservers for the zone before notifying Let's Encrypt. This avoids Let's Encrypt querying before the record is live, but adds latency (can be 1–5 minutes per cert depending on your authoritative NS response time).

For fast-moving environments (many certs, CI), you can set `dnsPropagationPolicy: None` to skip the wait — but you risk Let's Encrypt querying before the TXT record is live, causing a failed challenge and a backoff.

**Split-horizon DNS gotcha**: if your cluster uses internal DNS (CoreDNS serving a private zone for the same domain), and Let's Encrypt queries the public DNS, the internal zone might shadow the TXT record. This doesn't affect DNS-01 (Let's Encrypt always queries public resolvers, not your internal ones), but it can confuse debugging if you're testing DNS locally.

---

## TLS-ALPN-01 — when it's actually useful

TLS-ALPN-01 is the third challenge type, specified in RFC 8737. cert-manager supports it but it's rarely configured.

**How it works**: Let's Encrypt connects to port 443 of the domain and initiates a TLS handshake with a special ALPN extension (`acme-tls/1`). The server responds with a self-signed cert that embeds the challenge token as a Subject Alternative Name. Let's Encrypt verifies the token.

**When it's useful**:
- Port 80 is blocked (firewall policy, or NAT that doesn't allow port 80 traffic)
- You can't modify DNS (no API access to your provider)
- Port 443 is already reachable

**Limitations**:
- cert-manager's TLS-ALPN-01 solver is less mature than HTTP-01/DNS-01
- Wildcard certs not supported (same reason as HTTP-01 — can't prove zone control)
- The TLS termination on port 443 must pass through to the challenge handler — tricky with edge CDNs or proxies that terminate TLS

In practice: if you can reach port 443, you can usually reach port 80 or use DNS-01. TLS-ALPN-01 is a niche fallback.

---

## Challenge type decision matrix

| Scenario | Best challenge type | Why |
|---|---|---|
| Public cluster, ingress on port 80 | HTTP-01 | Simplest; no DNS API needed |
| Wildcard cert (`*.example.com`) | DNS-01 | **Required by spec** |
| Private cluster (no public ingress) | DNS-01 | HTTP-01 physically can't work |
| Cluster behind WAF with IP allowlist | DNS-01 | Can't allowlist Let's Encrypt source IPs |
| EKS + Route 53 | DNS-01 + IRSA | Clean IAM integration; works for wildcards |
| Cloudflare DNS | DNS-01 + API token | Simple Cloudflare token integration |
| Port 80 blocked, port 443 open, no DNS API | TLS-ALPN-01 | Niche fallback |
| Multi-tenant cluster, different teams own different domains | HTTP-01 per-domain or DNS-01 per-zone | Mix using solver `selector` |

### Mixing challenge types with selector

cert-manager `ClusterIssuer` supports multiple solvers with `selector` to route different `Certificate` requests to different challenge types:

```yaml
solvers:
  # Wildcard certs use DNS-01
  - selector:
      dnsZones:
        - "example.com"
    dns01:
      route53:
        region: us-east-1
        hostedZoneID: Z1234567890ABC
  # Everything else uses HTTP-01
  - selector: {}
    http01:
      ingress:
        class: nginx
```

---

## The interview answer in 60 seconds

> "HTTP-01 is the default for public clusters. cert-manager spins up a temporary solver pod and ingress rule, Let's Encrypt fetches a token from `/.well-known/acme-challenge/` over port 80. Simple, no DNS credentials needed. Doesn't work if port 80 isn't publicly reachable or if you need a wildcard cert.
>
> DNS-01 is required for wildcards — RFC 8555 mandates it. It's also the right choice for private clusters with no public HTTP endpoint. cert-manager calls the DNS provider API to create a TXT record on `_acme-challenge.<domain>`; Let's Encrypt queries that record from public resolvers. Infra requirement: DNS API credentials. In EKS, the best pattern is IRSA — cert-manager's ServiceAccount gets an IAM role annotation, the pod gets temporary credentials from STS, no long-lived secrets needed.
>
> Two gotchas worth naming: the HTTP redirect trap (HTTP-01 breaks if your ingress globally redirects HTTP → HTTPS before the challenge can be served) and DNS propagation delay (DNS-01 requires the TXT record to propagate before Let's Encrypt checks, adding 30–120 seconds per cert)."

---

## Self-test drills

### 1. When would you use DNS-01 instead of HTTP-01?

**Reference answer:**
- Wildcard certs (`*.example.com`) — DNS-01 required by RFC 8555.
- Private clusters without a public-facing port 80.
- Clusters behind WAFs or firewalls that can't allowlist Let's Encrypt's source IPs.
- Cert issuance for services that don't have a web server running (internal services, databases).
- Whenever you prefer to avoid port 80 exposure entirely.

### 2. Walk me through setting up DNS-01 with Route 53 in EKS.

**Reference answer:**
1. Create an IAM policy with `route53:ChangeResourceRecordSets`, `route53:ListResourceRecordSets`, `route53:GetChange` on the target hosted zone.
2. Create an IAM role with a trust policy allowing cert-manager's ServiceAccount to assume it via OIDC.
3. Annotate cert-manager's ServiceAccount with `eks.amazonaws.com/role-arn`.
4. Configure `ClusterIssuer` with `dns01.route53.region` and optionally `hostedZoneID`.
5. cert-manager automatically uses the projected ServiceAccount token for AWS API calls.
6. The Route 53 API creates `_acme-challenge.example.com TXT <token>`, Let's Encrypt verifies, cert issued.

### 3. Your HTTP-01 challenge keeps failing. What do you check?

**Reference answer:**
1. `kubectl describe challenge -n <namespace>` — read the exact error.
2. Check if port 80 is actually reachable: `curl -v http://<domain>/.well-known/acme-challenge/test` from outside the cluster.
3. Check if the ingress controller is globally redirecting HTTP → HTTPS. If so, configure `ssl-redirect: "false"` for the `/.well-known/acme-challenge/` path, or use the `ingressTemplate` in the `ClusterIssuer` solver.
4. Check if a WAF or security group is blocking port 80.
5. Check if the solver ingress class matches your ingress controller (`ingress.class` in the ClusterIssuer solver).
6. If all else fails: switch to DNS-01.

### 4. Why can't HTTP-01 issue wildcard certs?

**Reference answer:**
- HTTP-01 proves control of a specific hostname: the challenge is served at `http://api.example.com/.well-known/acme-challenge/<token>`.
- A wildcard `*.example.com` represents all possible subdomains — not a single hostname.
- There's no single URL Let's Encrypt can check to verify control of all possible subdomains.
- DNS-01 proves zone control: only the DNS zone owner can create a TXT record on `_acme-challenge.example.com`. Zone control implies control of all subdomains.
- RFC 8555 (section 10.4) explicitly states: "The identifier value in a 'dns' type identifier must be unique; it MUST NOT match a wildcard. [...] Wildcard domain names [...] can only be validated with the dns-01 challenge."

---

## Further reading / watching

- **Let's Encrypt challenge types** — [letsencrypt.org/docs/challenge-types](https://letsencrypt.org/docs/challenge-types/). The canonical explanation of each type.
- **RFC 8555 section 8** — Identifier Validation Challenges. The spec for HTTP-01 (8.3) and DNS-01 (8.4). Short and readable.
- **RFC 8737** — TLS-ALPN-01. The TLS-ALPN-01 specification. Worth skimming to understand what it does.
- **cert-manager DNS01 solvers** — [cert-manager.io/docs/configuration/acme/dns01](https://cert-manager.io/docs/configuration/acme/dns01/). Every supported DNS provider with config examples (Route 53, Cloudflare, Google Cloud DNS, Azure DNS, etc.).
- **cert-manager Route 53 tutorial** — [cert-manager.io/docs/configuration/acme/dns01/route53](https://cert-manager.io/docs/configuration/acme/dns01/route53/). Includes the exact IAM policy needed.

---

## The 4 dimensions (senior framing)

- **Tech**: HTTP-01 requires port 80 public and ingress controller cooperation; DNS-01 requires DNS API credentials and has a propagation delay; TLS-ALPN-01 is a niche port-443-only option; wildcard certs mandate DNS-01; solver selectors allow mixing types per domain zone.
- **People**: the ingress HTTP-redirect trap is the most common "I followed the tutorial and it doesn't work" problem. Document it in your internal runbook. New engineers assume that if HTTPS works, cert-manager will work — but HTTP-01 needs port 80 specifically. Document which `ClusterIssuer` to use and why: staging vs production, HTTP-01 vs DNS-01.
- **CI/CD**: ephemeral CI environments must use staging issuer and should prefer DNS-01 (since CI clusters often don't have predictable public IP or stable port 80). IRSA credentials for Route 53 should be provisioned by the platform team's IaC (Terraform) — not manually. The IAM policy is narrow: only the specific hosted zone, only TXT record operations.
- **Operations**: DNS-01 challenges occasionally time out due to slow DNS propagation. Monitor `Challenge` objects stuck in `pending` state for > 5 minutes — that's the alert signal. The fix is usually propagation lag (wait) or credentials issue (check the dns01 solver pod logs). For Route 53 via IRSA: check the EKS pod identity webhook is running and the ServiceAccount annotation is correct before debugging cert-manager itself.
