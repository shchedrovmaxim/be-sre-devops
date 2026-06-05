# Ingress vs Gateway API — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why is Gateway API replacing Ingress, and what does the new role-separation buy you?"** — covering Ingress's structural limitations, the GatewayClass/Gateway/Route model, RBAC role separation, cross-namespace routing via ReferenceGrant, and the major implementations.

> Start with the [simple version](./ingress-vs-gateway-api-simple.md) if you haven't read it. The building management analogy is the spine.

---

## The senior framing — Ingress is a lowest-common-denominator API that never grew up

Ingress was introduced in K8s 1.1 (2014). The design premise was: "expose an HTTP service to the internet with host-based and path-based routing." That's all it was ever meant to do.

The problem: real HTTP routing needs more. TLS passthrough (not just termination), request retries, circuit breakers, weighted traffic splitting for canary deploys, header-based routing, session persistence — none of these were in the Ingress spec. So each controller (NGINX, Traefik, AWS ALB, Contour) invented its own annotations to fill the gaps.

The result is a portable API that isn't portable. The spec is portable; the feature set is annotation hell.

Gateway API started from scratch with the lessons of 8 years of Ingress production use. The key insights: role separation and expressiveness in the core spec, not in annotations.

---

## Ingress — what it actually looks like and why it fails

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    # NGINX-specific — won't work on ALB or Traefik
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # ALB-specific — won't work on NGINX
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: nginx       # which controller handles this
  tls:
  - hosts: [api.example.com]
    secretRef:
      name: my-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

**What you can express natively**: host routing, path routing, TLS termination. That's it.

**What requires annotations (controller-specific)**:
- Timeouts and connection settings
- Retries and retry conditions
- Rate limiting
- Custom response headers
- TLS passthrough (instead of termination)
- Canary / weighted routing (NGINX: `nginx.ingress.kubernetes.io/canary-weight: "20"`)
- Authentication (OAuth2 proxy, basic auth)

**What Ingress can't express at all**:
- TCP/UDP routing (only HTTP/HTTPS)
- Cross-namespace backend references (backend must be in the same namespace as the Ingress)
- Role separation (any team with access to the namespace can write any Ingress rule)

---

## Gateway API — the architecture

Three resources, each with a distinct owner:

```
GatewayClass  →  Gateway  →  HTTPRoute / TCPRoute / UDPRoute / GRPCRoute
    │               │                │
platform team   platform team    app team
(write once)    (per-cluster)   (per-service, per-namespace)
```

### GatewayClass

The implementation declaration. Cluster-scoped. Written once by the platform team.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
  description: "Cilium Gateway API implementation"
```

### Gateway

The front door. Defines listeners (ports, protocols, TLS). Namespace-scoped (usually `infra` or `gateway-system`).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod
  namespace: infra
spec:
  gatewayClassName: cilium
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: prod-wildcard-cert
        namespace: infra
    allowedRoutes:
      namespaces:
        from: Selector      # only namespaces matching the label
        selector:
          matchLabels:
            gateway-access: "true"
```

The `allowedRoutes` field is the policy knob. Platform team controls which namespaces are allowed to attach routes.

### HTTPRoute

The routing rules. Namespace-scoped; owned by the app team.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: payments
  namespace: payments
spec:
  parentRefs:
  - name: prod
    namespace: infra       # references the Gateway above
    sectionName: https
  hostnames:
  - payments.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /checkout
      headers:
      - name: X-Canary
        value: "true"      # header-based canary routing
    backendRefs:
    - name: payment-api-canary
      port: 80
      weight: 20           # weighted traffic splitting
    - name: payment-api-stable
      port: 80
      weight: 80
  - matches:
    - path:
        type: PathPrefix
        value: /checkout
    backendRefs:
    - name: payment-api-stable
      port: 80
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Forwarded-Proto
          value: https
    timeouts:              # first-class timeout field, no annotation needed
      request: 30s
      backendRequest: 25s
```

Notice: weighted routing, header matching, timeout configuration — all in the core spec. No annotations.

---

## Cross-namespace references and ReferenceGrant

Gateway API supports routing to a backend **in a different namespace**. This requires explicit permission:

```yaml
# In the TARGET namespace (e.g., the namespace where the backend Service lives)
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-payments-gateway
  namespace: payments-db      # the namespace being referenced INTO
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: payments        # who is allowed to reference resources in this namespace
  to:
  - group: ""
    kind: Service              # what resources they're allowed to reference
```

Without this grant, an HTTPRoute in `payments` cannot reference a Service in `payments-db`. The grant is explicit and owned by the target namespace's team. This is the cross-namespace security model.

This was impossible with Ingress — you could not route to a backend in a different namespace at all.

---

## The major implementations

As of 2025, all major ingress/proxy projects support Gateway API:

| Implementation | Notes |
|---|---|
| **Envoy Gateway** | The CNCF-hosted reference implementation. Built on Envoy. Solid first choice for new clusters. |
| **Cilium Gateway API** | Native eBPF implementation. No Envoy needed. Best choice if you're already using Cilium as your CNI. |
| **Istio Gateway API** | Replaces Istio's own `VirtualService` / `DestinationRule` for L7 routing. `Gateway` maps to Istio's ingress gateway. |
| **NGINX Gateway Fabric** | F5/NGINX's official Gateway API implementation. Replaces the NGINX Ingress Controller. |
| **Contour** | Projectcontour's Envoy-based implementation. Mature, used in production at scale. |

**Conformance testing**: the Gateway API project runs a conformance test suite. Implementations must pass it to claim "Gateway API support." This is the portability story — unlike Ingress, you can actually rely on standard features working the same way.

---

## Migration strategy: running Ingress and Gateway API side by side

There's no automatic migration tool. The recommended path:

1. **Install Gateway API CRDs** (they're separate from core K8s): `kubectl apply -f gateway-api-crds.yaml`
2. **Deploy the new controller** alongside the existing ingress controller.
3. **Create GatewayClass and Gateway resources** for the new controller.
4. **Migrate services one at a time**: create HTTPRoute for the service, test it, then delete the Ingress rule.
5. **Use `ingressClassName`** on old Ingress objects to ensure they stay on the old controller during migration.
6. **When all services are migrated**: remove the old ingress controller and remaining Ingress objects.

The two controllers run on different ports / LB IPs by default, so you can test the new routes without affecting production traffic.

---

## The interview answer in 60 seconds

> "Ingress only natively supports host and path routing. Everything else — timeouts, retries, rate limits, weighted canary splits — is controller-specific annotations. So the spec looks portable but the feature set isn't. An NGINX annotation does nothing on an ALB controller. There's also no role model: any team that can write to a namespace can write an Ingress that conflicts with another team's rules.
>
> Gateway API fixes both. It introduces three resources: `GatewayClass` (which controller to use), `Gateway` (the front door — ports, TLS, protocols, owned by the platform team), and `HTTPRoute` (routing rules, owned by the app team in their namespace). The role separation is enforced by Kubernetes RBAC. App teams can attach routes to the Gateway without being able to modify the Gateway's TLS config. Platform teams can restrict which namespaces are allowed to attach routes at all.
>
> The other gains: cross-namespace references via `ReferenceGrant`, first-class support for weighted routing and header matching in the spec, and conformance testing so implementations actually behave the same. Ingress is feature-frozen; Gateway API is where new work happens."

---

## Self-test drills

### 1. Why is Gateway API replacing Ingress, and what does the new role-separation buy you?

**Reference answer**: Ingress is annotation-driven, not portable, and has no role model. Gateway API separates ownership (platform team owns Gateway, app team owns HTTPRoute), enforces it via RBAC, and puts features like timeouts and header routing into the core spec. Full answer above.

### 2. I want my HTTPRoute in the `payments` namespace to route to a backend Service in the `payments-db` namespace. What do I need?

**Reference answer**: A `ReferenceGrant` in the `payments-db` namespace. It explicitly allows HTTPRoutes in the `payments` namespace to reference Services in `payments-db`. Without it, the cross-namespace reference is rejected. This is the intentional security model — the target namespace has to opt in.

### 3. What's the difference between `GatewayClass` and `Gateway`?

**Reference answer**: `GatewayClass` is the implementation declaration — it points to a specific controller (Cilium, Envoy, NGINX). It's cluster-scoped and usually written once. `Gateway` is an instance of that class — the actual front door for a specific environment (prod, staging). A single `GatewayClass` can have many `Gateway` instances. Multiple `HTTPRoute`s attach to a `Gateway`.

### 4. My team is running NGINX Ingress Controller today. How do I migrate to Gateway API without a big-bang cutover?

**Reference answer**: Install Gateway API CRDs and deploy the new controller (e.g., NGINX Gateway Fabric or Envoy Gateway) alongside NGINX Ingress Controller. Create GatewayClass and a Gateway resource pointing to the new controller. Migrate services one at a time: create an HTTPRoute for each service, verify it works, then remove the corresponding Ingress rule. Use `ingressClassName` to keep old Ingress rules on the old controller during migration. No single point of breakage.

---

## Further reading

- [Gateway API official docs](https://gateway-api.sigs.k8s.io/) — the spec, API reference, and conformance info
- [Envoy Gateway getting started](https://gateway.envoyproxy.io/docs/)
- [Cilium Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/)
- [KEP-1364: Gateway API](https://github.com/kubernetes/enhancements/issues/1364)
- See also: `services.md` (the layer below — Gateway API routes to Services), `endpoints-endpointslices.md`

---

## The 4 dimensions (senior framing)

- **Tech**: GatewayClass (implementation pointer) → Gateway (front door, platform-owned) → HTTPRoute (routing, app-owned); `ReferenceGrant` for cross-namespace backends; native timeout, retry, weight fields; conformance testing for portability. TCP/UDP routes for non-HTTP protocols.
- **People**: Ingress has no role model — all teams fight over one resource. Gateway API formalizes ownership. The `allowedRoutes` field on Gateway is the platform team's policy knob. Give app teams autonomy over their routes; keep TLS and admission control with the platform team. Document the new ownership model before migrating — confusion about "who owns what" is the biggest migration friction.
- **CI/CD**: validate HTTPRoutes in CI before merge (the Gateway API project provides JSON schemas and `kubectl --dry-run=server` works against real cluster). Add a policy that every HTTPRoute must have a matching `ReferenceGrant` if it crosses namespaces (OPA/Kyverno). Track attached routes per Gateway in monitoring — sudden route count spikes can indicate misconfiguration.
- **Operations**: Gateway API controllers expose standard Prometheus metrics. Monitor unhealthy route status (`.status.conditions[type=Accepted, status=False]`). Set up alerts on `HTTPRoute` configuration errors — the controller will mark the route as not accepted if there's a problem, but teams won't notice unless you alert on it. Runbook for "my HTTPRoute isn't working": check route status conditions first, then check ReferenceGrant, then check Gateway allowedRoutes policy.
