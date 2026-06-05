# NetworkPolicy — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Design a NetworkPolicy strategy for a cluster running 3 isolated tenants."** — covering default-deny pattern, namespace isolation, kube-dns egress, L7 extensions, debugging flow, and the limitations that require CiliumNetworkPolicy.

> Start with the [simple version](./network-policy-simple.md) if you haven't. The badge-reader analogy is the spine.

---

## The senior framing — NetworkPolicy is a security posture, not a feature flag

Mid-level engineers add a NetworkPolicy when "security asks us to." Senior engineers design a NetworkPolicy strategy from day one because:

- **Default-open is a blast-radius problem**: one compromised pod can reach every other pod in the cluster — databases, internal APIs, secrets endpoints.
- **Namespace isolation is not free**: just because pods are in different namespaces doesn't mean they can't talk. Kubernetes namespaces are NOT a security boundary without NetworkPolicy.
- **Late addition is painful**: retrofitting NetworkPolicy onto running services requires understanding every traffic pattern. Starting with default-deny and opening incrementally is dramatically easier.

The discipline: **default-deny everywhere, then open precisely what you need.**

---

## Mental model: how NetworkPolicy actually works

NetworkPolicy is a *declarative specification*. The CNI plugin (Calico, Cilium) translates it into kernel enforcement (iptables rules, eBPF programs). Without a supporting CNI:

```
kubectl apply -f policy.yaml   → API server accepts it
                               → objects stored in etcd
                               → CNI plugin NOT watching → nothing enforced
                               → all traffic still open
```

This is the silent failure: your team thinks they're secure, they're not. **Always verify CNI support** before relying on NetworkPolicy.

### The selection model

A NetworkPolicy selects pods and defines rules:

```yaml
spec:
  podSelector:           # which pods this policy applies to
    matchLabels:
      app: payment-api
  policyTypes:
  - Ingress              # declaring Ingress here locks ingress even if no ingress: rules
  - Egress               # declaring Egress here locks egress even if no egress: rules
  ingress: [...]         # allowed ingress traffic
  egress: [...]          # allowed egress traffic
```

Key rules:
1. `podSelector: {}` selects ALL pods in the namespace (the default-deny pattern)
2. `policyTypes: [Ingress]` without any `ingress:` block → all ingress denied
3. `policyTypes: [Egress]` without any `egress:` block → all egress denied
4. Multiple policies for the same pod → union (OR). Each policy adds permissions; no policy removes permissions already granted by another.

---

## The default-deny pattern — do this first

Apply this to every namespace that needs isolation:

```yaml
# Step 1: lock the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Now every pod in `payments` drops all traffic — both incoming and outgoing. Then add allowances:

```yaml
# Step 2: allow DNS resolution (MUST have this or nothing works)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```yaml
# Step 3: allow specific ingress (e.g., from the checkout service)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-checkout-to-payment-api
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: checkout
      podSelector:
        matchLabels:
          app: checkout
    ports:
    - port: 8080
      protocol: TCP
```

Note: when you combine `namespaceSelector` AND `podSelector` in the same `from` element, it's an AND (only pods matching both selectors). If you put them in separate `from` elements, it's an OR.

```yaml
# AND — only checkout pods in checkout namespace:
- from:
  - namespaceSelector:
      matchLabels:
        name: checkout
    podSelector:
      matchLabels:
        app: checkout

# OR — any pod in checkout namespace OR any checkout pod anywhere:
- from:
  - namespaceSelector:
      matchLabels:
        name: checkout
  - podSelector:
      matchLabels:
        app: checkout
```

This AND vs OR distinction is the #2 most common bug after the kube-dns miss.

---

## Multi-tenant strategy — 3 isolated tenants

Given three tenants (say, `team-alpha`, `team-beta`, `team-gamma`), each in their own namespace:

### Step 1: namespace labels (prerequisite)

```bash
kubectl label namespace team-alpha tenant=alpha
kubectl label namespace team-beta  tenant=beta
kubectl label namespace team-gamma tenant=gamma
kubectl label namespace kube-system kubernetes.io/metadata.name=kube-system  # usually already set
```

### Step 2: apply default-deny to all tenant namespaces

Apply the `default-deny-all` policy to each namespace.

### Step 3: allow intra-tenant traffic

```yaml
# Allow any pod in team-alpha to talk to any other pod in team-alpha
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-tenant
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: alpha
```

### Step 4: allow egress to DNS and shared platform services

```yaml
# DNS + shared monitoring namespace
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
- to:
  - namespaceSelector:
      matchLabels:
        role: platform   # monitoring, logging namespace
```

### Step 5: allow ingress controller access for public pods

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: ingress-nginx
    podSelector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
  ports:
  - port: 8080
```

### Result

- `team-alpha` pods cannot reach `team-beta` or `team-gamma` pods in either direction.
- All three tenants can resolve DNS.
- All three tenants can reach shared platform services (monitoring, logging).
- Each tenant's public pods accept traffic from the ingress controller.

---

## The "I added a policy and now nothing works" debugging flow

This is the most common NetworkPolicy incident. Systematic approach:

### 1. Identify which pod is broken and what traffic is failing

```bash
kubectl exec -it <broken-pod> -n <namespace> -- nc -zv <target-ip> <port>
# or
kubectl exec -it <broken-pod> -n <namespace> -- curl -v http://<target-svc>:<port>
```

### 2. Check which NetworkPolicies are selecting the broken pod

```bash
kubectl get networkpolicies -n <namespace> -o yaml
# Look for podSelector matching the broken pod's labels
```

### 3. Check Cilium or Calico policy verdict

With Cilium:
```bash
# Live flow inspection — see exactly what's allowed and dropped
hubble observe --namespace <namespace> --pod <pod-name>

# Check policy enforcement
cilium endpoint list
cilium policy get
```

With Calico:
```bash
# Calico policy trace
calicoctl policy trace --src-pod <source-pod> --dst-pod <dest-pod>
```

### 4. Check the kube-dns policy first

```bash
kubectl exec -it <broken-pod> -- nslookup kubernetes.default.svc.cluster.local
# If this fails → DNS egress policy is missing
```

### 5. Check the AND vs OR mistake in from/to selectors

Look at your `from:` and `to:` blocks. If you meant "pods with label X in namespace Y" and wrote them as two separate `- ` items instead of one combined item, you have an OR instead of an AND.

### 6. Verify CNI is actually enforcing

```bash
# Cilium
kubectl get pods -n kube-system -l k8s-app=cilium
# If not running → no enforcement regardless of policies

# Calico
kubectl get pods -n calico-system
```

---

## NetworkPolicy limitations — what it cannot do

| Limitation | Standard NetworkPolicy | Workaround |
|---|---|---|
| L7 rules (HTTP path, method) | No | CiliumNetworkPolicy `toPorts.rules.http` |
| FQDN egress (api.stripe.com) | No | Cilium `toFQDNs`; or egress proxy (Squid, Envoy) |
| Host network pods | No — bypasses NetworkPolicy | OPA/Gatekeeper, Kyverno, or node-level firewall |
| Cross-namespace from same IP | No IPBlock by namespace | CiliumNetworkPolicy `fromCIDR` with namespace selectors |
| Rate limiting | No | Cilium L7 with rate-limit rules; Istio |
| mTLS enforcement | No | Istio or Cilium transparent encryption (WireGuard/IPSec) |

---

## CiliumNetworkPolicy — the L7 patterns

### HTTP path/method restriction

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: restrict-payment-api-methods
  namespace: payments
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: checkout
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: /payment/status/.*
        - method: POST
          path: /payment/charge
```

Only `GET /payment/status/*` and `POST /payment/charge` are allowed from the checkout service. Any other method or path is dropped at the eBPF level.

### FQDN egress (external APIs)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-stripe-egress
  namespace: payments
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  egress:
  - toFQDNs:
    - matchName: "api.stripe.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

The `cilium-agent` resolves `api.stripe.com` periodically and updates the eBPF policy map with the current IPs. No proxy needed.

---

## The interview answer in 60 seconds

> "For three isolated tenants I'd start with namespace isolation. Each tenant gets a namespace with labels. Then I apply a `default-deny-all` NetworkPolicy to every tenant namespace — pod selector empty, both ingress and egress declared, no rules. Everything is locked.
>
> Then I open what's needed: egress to DNS on port 53 to kube-system (this gets missed constantly — if you forget it, nothing resolves), egress to shared platform namespaces for monitoring and logging, intra-tenant ingress from the same namespace label, and ingress from the ingress controller for public-facing pods.
>
> The debugging flow when something breaks: check DNS first, check which policies select the broken pod, use Hubble or `calicoctl policy trace` to see policy verdicts, and double-check AND vs OR in the from/to selectors — that's the subtlest gotcha.
>
> Standard NetworkPolicy is L3/L4 only. If tenants need to share a service API but with method-level restrictions, that requires CiliumNetworkPolicy L7 rules or a service mesh. And for egress to external FQDNs like Stripe or Twilio, you need either Cilium `toFQDNs` or a proxy — standard NetworkPolicy has no FQDN concept."

---

## Self-test drills

### 1. Design a NetworkPolicy strategy for a cluster running 3 isolated tenants.

**Reference answer:** Namespace-per-tenant with labels. Default-deny-all (both ingress and egress) in each namespace. Allow egress to kube-dns (port 53 UDP+TCP). Allow intra-tenant traffic via namespaceSelector matching the tenant label. Allow egress to shared platform namespace. Allow ingress from ingress controller. Test with `kubectl exec` cross-tenant connectivity checks. Extend to L7 with CiliumNetworkPolicy if method-level restrictions are needed.

### 2. Why does adding egress NetworkPolicy break DNS resolution, and how do you fix it?

**Reference answer:** Egress NetworkPolicy restricts all outbound traffic from selected pods. DNS queries (UDP/TCP port 53 to kube-dns in kube-system) are outbound traffic. If the egress rules don't explicitly allow port 53 to kube-system namespace, DNS resolution silently fails — pods can't resolve any cluster or external hostname. Fix: always include a `toNamespaces: kube-system, ports: 53 UDP/TCP` egress rule when applying egress NetworkPolicy.

### 3. What's the difference between combining podSelector and namespaceSelector in one `from` element vs two separate elements?

**Reference answer:** One combined `from` element with both selectors = AND (pods matching BOTH the namespace label AND the pod label). Two separate `from` elements = OR (pods in the matching namespace OR pods with the matching label, regardless of namespace). The AND version is almost always what you want for isolation — "pods labeled `app: checkout` in the `checkout` namespace only." The OR version accidentally allows checkout-labeled pods from other namespaces too.

### 4. Name three things standard NetworkPolicy cannot do and what you'd use instead.

**Reference answer:** (1) L7 HTTP path/method rules — use CiliumNetworkPolicy with `toPorts.rules.http`. (2) FQDN-based egress (e.g., `api.stripe.com`) — use Cilium `toFQDNs` or an egress proxy (Envoy, Squid). (3) Host-network pod traffic (bypasses NetworkPolicy entirely) — use OPA/Gatekeeper or Kyverno to prevent host-network pods, or manage via node-level firewall rules.

---

## Further reading / watching

- **Kubernetes docs — Network Policies**: `kubernetes.io/docs/concepts/services-networking/network-policies/` — canonical reference
- **Ahmet Balkan — "kubernetes-network-policy-recipes"** on GitHub: copy-paste policy patterns for common scenarios (allow same namespace, deny all, allow from ingress)
- **networkpolicy.io visual editor**: `editor.networkpolicy.io` — draw the intent, get the YAML; or paste YAML and see a diagram. Excellent for learning selector behavior.
- **Cilium docs — CiliumNetworkPolicy**: `docs.cilium.io/en/stable/network/kubernetes/policy/` — L7 policy examples including HTTP, gRPC, Kafka, DNS
- **NeuVector** (open source, acquired by SUSE): a CNCF project for runtime network security that builds on NetworkPolicy concepts — worth knowing exists at the senior level

---

## The 4 dimensions (senior framing)

- **Tech**: default-deny-all applied at namespace creation; AND vs OR selector semantics; kube-dns egress rule as mandatory companion; CiliumNetworkPolicy for L7 and FQDN; test with `hubble observe` or `calicoctl policy trace`.
- **People**: NetworkPolicy complexity compounds fast — 3 tenants, 10 namespaces, 20 policies. Document the policy structure in a README per namespace. Write a runbook for "adding a new service that needs to call an external API." New engineers regularly break things by forgetting DNS egress — make it a policy template, not tribal knowledge.
- **CI/CD**: test network policies in CI using `kubectl exec` smoke tests: "can pod A reach pod B? (should fail)" and "can pod A reach pod C? (should succeed)". Enforce policy structure via Kyverno (e.g., "every namespace must have a default-deny policy"). Use `networkpolicy.io` in PR reviews to visualize the intent of policy changes.
- **Operations**: NetworkPolicy changes are often silent failures (pod can't talk, returns connection refused, looks like an app bug). Add a standard debugging step in your SRE runbook: "check NetworkPolicy before checking app logs." Monitor Cilium/Calico agent health — if the agent restarts, policy may temporarily not be enforced. Alert on policy agent pod restarts.
