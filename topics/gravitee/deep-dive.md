# Gravitee — Deep Dive (the longer reference)

> Use this for depth on specific topics. The simpler `gravitee-simple.md` is the main study doc.

---

## 1. The category framing — THE most important thing to get right

The single biggest mistake: thinking Gravitee is "like Nginx but fancier" or "like Istio ingress." It is neither.

**Gravitee is an API Management platform.** That category includes Apigee (Google), Kong Konnect, AWS API Gateway, Azure API Management, MuleSoft, Tyk. They share a model Istio doesn't have.

### What APIM does that Istio doesn't

APIM treats **each API as a managed product** with a full lifecycle. The non-Istio concepts are:

- **APIs as products** with versioning, lifecycle states (draft → published → deprecated → archived)
- **Consumers / Applications** — registered identities that subscribe to your APIs (mobile apps, partners, internal teams)
- **Plans** — tiers like Free/Gold/Enterprise, each with its own auth scheme, rate limit, and SLA
- **Subscriptions** — a Consumer subscribes to an API's Plan, getting an API key or OAuth client
- **Developer Portal** — public-facing self-serve UI where external devs discover, document, test, and subscribe
- **Analytics per API / per Plan / per Consumer** — dashboards business owners look at, not just SREs

### What APIM and Istio share (the gateway primitive)

Both can do: L7 routing, JWT validation, rate limiting, OAuth2, mTLS upstream, traffic splitting, request/response transformation, observability.

So at the **gateway layer**, your Istio experience transfers directly. What's new is the **management layer above the gateway**.

### The real-world framing

A mature setup almost always has **both** an APIM tool (Gravitee/Kong/Apigee) and a service mesh (Istio) in production:

```
Internet
   ↓
CDN / WAF (CloudFront)
   ↓
Gravitee Gateway      ← APIs as products: Plans, API keys, OAuth via AM, per-consumer rate limits, analytics
   ↓
Istio Ingress GW      ← Cluster edge: mTLS termination, route into mesh
   ↓
Istio sidecars        ← East-west: mTLS, retries, circuit-breaking between services
   ↓
Microservices (EKS)
```

Gravitee handles **product-layer concerns** (who can call what API, under what plan, with which key). Istio handles **infrastructure-layer concerns** (how services talk to each other safely and reliably).

---

## 2. Architecture — the three Gravitee components

### a) Gravitee APIM (API Management)
The main product. Includes:
- **Gateway** (Java/Vert.x runtime) — handles actual API traffic, applies policies, forwards to backends.
- **Management API** (REST API) — control plane that stores API definitions, plans, subscriptions in a backing store.
- **Management UI** — admin web app for designing/publishing APIs.
- **Developer Portal** — separate web app, public-facing, where external developers self-serve.

### b) Gravitee AM (Access Management)
A separate product. Essentially an **OIDC / OAuth2 provider** — issues tokens, manages users, federates with external IdPs (Google, Okta, SAML), enforces MFA.
- Often paired with APIM (AM issues tokens, APIM validates them).
- Comparable to Keycloak or Auth0.

### c) Gravitee Cockpit
SaaS-only control plane for multi-environment Gravitee installs. Less commonly used at smaller scales.

### Architecture diagram

```
                       ┌──────────────────────────────┐
                       │  Management API + UI         │
                       │  (REST API + Angular UI)     │
                       └─────────────┬────────────────┘
                                     │
                       ┌─────────────▼────────────────┐
                       │  Backing store               │
                       │  - MongoDB (config)          │
                       │  - Elasticsearch (analytics) │
                       │  - Redis (rate-limit counters)│
                       └─────────────┬────────────────┘
                                     │ pulls API defs
                                     ▼
   Client ──HTTP──►   ┌──────────────────────────┐  ──►  Backend microservices
                      │  Gravitee Gateway        │       (could be Istio mesh,
                      │  (Java/Vert.x, runs in K8s)│       direct services, etc.)
                      └──────────────────────────┘

                       ┌──────────────────────────┐
                       │  Developer Portal        │      ◄── external devs
                       │  (separate web app)      │           browse + subscribe
                       └──────────────────────────┘
```

**Key SRE concerns:**
- Gateway is the hot data path — scale this for throughput.
- Management API is control plane — small, but goes down → no new API publishes (existing traffic unaffected, like Istio control plane).
- MongoDB holds API definitions; Elasticsearch holds analytics; Redis holds rate-limit counters.
- Gateway is stateless and horizontally scalable.

---

## 3. The five core concepts

### a) API
A managed product. Has:
- **Name + version** (e.g. `wallet-api v2`)
- **Backend (target)** — the actual service it proxies to
- **Listening path** — what clients hit
- **Lifecycle state** — draft / published / deprecated / archived
- **Policies** — the chain of things to do on requests/responses

### b) Plan
A "tier" of access to an API. Defines:
- **Authentication type** — Keyless / API key / JWT / OAuth2
- **Rate limit / quota** — e.g. 1000 req/min, 1M req/month
- **Validation** — extra policies that only apply to this plan
- **Publish state** — published, deprecated, closed

Example Plans for a `wallet-api`:
- **Public Beta** — keyless, 60 req/min
- **Internal** — JWT from internal IdP, unlimited
- **Partner Gold** — OAuth2, 10k req/min
- **Partner Enterprise** — mTLS client cert, dedicated quota

### c) Application (aka Consumer)
An identity that subscribes to APIs.
- A consumer-facing mobile app
- A partner integration
- An internal microservice

Each Application gets credentials when it subscribes to a Plan.

### d) Subscription
The link between an Application and an API Plan. "Application X subscribes to API Y's Plan Z."

When a request hits the gateway with an API key, Gravitee looks up which Subscription that key belongs to → which Plan → applies the Plan's rate limit and quotas.

### e) Policy
A unit of logic that runs in the request/response flow. Equivalent to **Envoy HTTP filter** or **Express middleware**.

---

## 4. The Policy Chain (the actual mechanism)

Every Gravitee API has a **flow** — an ordered list of policies applied in distinct phases:

```
┌──────────────────────────────────────────────────────────────┐
│                  Request from Client                         │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│   ON-REQUEST PHASE (before backend call)                     │
│   1. Authentication policy (api-key / jwt / oauth2)          │
│   2. Plan-selection (which Plan does this auth belong to?)   │
│   3. Rate-limit / Quota (Plan-specific)                      │
│   4. Request transforms (headers, body, query params)        │
│   5. Authorization (RBAC, claim checks)                      │
│   6. Cache lookup (return cached if hit)                     │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                  CALL BACKEND                                │
│   (apply LB, retries, circuit-breaker, timeout here)         │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│   ON-RESPONSE PHASE (after backend response)                 │
│   7. Response transforms (headers, body)                     │
│   8. Cache store (if cacheable)                              │
│   9. Analytics logging                                       │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                  Response to Client
```

### Why this matters

The interviewer wants to know that you understand:
1. **Policies are ordered** — order matters; auth before rate limit, rate limit before transform.
2. **There are distinct phases** — request, backend call, response.
3. **Plan-level vs API-level policies** — a Plan can add its own policies (e.g. stricter rate limit for the Free tier) on top of the API's base policies.

### Compared to Istio

In Istio, the analog is the **Envoy filter chain on a listener**. You configure it via `EnvoyFilter` or via `Gateway` + `VirtualService` + `AuthorizationPolicy`. Gravitee makes the chain explicit and **product-shaped**.

**Senior framing:** "Gravitee's policy chain is conceptually similar to Envoy's HTTP filter chain — ordered, phase-aware filters that pre-process requests, call backend, and post-process responses. The difference is that Gravitee's chain is **product-aware** — the API key or JWT identifies a Plan, which adds Plan-specific policies on top of the API's base flow. That product/consumer model has no direct equivalent in Istio."

---

## 5. The common policies in depth

### `api-key`
- Validates an API key from header (default `X-Gravitee-Api-Key`) or query param.
- Looks up the subscription → resolves which Plan → which Application.
- **Gotcha:** key rotation requires re-subscription; stagger via overlapping subscriptions.

### `jwt`
- Validates JWT signature against JWKS or static public key.
- Checks issuer, audiences, expiry, optionally specific claims.
- Extracts claims into context for downstream policies.
- **Istio analog:** `RequestAuthentication` + `AuthorizationPolicy`.
- **Gotcha:** JWKS caching — too short = hit JWKS every request; too long = key rotation breaks auth.

### `oauth2`
- Validates an OAuth2 access token by introspection or by JWT validation.
- Usually paired with Gravitee AM as the token issuer.
- **Gotcha:** introspection adds network hop per request — JWT-format tokens are faster.

### `rate-limit` and `quota`
- **Rate limit** = short-window (per-second/minute).
- **Quota** = long-window (per-day/month).
- Backed by **Redis** for global counters across gateway pods.
- **Gotcha:** Redis becomes critical path. Pick fail-open or fail-closed explicitly.

### `cache`
- Cache responses by key (URI, headers, query params).
- **Gotcha:** cache keys must include auth context — otherwise User A gets User B's data.

### `transform-headers` / `transform-body`
- Add/remove/set headers; rewrite paths; transform JSON ↔ XML.
- **Gotcha:** body transforms parse + serialize every request — measure latency impact.

### `assign-attributes`
- Sets variables in request context for later policies (e.g. extract claim from JWT, use in `dynamic-routing`).

### `groovy` / `javascript`
- Run custom code. Escape hatch.
- **Senior take:** "Only when there's no first-class policy that does what we need. Custom scripts are hard to test, version, and review."

### `dynamic-routing`
- Route to different backends based on request attributes.
- **Istio analog:** `VirtualService.http.match` with header/path conditions.

### `circuit-breaker`
- Stop calling a failing backend.
- **Istio analog:** `DestinationRule.outlierDetection` + `connectionPool`.

---

## 6. APIM vs AM

| | APIM | AM |
|---|---|---|
| **Purpose** | Manage APIs and their consumption | Manage identities and issue tokens |
| **Sits where** | In front of backend services | In front of login flows / token endpoints |
| **Validates** | API keys, plans, subscriptions, JWTs | User credentials, MFA, federation |
| **Issues** | Nothing (it's a gateway) | OAuth2 / OIDC tokens, refresh tokens |
| **Comparable to** | Kong, Apigee, AWS API GW | Keycloak, Auth0, Okta |

**They're complementary:** AM issues tokens to users/applications; APIM validates those tokens on every API call.

**You can run APIM without AM** — many shops use Auth0 / Okta / Keycloak instead.

---

## 7. Comparison to Kong and Apigee

Many JDs list Gravitee preferred but accept Kong or Apigee as substitutes.

| | Gravitee | Kong | Apigee |
|---|---|---|---|
| **Runtime** | Java/Vert.x | Lua on Nginx (OpenResty) | Java (Edge) or Cloud (Google) |
| **Origin** | French open source (2015) | Open source (2015), now closed-source enterprise | Google Cloud (acquired 2016) |
| **Config model** | API definition with policy chain | Service + Route + Plugin | API Proxy with policy flows |
| **Custom logic** | Groovy / JS / Java plugins | Lua plugins / Custom plugins | JavaScript / Python / Java |
| **Identity** | Gravitee AM (separate) | Konnect (cloud) or external | Apigee + Cloud Identity |
| **Deployment** | Self-hosted (K8s, VM) | Self-hosted or Konnect SaaS | Mostly SaaS |
| **Strengths** | Open source, on-prem, integrated AM | Best for cloud-native, K8s-friendly | Enterprise features, Google ecosystem |
| **Weaknesses** | Smaller community than Kong | OSS lacks UI/portal | Vendor lock-in |

---

## 8. Deployment and operations

### Components running in K8s

```
namespace: gravitee
  - deployment/gravitee-gateway       (the hot data path)
  - deployment/gravitee-mgmt-api      (control plane)
  - deployment/gravitee-mgmt-ui       (admin UI)
  - deployment/gravitee-portal-ui     (developer portal)

dependencies:
  - MongoDB    (API config, subscriptions, applications, plans)
  - Elasticsearch (analytics, logs, audit trail)
  - Redis      (rate limit counters, cache)
```

### Operational concerns

1. **Scaling the gateway** — stateless, horizontally scale via HPA. JVM tuning (heap, GC) matters at high throughput.
2. **MongoDB** — primary persistence for API definitions. Replicated, backed up.
3. **Elasticsearch** — high write volume; manage retention, index rollover, ILM policies. Costly at scale.
4. **Redis** — Sentinel or Cluster + AOF persistence.
5. **GitOps** — Gravitee has a **Kubernetes operator** with CRDs (`ApiDefinition`, `ApiResource`). ArgoCD-friendly.

### Gravitee Kubernetes Operator

```yaml
apiVersion: gravitee.io/v1alpha1
kind: ApiDefinition
metadata:
  name: wallet-api
spec:
  name: "Wallet API"
  version: "v2"
  proxy:
    virtual_hosts: [{ path: "/v2/wallet" }]
    groups:
    - endpoints:
      - name: backend
        target: http://wallet-svc.payments.svc.cluster.local:8080
  plans:
  - name: "Internal"
    security: JWT
    flows:
    - methods: [GET, POST]
      pre:
      - policy: rate-limit
        configuration: { rate: { limit: 1000, periodTime: 1, periodTimeUnit: MINUTES } }
```

**Senior framing:** "I'd manage APIs as YAML via the Gravitee K8s operator, kept in Git and applied via ArgoCD."

---

## 9. The honest "have you used Gravitee?" answer

> "I haven't operated Gravitee in production. My API-gateway experience is with **Istio ingress gateway**, which covers the gateway primitives — routing, JWT validation, rate limiting, mTLS upstream, traffic splitting, observability. The conceptual gap to Gravitee is the **API-Management layer above the gateway**: APIs as products with Plans and Subscriptions, the Developer Portal, AM for OAuth2 token issuance, and the policy chain for product-layer transforms. I understand the model — it's the same model as Kong and Apigee — and I've read the Gravitee docs in prep for this conversation. I'd expect to be productive on day-to-day Gravitee config within a sprint; the gateway-side concepts transfer directly. The deeper operational reps — Gravitee AM federation, Cockpit multi-cluster, custom policy development — I'd learn on the job."

---

## 10. Likely interview questions

### Q: "What's the difference between an API gateway and a service mesh?"
> "Different concerns at different layers. **API gateway = product layer, north-south traffic**: APIs as products, consumers, plans, subscriptions, public-facing concerns. **Service mesh = infrastructure layer, east-west traffic**: service-to-service communication, mTLS, retries, circuit breaking, identity-based authz. Mature platforms run both."

### Q: "How would you handle rate limiting for a partner API?"
> "Three layers depending on the SLA: **CDN/WAF layer** for crude DDoS protection; **APIM layer (Gravitee)** for per-Plan / per-Subscription quotas, backed by Redis — this is where the SLA contract is enforced; **mesh layer (Istio)** for local rate limits to protect individual upstream pods from overload — defense in depth."

### Q: "How do you secure an API exposed to mobile apps?"
> "Layered: (1) Mobile gets OAuth2 token from Gravitee AM or external IdP. (2) Mobile sends requests to Gravitee with token; gateway validates. (3) Per-Plan rate limit via subscription. (4) Gateway forwards with mTLS upstream. (5) Inside mesh, Istio enforces service identity via SPIFFE certs. (6) Sensitive operations go through ext_authz to OPA for business-logic authz. (7) Fronted by CloudFront with WAF for OWASP + Shield for DDoS."

### Q: "An API consumer reports 429 errors. Walk me through debugging."
> "Step 1: Confirm Gravitee returned it — `X-Rate-Limit-Remaining: 0` header. Step 2: Identify Subscription via their API key. Step 3: Check Plan rate limit. Step 4: Gateway analytics for their consumer ID — actually exceeding rate, or stale counter? Step 5: If legitimate excess, higher Plan / waiver; if bug, check Redis counters + gateway logs. Senior reflex: spike across many consumers = Redis health, not consumer behavior."

### Q: "How would you canary-deploy a new version of an API?"
> "Two patterns. (1) **Gateway-level traffic split** via Gravitee's `dynamic-routing` policy with weighting. (2) **Behind the gateway, in the mesh**: one Gravitee API pointing at a K8s Service that fronts both v1 and v2 pods, Istio `VirtualService` for the 5%/95% split. Option 2 more flexible (Istio + Argo Rollouts/Flagger automation). Option 1 more visible at the API product layer — useful for per-Plan dark launches."

### Q: "How do you do JWT auth in Gravitee, and how does that differ from Istio?"
> "In Gravitee, API has Plan with `security: JWT`, gateway's `jwt` policy validates against JWKS. Subscriptions are implicit per JWT subject. In Istio: `RequestAuthentication` (validation contract) + `AuthorizationPolicy` (enforces, checks claims). Big difference: **Gravitee couples JWT to Plan + Subscription** — token's `sub` maps to an Application with quota and analytics view. Istio has no consumer/plan concept."

---

## 11. Troubleshooting basics

### Where do logs go?

| Log source | Where |
|---|---|
| Gateway access logs | stdout + optionally Elasticsearch via `logging` policy |
| Policy execution errors | Gateway logs (look for stack traces) |
| Management API errors | Management API pod logs |
| Backend errors | Backend pod logs |
| Analytics / metrics | Elasticsearch + APIM analytics UI |

### Common failure modes

1. **All API calls 401** — JWKS not reachable. Check egress NetworkPolicy and gateway logs.
2. **429s spiking across consumers** — Redis hiccup or counter desync.
3. **Sync lag between UI changes and gateway** — Gateway polls Management API periodically. Restart pods or shorten poll interval.
4. **Custom Groovy/JS policy throws** — 500. Check policy logs.
5. **Analytics dashboards empty** — Elasticsearch down or out of disk.

### Useful Gravitee gateway endpoints

```bash
curl localhost:18082/_node             # node info
curl localhost:18082/_node/health      # health check (K8s probes)
curl localhost:18082/_node/monitor     # JVM metrics
```

---

## 12. War stories you could borrow / adapt

### Story: rate limiting + Redis dependency
> "We rolled out per-consumer rate limiting backed by Redis. About a month in we had a brief Redis network blip — gateways failed open by default, so for ~90 seconds we served unlimited traffic to all consumers. We changed the policy to fail-closed for paid Plans and fail-open only for Free tier. Lesson: every shared dependency in a gateway needs an explicit failure-mode policy, picked deliberately rather than defaulted."

### Story: custom policy gone wrong
> "Someone wrote a custom Groovy policy to extract a tenant ID. Worked in dev. In prod under load it threw NullPointerException on requests with unusual paths, returning 500. No test for the policy because not in any CI pipeline. We pulled it, replaced with declarative `transform-headers`, instituted a rule that all custom policies need CI coverage + load tests. Lesson: the escape hatch is a tax."

### Story: API key rotation
> "Partner leaked their API key on GitHub. Needed rotation without downtime. Naive replace would have 401'd until partner deployed new key. Instead: parallel subscription with new key, 24h overlap, then close old. Became our standard pattern."

---

## 13. Mnemonics

- **Five core concepts:** **A-P-A-S-P** → **A**PI, **P**lan, **A**pplication, **S**ubscription, **P**olicy
- **Three Gravitee products:** **APIM, AM, Cockpit**
- **Backing stores:** **M-E-R** → **M**ongoDB (config), **E**lasticsearch (analytics), **R**edis (rate-limit counters)
- **Policy phases:** **req → backend → res** → On-Request, Backend Call, On-Response

---

## 14. What NOT to say

- ❌ "Gravitee is like Nginx" — it's an API Management platform.
- ❌ "Gravitee replaces Istio" — different layers; use both.
- ❌ "I'll just write a Groovy policy" — escape hatch, not the default.
- ❌ "Rate limiting works the same as Istio's" — Istio is per-pod or RLS; Gravitee is per-Plan/per-Subscription with global Redis counters.
- ❌ Claim production experience you don't have.

---

## 15. Final readiness check

If you can:
- Explain the **category difference** between APIM and service mesh in 30 seconds.
- Name the **5 core concepts** and what each does.
- Describe the **3 components** (APIM, AM, Cockpit) and **3 backing stores** (Mongo, ES, Redis).
- Walk through the **policy chain** phases.
- Give the **honest "I haven't used Gravitee" answer** without sounding lost.
- Compare to **Kong and Apigee** at a high level.

…you're at 80-85% without hands-on. The honest framing closes the rest.
