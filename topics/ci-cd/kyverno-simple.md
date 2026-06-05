# Kyverno — the simple version (the bouncer + auto-fixer)

> Read this first. Once the analogy clicks, the [deep-dive](./kyverno.md) is easy.

This doc explains **one idea**:

> **Kyverno is a policy engine that intercepts every Kubernetes resource before it's admitted and either blocks it, fixes it, or generates related resources — all configured in plain YAML, no new language to learn.**

You use Kyverno in production. This is your home tool. This doc is the mental model to articulate it clearly in an interview.

---

## The nightclub bouncer (who also adjusts your outfit)

Imagine a nightclub with a very particular dress code. At the door:

- **The bouncer (validate)** checks: "are you wearing a collared shirt? No sneakers?" If not — you're not getting in.
- **The stylist (mutate)** adjusts things: "your collar is popped — let me fix that." You didn't know the rule, but you're now compliant before you walk in.
- **The badge maker (generate)** creates things: "new VIP member? Here's your loyalty card automatically."

Kyverno plays all three roles for every resource that enters your Kubernetes cluster.

| Role | Kyverno mode | Example |
|---|---|---|
| Bouncer | `validate` | Block pods that run as root |
| Stylist | `mutate` | Auto-add `runAsNonRoot: true` if missing |
| Badge maker | `generate` | Auto-create a NetworkPolicy when a new Namespace is created |

There's also a fourth: `verifyImages` — the bouncer who checks that your ID (image signature) is authentic.

---

## Why YAML matters here

Other policy engines (like OPA/Gatekeeper) use **Rego** — a purpose-built policy language. Powerful, but you have to learn it.

Kyverno policies are just Kubernetes YAML — the same syntax you already use everywhere. If you know how to write a PodSpec, you can write a Kyverno policy. That's a big operational win for a team where everyone is already fluent in YAML but few know Rego.

---

## The quick example — block runAsRoot + require signed images

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: baseline-pod-security
spec:
  validationFailureAction: Enforce   # block, don't just audit
  rules:
    # Rule 1: no root
    - name: no-run-as-root
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Running as root is not allowed."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true

    # Rule 2: signed images only
    - name: require-signed-images
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/myorg/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
```

That's two rules: no root, and images must be signed by your CI pipeline. Both in YAML, no Rego.

---

## Background scanning vs admission-time

Kyverno enforces at **admission time** (when you try to apply or create a resource). But it also **scans existing resources** periodically in the background.

This means:
- New resources are blocked or fixed before they're admitted.
- Old resources that pre-date the policy are also evaluated — violations show up in `PolicyReport` / `ClusterPolicyReport` CRDs.

In an interview: "we audit the whole cluster, not just new resources."

---

## The `PolicyException` escape hatch

Sometimes you have a legitimate reason to break the rule — a legacy app that genuinely needs root, a debug pod during incident response.

```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: legacy-app-root-exception
  namespace: legacy
spec:
  exceptions:
    - policyName: baseline-pod-security
      ruleNames: ["no-run-as-root"]
  match:
    any:
      - resources:
          kinds: ["Pod"]
          namespaces: ["legacy"]
          names: ["legacy-app-*"]
```

This grants a scoped exception — only for `legacy-app-*` pods in the `legacy` namespace, only for the `no-run-as-root` rule. It's documented, tracked in Git, and reviewable. No blanket disabling of the policy.

---

## Self-test (the killer question)

Out loud:

> **"How would you use Kyverno to enforce 'no runAsRoot, no privileged, all images signed' across the cluster?"**

**Reference answer (intuitive version):**

"I'd write a ClusterPolicy with `validationFailureAction: Enforce` — so violations are blocked at admission, not just logged. Three rules: a validate rule checking `runAsNonRoot: true` in the pod's securityContext, another validate rule blocking `privileged: true` in any container's securityContext, and a verifyImages rule checking that every image from our registry has a valid Cosign keyless signature from our CI pipeline. I'd also enable Kyverno's background scanning so existing resources that pre-date the policy are also audited — violations show up in ClusterPolicyReport. For legitimate exceptions — say, a legacy app that genuinely needs root — I'd use a PolicyException CRD, scoped to just that namespace and workload, tracked in Git like any other resource."

---

## Further reading

- [Kyverno docs](https://kyverno.io/docs/) — start with the policies reference
- [Kyverno Playground](https://playground.kyverno.io/) — test policies without a cluster
- [kyverno.io/policies](https://kyverno.io/policies/) — the community policy library

---

## Next: the deep-dive

When the bouncer analogy feels obvious, jump to [`kyverno.md`](./kyverno.md). The deep-dive covers:

- ClusterPolicy CRD in full
- validate / mutate / generate / verifyImages mechanics
- match/exclude selectors in detail
- Background scanning vs admission-time enforcement
- PolicyException for edge cases
- ClusterPolicyReport querying
- 4-dimensions framing + 3 drills
