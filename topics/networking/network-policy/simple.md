# NetworkPolicy — the simple version (the building security badge analogy)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

One idea:

> **A NetworkPolicy is a badge-reader rule. By default, every pod in a K8s cluster is a building with no security — anyone can walk in. The moment you add one NetworkPolicy that selects a pod, that pod's building gets locked. Then you decide exactly who gets a badge.**

That's it. The surprising part is the default: without any NetworkPolicy, *everything talks to everything*. The moment you add the first policy, you need to add rules for everything you want to allow — or it breaks.

---

## The building security analogy

Your cluster is an office building:

| Building world | K8s world |
|---|---|
| Every room is unlocked by default | Every pod accepts connections from every other pod by default |
| You install a badge reader on one room | You apply a NetworkPolicy that selects a pod |
| Now that room is locked — even the CEO can't get in without a badge | Now that pod drops all traffic not explicitly allowed |
| You issue badges: "QA team allowed 9am–5pm, engineers allowed anytime" | You add ingress rules: "allow from namespace=payments, port 8080" |
| Egress: you also lock the door from the inside so employees can't walk out to certain areas | Egress NetworkPolicy: control what the pod can call out to |
| A room with no badge reader still unlocked | Pods not selected by any NetworkPolicy: open to all traffic |

The trap everyone falls into: they add a NetworkPolicy to lock down the frontend, forget to add a rule allowing kube-dns, and suddenly the pod can't resolve any hostnames. Badge-reader room, no badge for the mailroom that delivers DNS answers.

---

## The whole idea in 2-3 sentences

NetworkPolicy is the K8s API for declaring which pods can talk to which other pods (ingress) and which destinations a pod can reach (egress). It's additive — you start with default-deny (an empty policy that selects all pods) and then add rules for what's allowed. The gotcha: NetworkPolicy is only enforced if your CNI plugin supports it; without Calico or Cilium, the objects are accepted but silently ignored.

---

## The 2 concepts that confuse people

### 1. The "default-deny" pattern — one empty policy locks the whole namespace

An empty NetworkPolicy with `podSelector: {}` (selects all pods in the namespace) and no ingress or egress rules creates an implicit deny-all:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}   # selects ALL pods in this namespace
  policyTypes:
  - Ingress
  - Egress
  # no ingress: or egress: blocks = deny all
```

Apply this first. Then add specific allow rules in separate NetworkPolicy objects. Multiple policies for the same pod are unioned (OR'd) — the pod gets everything all its matching policies allow.

### 2. The kube-dns gotcha

When you add egress NetworkPolicy to a namespace, you must explicitly allow UDP/TCP on port 53 to kube-dns. Otherwise every DNS lookup from the pod silently fails:

```yaml
# Always include this when adding egress policies
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

This is the #1 cause of "I added a NetworkPolicy and now nothing works."

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What's the default? | All pods can talk to all other pods — no policy means no restrictions |
| When does a pod become "locked"? | The moment ANY NetworkPolicy with a matching podSelector is applied |
| How do multiple policies interact? | OR logic — a pod gets the union of all allowed traffic across all matching policies |
| What's a namespaceSelector? | Match pods across namespaces by namespace labels, not pod labels |
| Does NetworkPolicy do L7? | Standard K8s NetworkPolicy: no. CiliumNetworkPolicy: yes (HTTP methods, paths) |
| What about FQDN egress (e.g., api.stripe.com)? | Standard NetworkPolicy: no FQDN support. Cilium adds `toFQDNs`. Otherwise use a proxy. |
| What if my CNI doesn't support it? | Objects apply, but no enforcement — silently open. Calico and Cilium both enforce. |
| Does NetworkPolicy cover node-to-pod traffic? | No — host network traffic bypasses NetworkPolicy. Use OPA/Kyverno or node firewalls. |

---

## Self-test (the killer interview question)

Out loud:

> **"Design a NetworkPolicy strategy for a cluster running 3 isolated tenants."**

**Reference answer (intuitive version):**

"I'd start with **namespace isolation as the foundation**. Each tenant gets their own namespace (or set of namespaces). Then I apply a `default-deny-all` NetworkPolicy in each tenant namespace — both ingress and egress. That locks each namespace immediately.

Then I add explicit allow rules for what each tenant actually needs:
- Allow intra-tenant traffic: `namespaceSelector` matching the tenant's own namespace labels
- Allow egress to kube-dns (port 53) — this one gets forgotten and breaks everything
- Allow egress to any shared services (monitoring, logging) from the platform namespace
- Allow ingress from the ingress controller namespace to whatever pods serve public traffic

For the senior layer: standard NetworkPolicy gets you L3/L4. If I need 'tenant A can only call the /read endpoints of the shared API, not /write', that's CiliumNetworkPolicy L7 rules. And I'd test the policies — not just apply them. I'd run `kubectl exec` smoke tests from a pod in one tenant trying to reach a pod in another tenant and assert it fails. Policies that aren't tested don't exist."

---

## Further reading / watching

- **Kubernetes docs — Network Policies**: `kubernetes.io/docs/concepts/services-networking/network-policies/` — the canonical reference
- **Ahmet Balkan — "NetworkPolicy recipes"** GitHub repo: search "ahmetb kubernetes-network-policy-recipes" — pre-built policy YAMLs for common patterns (deny all, allow same namespace, allow from ingress)
- **Cilium NetworkPolicy editor**: `editor.networkpolicy.io` — visual editor for building policies; paste YAML and see a diagram of what's allowed. Invaluable for learning.
- **Liz Rice — "Container Security"** (O'Reilly book, chapter on network isolation): the NetworkPolicy chapter is the clearest explanation of the pod selector + namespace selector interaction

---

## Next: the deep-dive

When the badge-reader analogy and the kube-dns gotcha feel obvious, jump to [`network-policy.md`](./deep-dive.md). The deep-dive covers:

- The default-deny pattern in detail (and why you need two separate policies for ingress + egress)
- Pod selector vs namespace selector — the exact match semantics
- The "I added a policy and now nothing works" debugging flow (step by step)
- Standard NetworkPolicy limitations: no L7, no FQDN, no host network
- CiliumNetworkPolicy L7 patterns for senior-level answers
- Multi-tenant strategy with real YAML examples
- The 4-dimensions framing (Tech / People / CI/CD / Operations)
