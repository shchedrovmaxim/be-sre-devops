# Istio Interview Prep — Senior SRE Reference

> Consolidated study notes for senior SRE interviews where Istio comes up.
> Read top to bottom once, then drill the bold lines, tables, and Part 4 Q&A until they're automatic.

---

## TL;DR — the one-paragraph mental model

Istio is a **service mesh**. It puts a small proxy (**Envoy**) next to every pod (sidecar) and another at the cluster edge (ingress gateway). All your service-to-service traffic flows through these proxies. The proxies are configured centrally by a control plane (**istiod**) that watches Kubernetes and pushes config to every Envoy via gRPC (**xDS**). The result: timeouts, retries, circuit-breaking, load-balancing, mTLS encryption with identity, rate-limiting, observability, and routing all happen at the **infrastructure layer** instead of in your app code. You configure them once via YAML CRDs (`Gateway`, `VirtualService`, `DestinationRule`, `ServiceEntry`, `PeerAuthentication`, `AuthorizationPolicy`), they apply to every service in the mesh.

---

# PART 1 — THE BIG PICTURE

## 1. Architecture

```
   Control Plane                Data Plane
   ┌─────────┐  xDS (gRPC)   ┌──────────────┐
   │ istiod  │ ─────────────►│ Envoy sidecar│ in every pod
   │ (one    │               │ Envoy gateway│ at cluster edge
   │ process)│               └──────────────┘
   └─────────┘
```

- **Control plane = `istiod`** (one process; historically Pilot + Citadel + Galley consolidated in 1.5).
- **Data plane = Envoy** (same binary used as sidecar and as gateway pod).
- **xDS** = config-distribution protocol over gRPC: LDS (listeners), CDS (clusters), EDS (endpoints), RDS (routes), SDS (secrets/certs).
- **Sidecar injection** = MutatingAdmissionWebhook at pod creation; triggered by namespace label `istio-injection=enabled` or pod annotation `sidecar.istio.io/inject=true`.

---

## 2. The core CRDs — get the responsibilities crisp

| CRD | Side | One-line responsibility | Mental shortcut |
|---|---|---|---|
| **`Gateway`** | edge | Bind host + port + TLS to the edge Envoy | "Is the front door open?" |
| **`VirtualService`** | request | Routing rules: match request → destination, with retries/timeouts/mirror/fault | "Where does the request go?" |
| **`DestinationRule`** | upstream | Define subsets + LB algorithm + connection pool + outlier detection + upstream mTLS | "How do we talk to the place it's going?" |
| **`ServiceEntry`** | registry | Add an external host to mesh registry so routing/policy/metrics apply | "Is this external thing visible to the mesh?" |
| `Sidecar` | scoping | Limit which services a sidecar knows about (perf optimization at scale) | |
| `PeerAuthentication` | mTLS | Inbound mTLS mode (STRICT/PERMISSIVE/DISABLE) | |
| `RequestAuthentication` | AuthN | Validate JWT (issuer + JWKS) | |
| `AuthorizationPolicy` | AuthZ | ALLOW/DENY/CUSTOM/AUDIT rules on principals, paths, methods, headers | |

**Burned-in answer: `VS` is the request side, `DR` is the upstream side. `Gateway` opens the door. `ServiceEntry` makes external hosts mesh-visible.**

---

## 3. App-level → Istio-level pattern mapping

The single most important table to memorize.

| What you did in code | Istio equivalent | Key resource |
|---|---|---|
| `axios timeout: 2000` | Mesh-wide HTTP timeout | `VirtualService.http.timeout` |
| `axios-retry` config | Mesh-wide retry policy | `VirtualService.http.retries` |
| `resilience4j CircuitBreaker` | Eject bad endpoints + cap concurrency | `DR.outlierDetection` + `DR.connectionPool` |
| `express-rate-limit` | Per-pod or global rate limit | `EnvoyFilter` (local) / RLS (global) |
| TLS client cert config | Auto-issued workload identity certs (SPIFFE) | `PeerAuthentication` |
| Eureka + Ribbon | Per-request L7 load balancing | `DestinationRule.loadBalancer` |
| OpenTelemetry SDK | Auto-emitted spans (app still propagates trace headers) | Built-in + `Telemetry` CR |
| `prom-client` HTTP metrics | Auto RED metrics per service | Built-in + `Telemetry` CR |
| `process.env.WALLET_URL` | DNS via K8s + ServiceEntry for external | `ServiceEntry` |
| Feature flags for routing | Weighted/header-based traffic split | `VS` + `DR subsets` |

**Big-picture pitch:** Istio hoists these concerns out of app code into infrastructure, with mesh-wide consistency and zero-app-change rollout.

---

## 4. Timeouts and retries

```yaml
http:
- timeout: 2s
  retries:
    attempts: 3
    perTryTimeout: 500ms
    retryOn: gateway-error,connect-failure,reset
```

- **Timeout = total request time including retries**, not per-attempt.
- **`perTryTimeout` < `timeout`** must hold or retries can't fire.
- **Never `retryOn: 5xx`** — retry storms can self-DDoS a half-dead service.
- **Safe-to-retry only**: `gateway-error` (502/503/504), `connect-failure` (TCP never opened), `reset` (conn dropped). These all mean "the request never reached the app" — replaying is safe.

---

## 5. Circuit breaking — Istio doesn't have a CRD called "circuit breaker"

Two `DestinationRule` mechanisms together = circuit breaker behavior:

### Mechanism 1 — `outlierDetection` (eject bad endpoints)
```yaml
trafficPolicy:
  outlierDetection:
    consecutive5xxErrors: 5     # N failures in a row → eject
    interval: 30s
    baseEjectionTime: 30s
    maxEjectionPercent: 50      # never eject >50%
    minHealthPercent: 50        # stop ejecting if <50% healthy
```
Per-endpoint passive health check. Equivalent to "open circuit on this specific pod."

### Mechanism 2 — `connectionPool` (fail fast on overload)
```yaml
connectionPool:
  tcp:  { maxConnections: 100 }
  http: { http1MaxPendingRequests: 50, http2MaxRequests: 1000, maxRequestsPerConnection: 10 }
```
When limits hit → Envoy returns 503 immediately instead of queueing. Protects upstream from being buried.

**Burned-in answer:** "Istio doesn't have a single circuit-breaker resource. `outlierDetection` ejects unhealthy endpoints, `connectionPool` fails fast on overload. Together they cover what resilience4j gives you, but more granular — per-endpoint rather than per-service."

**Trap to know:** Aggressive outlier detection + high retry count = catastrophic. Brief blip → ejection → survivors crushed → all ejected → `UH` for everyone. `minHealthPercent` is the safety rail.

---

## 6. mTLS — the three modes and migration path

| Mode | Behavior |
|---|---|
| `STRICT` | Reject anything not mTLS |
| `PERMISSIVE` | Accept both mTLS and plaintext (for migration) |
| `DISABLE` | Plaintext only |

**Migration:** PERMISSIVE mesh-wide → inject sidecars everywhere → verify all traffic is actually mTLS via metrics → flip to STRICT namespace-by-namespace.

**Cert chain:** istiod is the CA. Issues SPIFFE x.509 certs (`spiffe://cluster.local/ns/<ns>/sa/<sa>`) to each workload, delivered via SDS, in-memory only, rotated ~24h.

**The real value isn't encryption — it's identity.** The cert proves which ServiceAccount is calling, so AuthorizationPolicy can reference it:
```yaml
rules:
- from:
  - source: { principals: ["cluster.local/ns/frontend/sa/frontend-sa"] }
```

**Classic trap:** `STRICT` breaks anything without a sidecar (non-mesh CronJobs, Prometheus scraping a non-injected pod). Solution: per-port exemptions or scrape via Envoy's merged metrics port (15020).

**Another trap:** `PeerAuthentication: STRICT` (inbound on server) but `DestinationRule.tls: DISABLE` (outbound on client) = handshake fails. Both sides must agree.

---

## 7. The 503 response flag zoo

You **will** be handed an access log line. Memorize:

| Flag | Meaning | Most common cause |
|---|---|---|
| **`UH`** | No Healthy Upstream | Zero endpoints, or outlier-detection ejected them all |
| **`UF`** | Upstream connection Failure | TCP couldn't connect (NetworkPolicy block, DNS, no endpoints) |
| **`UC`** | Upstream Connection terminated | Upstream closed mid-response (app crash, idle-timeout mismatch) |
| **`NR`** | No Route configured | VirtualService doesn't match request (host/port mismatch) |
| **`URX`** | Retry limit eXceeded | Retried N times, all failed |
| **`UT`** | Upstream request Timeout | Hit `timeout` value in VS |
| **`DC`** | Downstream Connection terminated | Client gave up |
| `-` (no flag) | Backend returned 5xx itself | App is actually broken; Envoy just relayed |

### Debugging `UH` flow
1. `istioctl proxy-config endpoints <pod>` — are there endpoints? Healthy?
2. If endpoints exist but ejected — outlier detection. Check `curl localhost:15000/stats | grep outlier_detection.ejections_active` and app logs.
3. If no endpoints — the K8s Service has no pods matching selector (`kubectl get endpoints <svc>`) or wrong subset labels.

---

## 8. Reading an Envoy access log line

```
[ts] "GET /v2/balance HTTP/1.1" 503 UH "-" "-" 0 91 12 - "10.0.3.42" "Mobile-Client/3.42"
"req-id" "api.example.com" "10.0.5.11:8080"
outbound|8080||wallet-svc.payments.svc.cluster.local
10.0.5.10:54321 10.0.5.11:8080 10.0.3.42:0 - default
```

Key fields: **response code**, **response flag** (UH/UF/NR/etc.), **duration ms**, **upstream cluster** (format `outbound|port|subset|fqdn` — empty subset means no DR subset matched), **upstream host** (`-` if never picked).

If you see **`BlackHoleCluster`** or **`PassthroughCluster`** as the upstream cluster — Envoy had no config for that destination (no ServiceEntry, fell through to default outbound policy).

---

## 9. Traffic management — canary, mirror, header-routing

### Canary (weighted)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  hosts: [wallet-svc]
  http:
  - route:
    - { destination: { host: wallet-svc, subset: stable }, weight: 95 }
    - { destination: { host: wallet-svc, subset: canary }, weight: 5 }
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
spec:
  host: wallet-svc
  subsets:
  - { name: stable, labels: { version: v1 } }
  - { name: canary, labels: { version: v2 } }
```

### Mirror (shadow)
```yaml
mirror: { host: wallet-svc, subset: canary }
mirrorPercentage: { value: 10.0 }
```
Fire-and-forget; response discarded. **Never mirror writes** — you'll double-charge users.

### Header-based dark launch
```yaml
- match: [{ headers: { x-user-tier: { exact: beta } } }]
  route: [{ destination: { subset: canary } }]
- route: [{ destination: { subset: stable } }]
```

### Progressive delivery
Istio shifts traffic; **Argo Rollouts** or **Flagger** automates the progression (5→25→50→100), watches Prometheus metrics, auto-rolls back on regression.

---

## 10. Load balancing

| Algorithm | When |
|---|---|
| `ROUND_ROBIN` | Default. Similar pods. |
| `LEAST_REQUEST` | Default for HTTP. Handles mixed pod capacity. |
| `RANDOM` | Many endpoints where RR ordering is biased. |
| `consistentHash` | Sticky sessions / shard by key. Painful under endpoint churn. |

**Key insight:** Istio LB is **L7, per-request**. Each HTTP call can go to a different pod. Without a mesh, you get **L4, per-connection** kube-proxy LB — once a TCP conn opens, all requests on it ride the same pod. That's why HPA scale-up often doesn't help: new pods get no traffic until existing connections churn. Mesh fixes this.

**HTTP/2 trap:** With `maxRequestsPerConnection: 0` (unlimited, default), Envoy keeps using existing conns → new pods get no traffic after HPA. Set `maxRequestsPerConnection: 1000` to force periodic re-balance.

---

## 11. Locality-aware load balancing (multi-region must-know)

Two failure modes:
1. **Locality LB needs `outlierDetection` enabled** — without it, Envoy doesn't know endpoints are "unhealthy" → failover never triggers.
2. **Locality LB needs accurate locality labels** (`topology.kubernetes.io/region`, `topology.kubernetes.io/zone`). EKS sets them automatically.

```yaml
trafficPolicy:
  outlierDetection: { ... }
  loadBalancer:
    localityLbSetting:
      enabled: true
      failover:
      - from: eu-west-1
        to:   eu-central-1
```

---

## 12. Rate limiting — local vs global

**Local (per-Envoy, no coordination)** — each pod takes max N req/s. Good for protecting individual pods. Bad for global quotas. Wired via `EnvoyFilter`.

**Global (RLS service + Redis)** — Envoys call out to a Rate Limit Service that holds shared counters. Real per-user / per-API-key quotas. Adds latency + new SPOF.

**Where each belongs:**
- Mesh layer (Istio) — local rate limit for pod overload protection.
- API gateway layer (Gravitee/Kong) — global per-consumer quotas tied to Plans/Subscriptions.

Use both, defense in depth.

---

## 13. Egress control

Three patterns:

1. **`ServiceEntry`** — makes external host known to mesh; can now apply VS retries/timeouts and DR outlier detection to it. Same primitives, external and internal.
2. **Egress gateway** — funnel all egress through dedicated pods on dedicated nodes with stable public IP (for partner allowlists), centralized audit, kill-switch capability.
3. **`outboundTrafficPolicy: REGISTRY_ONLY`** — block any destination not in mesh registry. Strong zero-trust egress; forces every external call to have a ServiceEntry.

**Production pitch:** "For a Web3 company calling dozens of blockchain RPC providers, I'd combine REGISTRY_ONLY + egress gateway + explicit ServiceEntries per provider. You get full audit trail, per-provider outlier detection (kick a node provider out for 30s if it 5xxs), dedicated public source IP for provider allowlists, and a kill-switch. The friction is real but worth it for crypto."

---

## 14. Sidecar lifecycle gotchas

1. **App starts before sidecar ready** → outbound traffic fails. Fix: `holdApplicationUntilProxyStarts: true`.
2. **App exits before sidecar drains** → drops inflight. Fix: `EXIT_ON_ZERO_ACTIVE_CONNECTIONS=true` + sane `terminationGracePeriodSeconds`.
3. **Jobs/CronJobs never finish** → sidecar runs forever. Fix: `curl -X POST localhost:15020/quitquitquit` when done, OR use Istio 1.21+ **native sidecars** (K8s 1.29+ init containers with `restartPolicy: Always`).
4. **Init containers have no network** → iptables redirect set up but Envoy isn't running yet during init phase. Fix: native sidecars.
5. **Sidecar OOM at scale** → each Envoy gets the whole service registry by default. At 1000+ services: 200-300MB per sidecar. Fix: `Sidecar` CRD to scope which services this workload knows about.

---

## 15. Observability — built-in and the cardinality problem

**Auto-emitted metrics** (Prometheus, `istio_*`):
- `istio_requests_total{response_code, source_workload, destination_workload, ...}` — RED rate + errors
- `istio_request_duration_milliseconds_bucket` — RED duration
- `istio_request_bytes` / `istio_response_bytes`

**Distributed tracing:** Envoy generates spans automatically and propagates `x-request-id` / `traceparent`. **App MUST forward those headers** when calling downstream — otherwise spans don't link. This is the #1 reason people say "tracing doesn't work."

**Cardinality is your enemy** — default 10+ labels × N services × M versions = millions of series. Use `Telemetry` CR to drop unused dimensions:
```yaml
metrics:
- providers: [{ name: prometheus }]
  overrides:
  - tagOverrides:
      source_version: { operation: REMOVE }
```

**Sampling:** trace sampling 1% default; for high-traffic, 0.1% + tail-based sampling at OTel collector (only keep traces with errors or high latency).

---

## 16. Multi-cluster topologies

| Pattern | Trade-off |
|---|---|
| **Primary-remote** | One istiod, others are remote. Simple. Blast radius = primary cluster. |
| **Multi-primary** | istiod per cluster, shared root CA. No SPOF, more config sync. |
| **External control plane** | istiod outside data clusters in a dedicated mgmt cluster. Clean separation, more plumbing. |

**Three failure modes to mention:**
1. Trust-domain mismatch — must share root CA via cert-manager / Vault.
2. Endpoint discovery lag during control-plane partition → stale endpoints → traffic to dead pods.
3. East-west gateways are chokepoints; capacity-plan them.

---

## 17. Ambient mode (currency signal)

- **No sidecars.** `ztunnel` DaemonSet per node handles L4 mTLS + identity. `waypoint` proxy (per-ns or per-svc) handles L7 policy.
- **Wins:** lower overhead, easier upgrades (one ztunnel per node, not 1000 sidecars), works with non-cooperative workloads.
- **Trade-offs:** newer (beta in 1.22+), L7 features need waypoint hop = extra latency.

**Smart pitch:** "I'd pilot on non-critical namespaces, validate identity + policy parity, benchmark L7 latency before migrating latency-sensitive workloads. The L4-only path is a clear win for stable services that don't need L7 policy."

---

## 18. Revision-based upgrades

Never do in-place. Install new control plane alongside old:

```bash
istioctl install --set revision=1-22-1
```

Migrate namespace by namespace:
```bash
kubectl label ns payments istio.io/rev=1-22-1 --overwrite
kubectl rollout restart deploy -n payments
```

**Revision tags** (cleaner): `istioctl tag set prod --revision=1-22-1`. Namespaces label `istio.io/rev=prod`. Flip the tag to switch mesh-wide. Rollback = flip back.

**Discipline:** control plane first, then data plane (sidecars), never both at once. Don't skip minor versions.

---

## 19. External authz (`ext_authz`) — security-critical pattern

For complex authz beyond "ALLOW if SA == X" — Envoy calls an external service (OPA, custom) before forwarding.

```yaml
kind: AuthorizationPolicy
spec:
  action: CUSTOM
  provider: { name: opa-ext-authz }
  rules:
  - to: [{ operation: { paths: ["/v2/wallet/*/transfer"] } }]
```

**Why mention it:** "If KYC tier ≥ 2 AND amount ≤ daily limit AND destination not sanctioned" isn't expressible as AuthorizationPolicy alone. ext_authz to OPA is the pattern for crypto-grade authz.

---

## 20. EnvoyFilter — the escape hatch

Patches raw Envoy config when CRDs can't express what you need (Wasm extensions, exotic filters, custom listener TLS params).

**Why use sparingly:**
- Version-coupled to Envoy — Istio upgrade can silently break it.
- Hard to test in CI, easy to brick the mesh.

**Discipline:** scope tightly with `workloadSelector`, pin Istio version, have a removal runbook. Prefer `WasmPlugin` CRD for Wasm.

---

## 21. Gateway API (currency signal)

Istio supports `gateway.networking.k8s.io/v1` as an alternative to its `Gateway` + `VirtualService` CRDs.

- Multi-vendor standard (works on Istio, Linkerd, Cilium, Contour).
- Role split: GatewayClass (platform), Gateway (admin), HTTPRoute (app).
- For new clusters → default to Gateway API for north-south.
- Keep Istio CRDs for east-west / advanced policy until GAMMA (mesh) matures.

---

## 22. How kube-proxy, CoreDNS, and Istio coexist

**Packet flow when pod-with-sidecar calls a service:**

1. App calls `wallet-svc.payments.svc.cluster.local`.
2. **CoreDNS** resolves → ClusterIP `10.96.0.42`. (CoreDNS unchanged by Istio.)
3. App sends TCP SYN to `10.96.0.42:80`.
4. **`istio-init`'s iptables rules** redirect the packet to `localhost:15001` (Envoy outbound) — **before** kube-proxy NAT can act.
5. Envoy reads original destination via `SO_ORIGINAL_DST`.
6. Envoy looks up destination in **its own EDS-pushed registry** (from istiod, not kube-proxy).
7. Envoy applies LB, retries, mTLS, and connects **directly** to the destination pod IP.
8. On destination pod, inbound iptables redirects port 8080 → 15006 (Envoy inbound). Envoy terminates mTLS, checks AuthorizationPolicy, forwards plaintext to app on `localhost:8080`.

**Responsibility split:**
| Component | What it still does with Istio installed |
|---|---|
| CoreDNS | All DNS. Unchanged. |
| kube-proxy | NodePort/LoadBalancer NAT. Pods without sidecars. **Bypassed for mesh-to-mesh traffic.** |
| CNI (Cilium/Calico/VPC CNI) | Pod IPs, node routing, NetworkPolicies. **Runs below Istio.** |
| Istio iptables | Redirects outbound→15001, inbound→15006. |
| Envoy | Picks endpoint via own registry, applies all L7 policy, opens direct pod-to-pod conn. |

**Critical detail:** Envoy runs as **UID 1337**. iptables rules skip UID 1337's outbound traffic — otherwise infinite redirect loop.

**NetworkPolicy interaction:** enforced at CNI layer, BELOW Istio. If a NetworkPolicy drops a packet, Istio never sees it. Use both — L3/L4 NetworkPolicy + L7 AuthorizationPolicy = defense in depth.

---

## 23. AWS NLB / ALB integration

**Default for Istio ingress GW: NLB. Not ALB.** Because ALB does its own L7 routing + TLS, which duplicates what Envoy is already doing.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ...
```

**Why each annotation matters:**
- `nlb-target-type: ip` → pods registered directly as targets (requires AWS VPC CNI). Kube-proxy bypassed for ingress.
- `externalTrafficPolicy: Local` → preserves client source IP (vs `Cluster` which SNATs).
- `proxy-protocol: "*"` → PROXY protocol v2; preserves source IP through L4 LB. Must enable Envoy listener filter too.
- `cross-zone: true` → NLB distributes across zones (default is zone-isolated → uneven load).

**TLS termination — three patterns:**
- **A: NLB passthrough → TLS at Envoy** (recommended default — single termination point, cert-manager + K8s secrets).
- **B: NLB terminates (ACM) → plaintext to Envoy** (simpler certs, traffic plaintext in VPC).
- **C: NLB terminates → re-encrypt to Envoy** (most secure, double termination, most painful ops).

**Health checks:** point at port `15021` `/healthz/ready` — don't health-check on 443 (cert misconfig = full outage).

**DNS:** ExternalDNS controller watches Services/Gateways, creates Route 53 ALIAS records automatically.

**Multi-region:** Global Accelerator over Route 53 latency-based — sub-30s failover beats DNS TTL caching.

**WAF:** doesn't attach to NLB directly. Put CloudFront (with WAF + Shield Advanced) in front of NLB for crypto-grade edge protection.

**Production gotchas:**
- PROXY protocol mismatch = every request fails.
- `externalTrafficPolicy: Cluster` + no PROXY protocol = source IP becomes node IP.
- Health check on TLS port + cert problem = full outage.
- AWS LB Controller IAM via IRSA — missing perms = Service stuck pending.
- Enable `deletion_protection.enabled=true` — deleting the Service deletes the NLB + EIPs allowlisted with partners.
- Subnet tagging (`kubernetes.io/role/elb`) required.

---

## 24. Useful `istioctl` commands

```bash
istioctl analyze                          # validate config across cluster
istioctl proxy-status                     # are all sidecars in sync with control plane?
istioctl proxy-config endpoints <pod>     # what endpoints does this Envoy know about
istioctl proxy-config routes <pod>        # what routes loaded
istioctl proxy-config clusters <pod>      # what clusters loaded
istioctl proxy-config listeners <pod>     # listeners
istioctl x describe pod <pod>             # human-readable mesh config for this pod
istioctl x authz check <pod>              # which AuthorizationPolicies apply
istioctl tag set prod --revision=1-22-1   # flip mesh revision via tag
```

Plus on a sidecar directly:
```bash
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/stats | grep <metric>
kubectl exec <pod> -c istio-proxy -- iptables-save -t nat   # see redirect rules
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/config_dump
```

---

# PART 2 — THE 6 HIGH-LIKELIHOOD DEEP-DIVES

## Gap 1: JWT validation + claim-based authz

### App-level mental model
```js
app.use(jwt({ secret: jwks(), algorithms: ['RS256'] }))
app.post('/transfer', (req, res) => {
  if (req.user.tier < 2) return res.status(403).end()
})
```

### Istio way — two CRDs together

**`RequestAuthentication`** validates the JWT (issuer X, JWKS Y). Does NOT allow/deny — just attaches validated claims.

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata: { name: ledger-jwt, namespace: payments }
spec:
  selector:
    matchLabels: { app: wallet-svc }
  jwtRules:
  - issuer: "https://auth.ledger.com"
    jwksUri: "https://auth.ledger.com/.well-known/jwks.json"
    audiences: ["wallet-api"]
    forwardOriginalToken: true
```

**`AuthorizationPolicy`** makes the actual allow/deny decision, including on JWT claims:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: transfer-requires-tier-2 }
spec:
  selector:
    matchLabels: { app: wallet-svc }
  action: ALLOW
  rules:
  - to:
    - operation: { methods: ["POST"], paths: ["/transfer"] }
    when:
    - key: request.auth.claims[tier]
      values: ["2", "3", "premium"]
```

### The trap most people miss

**`RequestAuthentication` alone does NOT require a token.** It only validates IF a token is present. To require one, add an AuthorizationPolicy with:

```yaml
- from:
  - source:
      requestPrincipals: ["*"]    # require any authenticated principal
```

Or DENY missing tokens:
```yaml
action: DENY
rules:
- from:
  - source:
      notRequestPrincipals: ["*"]
```

---

## Gap 2: AuthorizationPolicy semantics — evaluation order

### The four actions

| Action | Behavior |
|---|---|
| **`ALLOW`** | Allow if any rule matches |
| **`DENY`** | Deny if any rule matches (and stop) |
| **`AUDIT`** | Log the match but pass through |
| **`CUSTOM`** | Delegate to `ext_authz` provider (OPA) |

### Evaluation order — memorize this

1. **CUSTOM** policies first. If any CUSTOM provider returns DENY → denied.
2. **DENY** policies. If any matches → denied immediately.
3. **ALLOW** policies. If at least one ALLOW policy exists for this workload:
   - If at least one matches → allowed.
   - If none match → **denied** (default-deny when any ALLOW exists).
4. If **no ALLOW policies exist at all** for this workload → **allowed by default**.
5. **AUDIT** runs in parallel; never affects the outcome.

### The mental model — counterintuitive bit

**The presence of any ALLOW policy on a workload flips the default from "allow all" to "deny by default."**

| Scenario | Result |
|---|---|
| No policies at all | ALLOW |
| Only DENY policies, none match | ALLOW |
| Only ALLOW policies, one matches | ALLOW |
| Only ALLOW policies, none match | **DENY** ← surprise |
| ALLOW matches AND DENY matches | DENY (DENY wins) |
| CUSTOM denies | DENY |

### Zero-trust starter
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: default-deny, namespace: payments }
spec: {}                # empty = matches nothing for ALLOW = deny all
```

---

## Gap 3: TLS origination

### Why centralize TLS at the sidecar

- Centralized cert verification (one CA bundle, not per-app).
- Mesh-level observability of upstream calls.
- Apps speak HTTP locally and stop caring about TLS.

### How — two resources

```yaml
# 1. Make external host known
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata: { name: stripe }
spec:
  hosts: ["api.stripe.com"]
  ports:
  - { number: 443, name: https, protocol: HTTPS }
  - { number: 80,  name: http,  protocol: HTTP }    # app speaks plain HTTP
  resolution: DNS
  location: MESH_EXTERNAL
---
# 2. Upgrade plaintext to TLS at Envoy
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: { name: stripe-tls }
spec:
  host: api.stripe.com
  trafficPolicy:
    portLevelSettings:
    - port: { number: 80 }
      tls:
        mode: SIMPLE                # Envoy initiates TLS to upstream
        sni: api.stripe.com         # mandatory for modern HTTPS
```

### `tls.mode` options

| Mode | What Envoy does |
|---|---|
| `DISABLE` | No TLS upstream |
| `SIMPLE` | Standard TLS to upstream (most external HTTPS APIs) |
| `MUTUAL` | mTLS with custom client cert (clientCert/privateKey/caCert) |
| `ISTIO_MUTUAL` | mTLS with mesh-issued SPIFFE cert (internal services) |

### Gotcha: missing SNI = handshake fails

---

## Gap 4: Control plane outage blast radius

### Survives a control plane outage
- Existing pod-to-pod traffic (cached config)
- mTLS for existing pods (until cert expiry ~24h)
- Existing routing rules
- Metrics + tracing

### Breaks during outage
- New pods — no sidecar injection (webhook unavailable)
- Config changes — can't push VS/DR/etc.
- Cert rotation — after ~24h, certs expire, mTLS breaks
- **Endpoint changes** — sidecars don't learn about new/removed pods → traffic to dead pods, no traffic to new pods
- Webhook validation — can't create new Istio resources

### Mitigations
- Replicated istiod (`replicas: 3`), PodDisruptionBudget `minAvailable: 2`.
- Spread across AZs.
- K8s API server is the deeper SPOF — istiod is stateless.

---

## Gap 5: VirtualService L7 features

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: api }
spec:
  hosts: [api.ledger.com]
  gateways: [public-gw]
  http:

  # Rewrite — change path before forwarding upstream
  - match: [{ uri: { prefix: /api/v1/ } }]
    rewrite: { uri: /v1/ }
    route: [{ destination: { host: wallet-svc } }]

  # Redirect — send 301 to client, don't forward upstream
  - match: [{ uri: { exact: /old-path } }]
    redirect:
      uri: /new-path
      redirectCode: 301

  # Header manipulation
  - match: [{ uri: { prefix: /payments/ } }]
    route:
    - destination: { host: payments-svc }
      headers:
        request:
          set: { x-internal-region: eu-west }
          add: { x-trace-id: "%REQ(X-REQUEST-ID)%" }
          remove: [x-internal-secret]
        response:
          set: { strict-transport-security: "max-age=31536000" }

  # CORS
  - match: [{ uri: { prefix: /api/ } }]
    corsPolicy:
      allowOrigins:
        - { exact: https://app.ledger.com }
        - { regex: "https://.*\\.ledger\\.com" }
      allowMethods: [GET, POST, OPTIONS]
      allowHeaders: [authorization, content-type]
      allowCredentials: true
      maxAge: 24h
    route: [{ destination: { host: api-svc } }]
```

### Match precedence
**First matching rule wins.** Put specific rules above general ones.

### Match conditions
```yaml
match:
- uri: { exact: /path } | { prefix: /api/ } | { regex: ".*\\.json" }
  method: { exact: POST }
  headers:
    x-tier: { exact: premium }
  queryParams:
    debug: { exact: "true" }
  port: 443
  sourceLabels: { app: frontend }
```

---

## Gap 6: Telemetry CR — concrete examples

### Enable access logs only on errors
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata: { name: error-only-logs, namespace: istio-system }
spec:
  accessLogging:
  - providers: [{ name: envoy }]
    filter:
      expression: 'response.code >= 400'
```

### Increase trace sampling for one service
```yaml
spec:
  selector: { matchLabels: { app: payments } }
  tracing:
  - providers: [{ name: otel }]
    randomSamplingPercentage: 10.0      # default is 1%
```

### Drop high-cardinality dimensions
```yaml
spec:
  metrics:
  - providers: [{ name: prometheus }]
    overrides:
    - match:
        metric: REQUEST_COUNT             # affects istio_requests_total
      tagOverrides:
        source_version: { operation: REMOVE }
        destination_version: { operation: REMOVE }
        source_principal: { operation: REMOVE }
```

### Add custom dimension from JWT claim
```yaml
spec:
  metrics:
  - providers: [{ name: prometheus }]
    overrides:
    - match: { metric: REQUEST_COUNT }
      tagOverrides:
        user_tier:
          value: 'request.auth.claims["tier"]'
```

### Disable telemetry for noisy workload
```yaml
spec:
  selector: { matchLabels: { app: noisy-batch-job } }
  metrics:
  - providers: [{ name: prometheus }]
    disabled: true
```

---

# PART 3 — GENERAL TROUBLESHOOTING PLAYBOOK

For any issue you haven't seen before, walk through these steps.

## Step 1: What's the symptom?

| Symptom | First place to look |
|---|---|
| 503 with response flag | Envoy access log (response flag tells you cause) |
| 401/403 | `istioctl x authz check` on destination pod |
| Latency spike | Distributed tracing → `istio_request_duration` → sidecar CPU |
| Cert errors | `istioctl proxy-config secret <pod>`; istiod logs |
| Routing surprise | `istioctl proxy-config routes <pod>`; check VS order |
| Sidecar not injecting | Namespace label, webhook config, istiod logs |

## Step 2: Is the control plane healthy?

```bash
istioctl proxy-status               # are sidecars synced with istiod?
kubectl -n istio-system get pods
kubectl -n istio-system logs deploy/istiod --tail 200
```

`STALE` or `NOT SENT` in proxy-status → config isn't reaching that sidecar.

## Step 3: Inspect the failing pod's sidecar from inside

```bash
istioctl proxy-config listeners <pod>
istioctl proxy-config clusters <pod>
istioctl proxy-config endpoints <pod>
istioctl proxy-config routes <pod>

kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/stats | grep <something>
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/config_dump
```

## Step 4: Trace a single request

```bash
istioctl x describe pod <destination-pod>

kubectl exec <source-pod> -c <app> -- curl -v http://destination/path -H "x-request-id: debug-1"

kubectl logs <source-pod> -c istio-proxy | grep debug-1
kubectl logs <destination-pod> -c istio-proxy | grep debug-1
```

## Step 5: Localize the failing layer

| Layer | If failing, check |
|---|---|
| **DNS** | `nslookup` from inside pod → CoreDNS |
| **CNI / NetworkPolicy** | `nc -zv` from non-injected debug pod |
| **iptables redirect** | `iptables-save -t nat` inside istio-proxy |
| **Envoy outbound listener** | Sidecar logs for `outbound\|<port>\|<subset>\|<host>` |
| **Envoy clusters** | `istioctl proxy-config endpoints` |
| **mTLS handshake** | Sidecar logs for SSL errors; `istioctl proxy-config secret` |
| **AuthorizationPolicy** | `istioctl x authz check` + Envoy logs `RBAC: denied` |
| **App** | Container logs |

## Step 6: The bypass test

```bash
kubectl run debug --image=nicolaka/netshoot -it --rm \
  --overrides='{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"false"}}}}}' \
  -- /bin/bash

# from inside, curl destination directly
```

Works without sidecar but fails with → Istio is the cause.
Fails both ways → K8s/network/app, not mesh.

## Step 7: Senior hands-on moves

```bash
# force sidecar restart for fresh config
kubectl exec <pod> -c istio-proxy -- curl -X POST localhost:15000/quitquitquit

# force resync from istiod
kubectl rollout restart deploy <deployment>

# live Envoy admin UI
kubectl port-forward <pod> 15000:15000
# then http://localhost:15000/ in browser
```

## Step 8: Escalation signals

- istiod logs full of cert errors → CA / cert-manager problem.
- Many `proxy-status` STALE simultaneously → istiod overloaded or split-brained.
- Mass `UH` across many services → upstream dep down + outlier detection amplifying.

---

# PART 4 — THE 26 Q&A (memorize these answers verbatim if possible)

## Fundamentals

### Q1. What's the single process that runs the control plane, and what's the data-plane proxy?

Control plane is **`istiod`** — one process that consolidated Pilot/Citadel/Galley in 1.5. Data plane is **Envoy** — same binary as sidecar in every pod and as ingress gateway pods at the edge. istiod pushes config via **xDS (gRPC)**.

### Q2. How does a pod's outbound packet get to its sidecar before kube-proxy NAT can act?

`istio-init` writes **iptables rules** redirecting outbound on most ports to `localhost:15001` (Envoy outbound) and inbound to `localhost:15006`. Those rules run **before** kube-proxy's NAT chain, intercepting the packet. Envoy reads original destination via `SO_ORIGINAL_DST`, picks a healthy endpoint from its xDS registry, connects directly to a pod IP — bypassing kube-proxy entirely for mesh traffic.

### Q3. Why does Envoy run as UID 1337?

So iptables can **exclude UID 1337's outbound traffic** from the redirect. Without that, Envoy's own upstream connections would be redirected back to Envoy in an infinite loop.

## CRDs

### Q4. VS vs DR responsibilities?

**`VirtualService` is the request side** — matches requests (path/header/method), defines routing, timeouts, retries, mirror, fault injection.
**`DestinationRule` is the upstream side** — defines subsets by label, picks LB algorithm, configures connection pool, outlier detection, upstream mTLS mode.
Shortcut: VS = "where does the request go?", DR = "how do we talk to the place it goes?"

### Q5. What does `ServiceEntry` do?

Adds an external host to the mesh's service registry. Without it, calls to external hosts go through Envoy as passthrough — no policy, no proper metrics. With it, you can apply VS retries/timeouts and DR outlier detection to external dependencies — same primitives, internal and external.

### Q6. What does the `Sidecar` CRD do?

Scopes which services a workload's sidecar knows about. By default every Envoy gets the entire service registry — at 800+ services that's ~200-300MB per sidecar. `Sidecar` CRD lets you limit it to specific namespaces/hosts. Performance lever at scale.

## Resilience

### Q7. Why is `retryOn: 5xx` dangerous? Safe alternative?

Causes **retry storms** — upstream overload → retry → more overload → death spiral. Safe: `retryOn: gateway-error,connect-failure,reset` — request never reached the app, replay is idempotent. Pair with retry budget capping retries at ~20% of original traffic.

### Q8. Which two fields = circuit-breaker? Which CRD?

Both in `DestinationRule.trafficPolicy`:
- **`outlierDetection`** — passively ejects unhealthy endpoints (open circuit per pod).
- **`connectionPool`** — caps max connections/pending requests, fails fast on overload.
Together: protect from sending traffic to dying pods + protect upstream from being buried.

### Q9. HPA added pods but they don't get traffic. Most likely Istio cause?

**HTTP/2 connection reuse.** Default `maxRequestsPerConnection: 0` (unlimited) → Envoy reuses existing connections → new pods get zero traffic until conns churn. Fix: `DestinationRule.connectionPool.http.maxRequestsPerConnection: 1000`.

## Security

### Q10. Three `PeerAuthentication` modes, migration to STRICT?

**STRICT** (only mTLS), **PERMISSIVE** (both), **DISABLE** (plaintext only).
Migration: PERMISSIVE mesh-wide → inject sidecars everywhere → verify via metrics that all traffic is mTLS → flip to STRICT namespace-by-namespace. Keep PERMISSIVE where non-mesh clients exist.

### Q11. AuthorizationPolicy evaluation order? No-policy vs only-ALLOW behavior?

**CUSTOM → DENY → ALLOW**, AUDIT in parallel.
- No policies: ALLOW (default).
- Only ALLOW policies, none match: **DENY** ← counterintuitive.
- ALLOW match + DENY match: DENY wins.
The presence of any ALLOW policy flips the default to deny-by-default for that workload.

### Q12. Two CRDs to require valid JWT on every request?

**`RequestAuthentication`** validates the JWT (issuer, JWKS, audiences) but does NOT require one alone.
**`AuthorizationPolicy`** with `from.source.requestPrincipals: ["*"]` requires presence. Can also match on `request.auth.claims[...]` for claim-based authz.
Trap: only deploying `RequestAuthentication` lets unauthenticated requests through.

### Q13. When to use `action: CUSTOM`?

When authz needs business logic not expressible as static rules — KYC tier, daily limits, sanction lists. `CUSTOM` delegates to an external authz service (typically OPA via ext_authz). For crypto-grade compliance authz.

## Traffic management

### Q14. YAML for 10% canary v2, 90% v1 of `wallet-svc`?

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: wallet-svc }
spec:
  hosts: [wallet-svc]
  http:
  - route:
    - { destination: { host: wallet-svc, subset: stable }, weight: 90 }
    - { destination: { host: wallet-svc, subset: canary }, weight: 10 }
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: { name: wallet-svc }
spec:
  host: wallet-svc
  subsets:
  - { name: stable, labels: { version: v1 } }
  - { name: canary, labels: { version: v2 } }
```

### Q15. Istio L7 LB vs kube-proxy L4 LB — why HPA care?

**Kube-proxy is L4, per-connection** — once TCP conn lands on a pod, every request on it rides the same pod. With HTTP/2 keep-alive, HPA scale-up adds pods that get zero traffic until conns churn.
**Istio is L7, per-request** — every HTTP request can go to a different pod, new pods immediately get traffic. Plus algorithm choice, outlier detection, locality awareness.

### Q16. App calls `http://api.stripe.com`, Envoy does HTTPS. How?

`ServiceEntry` declaring `api.stripe.com` on port 80 protocol HTTP. `DestinationRule` with `tls.mode: SIMPLE` and explicit `sni: api.stripe.com`. Envoy originates TLS to Stripe. SNI mandatory — modern HTTPS rejects without it.

### Q17. VS snippet for `/old-path` → `/new-path` 301?

```yaml
http:
- match: [{ uri: { exact: /old-path } }]
  redirect:
    uri: /new-path
    redirectCode: 301
```
No `route` field — Envoy generates 301 itself without forwarding upstream.

## Observability

### Q18. Default cardinality problem and fix?

Default ~10 labels × 100 services × 10 versions × 50 paths = millions of series → Prometheus dies. Fix: `Telemetry` CR with `tagOverrides` REMOVE for `source_version`, `destination_version`, `source_principal`. Audit cardinality monthly.

### Q19. Access logs only on failures?

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata: { name: error-only, namespace: istio-system }
spec:
  accessLogging:
  - providers: [{ name: envoy }]
    filter:
      expression: 'response.code >= 400'
```
~99% log volume reduction.

## Operations

### Q20. What survives an istiod outage, what breaks?

**Survives:** existing traffic (cached config), mTLS until cert expiry (~24h), metrics, tracing, existing routing.
**Breaks:** new pod sidecar injection, config push, **endpoint discovery** (traffic to dead pods, starvation of new ones), cert rotation after 24h.
Mitigation: 3 replicas + PDB + AZ spread. K8s API server is the deeper SPOF.

### Q21. Revision-tag upgrade from 1.21 to 1.22?

```bash
istioctl install --set revision=1-22-0
istioctl tag set prod --revision=1-22-0
kubectl rollout restart deploy -n payments
```
Namespaces label `istio.io/rev=prod`. Rollback = flip tag back. Discipline: control plane first, then data plane; never skip minor versions.

### Q22. Three sidecar lifecycle gotchas?

1. App starts before sidecar ready → outbound fails. Fix: `holdApplicationUntilProxyStarts: true`.
2. Jobs/CronJobs never finish → sidecar runs forever. Fix: `curl localhost:15020/quitquitquit` on completion, or K8s 1.29+ native sidecars.
3. App exits before sidecar drains → connection resets. Fix: `EXIT_ON_ZERO_ACTIVE_CONNECTIONS=true` + `terminationGracePeriodSeconds: 60+`.

## Networking integration

### Q23. Delete kube-proxy, what breaks?

**NodePort and LoadBalancer services** — including path INTO cluster via Istio ingress GW. Pods without sidecars lose ClusterIP NAT. Pod-to-pod mesh traffic still works (kube-proxy was bypassed). Cilium with kube-proxy-replacement works fine.

### Q24. NetworkPolicy vs AuthorizationPolicy — difference + order?

**NetworkPolicy** = L3/L4, enforced at CNI (Cilium/Calico/VPC CNI), packets dropped at IP+port.
**AuthorizationPolicy** = L7, enforced by Envoy, operates on identity/path/method/JWT claims.
NetworkPolicy runs **first** (CNI is below Istio iptables). Defense in depth = use both.

## AWS integration

### Q25. NLB vs ALB in front of Istio ingress GW + where for CloudFront?

ALB duplicates Envoy's L7 work — same logic in two places. NLB is L4 only, lets Envoy be single source of L7 truth. Plus native gRPC/WebSocket, static EIPs per AZ for partner allowlists.
WAF doesn't attach to NLB → put **CloudFront + WAF + Shield Advanced** in front of NLB for crypto-grade edge protection. Stack: `Client → CloudFront → NLB → Istio ingress GW → mesh`.

## Troubleshooting

### Q26. Widespread `UH` 503s across services — debug walkthrough?

1. **Scope check** — really mesh-wide or one shared upstream? `istio_requests_total{response_flags="UH"}` grouped by destination.
2. **Control plane health** — `istioctl proxy-status` for STALE sidecars; istiod logs.
3. **Endpoints check** — `istioctl proxy-config endpoints <caller> | grep <dest>`. Zero endpoints = Service selector broken. Endpoints exist but unhealthy = outlier detection ejected.
4. **Outlier cascade** — `curl localhost:15000/stats | grep outlier_detection.ejections_active`. High = shared dep broken, upstream 5xxs, ejection cascade. Fix `minHealthPercent: 50` + find root.
5. **App logs** on a representative upstream pod.
6. **Bypass test** — non-injected pod in same ns; works without sidecar but fails with → Istio is the cause.

Senior framing: "`proxy-status` for control-plane sync, then `proxy-config endpoints` for data-plane reality, then outlier stats for cascading failures, then app logs. Bypass test localizes whether Istio is even the cause."

---

# PART 5 — REFERENCE: MNEMONICS, WAR STORIES, WHAT NOT TO SAY

## Mnemonics

- **Four CRDs:** **G**ateway-**V**S-**DR**-**SE**
- **mTLS modes:** **S-P-D** → Strict, Permissive, Disable (migrate via P)
- **503 flags to know first:** **UH-UF-UC-NR** → "Up-Healthy, Up-Failure, Up-Closed, No-Route"
- **Circuit breaker = OD + CP** → **O**utlier**D**etection + **C**onnection**P**ool (both in DR)
- **Retry safety:** **gcr** → **g**ateway-error, **c**onnect-failure, **r**eset
- **AuthZ order:** **CDA(A)** → CUSTOM, DENY, ALLOW (AUDIT in parallel)

## Three production war stories (have one ready)

### Story A — Retry storm
> Service A retried Service B on every 5xx, 3 attempts. B had 30s deploy blip. Retries amplified 4×, outlier detection ejected all endpoints, A saw UH and retried more. 15 min recovery. Fix: `retryOn: gateway-error,connect-failure,reset` only, retry budget capped at 20%, `minHealthPercent: 50`. Drove platform-wide retry budgets.

### Story B — STRICT mTLS broke monitoring
> Flipped namespace to STRICT. CronJob (no sidecar) scraping `/metrics` started failing silently — error budget gone before alerts fired. STRICT rejects plaintext even for metrics. Fix: per-port `PeerAuthentication` exemption on 15020, pre-flip check listing all incoming traffic seen by sidecars.

### Story C — Sidecar OOMs at scale
> Crossed ~800 services. Sidecars OOMing — heap 300MB on workloads talking to maybe 5 services. Default: every sidecar gets full registry. Fix: `Sidecar` CRD scoping per namespace. Avg memory 250MB → 60MB.

## Honest "have you used it in production?" answer

> "I've operated Istio's ingress gateway in production for north-south traffic — TLS termination, routing, basic policy. The resilience patterns — retries, timeouts, circuit-breaking, rate limiting — we implemented at the **application layer** with libraries like resilience4j, axios-retry, OpenTelemetry SDKs. I know the patterns deeply from the inside, but I haven't run them as mesh policy via VirtualService and DestinationRule in a production east-west mesh. Mapping is direct — VS for routing/retries/timeouts, DR for LB/circuit-breaking/conn-pool, PeerAuthentication for mTLS, Telemetry CR for metrics tuning. Productive on mesh config quickly. Would lean on the team for multi-cluster topology and large-scale Envoy debugging until I had real reps."

## What NOT to say

- ❌ "Istio is just an ingress controller" — it's a service mesh; ingress GW is one component.
- ❌ "We enable mTLS to encrypt traffic" — encryption is a side effect; the real value is **workload identity**.
- ❌ "We retry on all 5xx" — retry storms.
- ❌ "Sidecars have no overhead" — they do (~1-3ms p50, 5-10ms p99 per hop; ~50-150MB memory).
- ❌ "Istio is for API management" — it's not; that's Gravitee/Kong/Apigee.
- ❌ Confusing `Sidecar` CRD (scoping) with sidecar **container** (the Envoy).
- ❌ Confusing VS responsibilities (request side) with DR (upstream side).

---

# FINAL READINESS CHECK

If you can:
- Recall all Part 4 Q&A answers in your own words.
- Explain the request flow (CoreDNS → iptables → Envoy → mTLS → app) without looking.
- Write the canary YAML (Q14) from memory.
- Walk through the `UH` debug flow (Q26) systematically.
- Give the honest production-experience answer above.

**You're at 95%+.** That's interview-ready.

For the last 5% — when the interviewer finds something you don't know, fall back to the playbook: *"I'd start by running `istioctl proxy-status` to confirm sync, then `istioctl proxy-config` to inspect the sidecar, then trace a labeled request through to localize the layer."* That demonstrates a systematic approach, which is the senior signal.
