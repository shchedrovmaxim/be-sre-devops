# Ingress vs Gateway API — the simple version (the building management analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./ingress-vs-gateway-api.md) becomes easy.

This doc only explains **one idea**:

> **Ingress let app teams configure the front door, but gave them no role separation. Gateway API splits responsibility: the platform team owns the door, the app team owns the routes behind it.**

---

## The building management analogy

Imagine your company's office building. There's a front door (the load balancer / ingress point). People walk in and get directed to the right floor or department.

### The old way (Ingress): one receptionist does everything

The single receptionist does it all: unlocks the front door, manages the security system, decides who goes where, configures the intercom. If you want to add a new rule ("send visitors for the Sales department to floor 3"), you write a sticky note and put it on the receptionist's desk.

**Problem**: every team pastes their own sticky notes on the same desk. The receptionist is overwhelmed. There's no standard sticky note format — each department uses whatever notation they invented. Some sticky notes conflict.

This is Ingress. One API object. Every team annotates it in their own way. No standard for TLS passthrough, timeouts, retries — every controller (NGINX, Traefik, AWS ALB) invents its own annotations. An NGINX annotation doesn't work on an ALB controller.

### The new way (Gateway API): building management is a proper team

- **Facilities team** (platform team) owns the building's **security and infrastructure** — they manage the front doors, the security desk, the intercom backbone. They decide which entrances exist and what rules govern them.
- **Department heads** (app teams) own their own **routing rules** — "visitors for the Sales department go to floor 3, conference room B." They file a standard form with facilities.
- **Facilities can delegate**: "The HR department is allowed to manage routes for floors 2-4, but not touch the front door."

This is Gateway API. Platform team owns `GatewayClass` + `Gateway`. App teams own `HTTPRoute`. The roles are formally separated.

---

## The quick comparison

| | Ingress | Gateway API |
|---|---|---|
| API version | `networking.k8s.io/v1` | `gateway.networking.k8s.io/v1` |
| Platform team owns | Nothing formally | `GatewayClass` + `Gateway` |
| App team owns | Ingress resource (shared) | `HTTPRoute`, `TCPRoute`, etc. |
| Cross-namespace routing | Not supported | Yes, via `ReferenceGrant` |
| TLS, retries, timeouts | Implementation-specific annotations | First-class fields in the spec |
| Portability across controllers | Poor (annotations differ) | Good (standardized spec) |
| Status of Ingress | Still works, no plans to remove | Ingress is feature-frozen |

---

## What Gateway API looks like

Three resources, each owned by a different team:

```yaml
# 1. GatewayClass — platform team writes once
# "This type of Gateway is powered by Envoy Gateway"
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller

---
# 2. Gateway — platform team owns, defines the front door
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: envoy-gateway
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: prod-tls-cert

---
# 3. HTTPRoute — app team owns, in their own namespace
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: payment-api-routes
  namespace: payments  # different namespace from the Gateway
spec:
  parentRefs:
  - name: prod-gateway
    namespace: infra   # references the Gateway the platform team owns
  hostnames:
  - payments.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: payment-api
      port: 80
```

The app team (payments namespace) defines where their traffic goes. They can't touch the Gateway's TLS config or the GatewayClass. The platform team can't accidentally break app routing by reconfiguring the gateway.

---

## Why this matters in interviews

The killer senior insight is **role separation buys autonomy and safety simultaneously**:

- App teams ship routing changes without waiting for a central ops team to edit a shared Ingress.
- Platform team enforces security (TLS, auth policies) at the Gateway level, and app teams can't override it.
- Multiple teams can attach `HTTPRoute`s to the same Gateway without conflicts.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's wrong with Ingress? | It only supports host/path routing. Everything else (TLS passthrough, retries, timeouts) is a controller-specific annotation. Not portable. No role separation. |
| What's a GatewayClass? | The "type" of gateway — points to which controller implements it (Envoy, Cilium, NGINX). Written once by the platform team. |
| What's a Gateway? | The actual front door — defines ports, TLS certs, protocols. Owned by the platform team. |
| What's an HTTPRoute? | The routing rules — which path goes where. Owned by the app team, in the app's own namespace. |
| Can Ingress and Gateway API coexist? | Yes. Many clusters run both during migration. They don't interfere. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Why is Gateway API replacing Ingress, and what does the new role-separation buy you?"**

**Reference answer (intuitive version):**

"Ingress only handles host and path routing. Any other feature — TLS passthrough, custom timeouts, retries, auth — requires controller-specific annotations. An NGINX annotation doesn't work on an ALB controller, so the YAML isn't portable. There's also no role model: any team that can write to the Ingress resource can touch every other team's routing rules.

Gateway API separates this into three resources. The platform team owns `GatewayClass` (which controller to use) and `Gateway` (the front door — ports, TLS, protocols). The app team owns `HTTPRoute` in their own namespace. They can add or change their routes without going through the platform team, and without being able to touch the platform team's TLS config. The split is enforced by Kubernetes RBAC — you literally can't edit what you don't own. That's the buy: teams move independently and safely."

---

## Further reading

- [Gateway API docs](https://gateway-api.sigs.k8s.io/)
- [Deep-dive: ingress-vs-gateway-api.md](./ingress-vs-gateway-api.md) — covers ReferenceGrant for cross-namespace routing, the major implementations (Istio, Cilium, Envoy Gateway, NGINX Gateway Fabric), migration strategy, and multi-cluster patterns
