# OPA Gatekeeper — the simple version (the law library)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **OPA Gatekeeper enforces Kubernetes policy using Rego — a Turing-complete policy language. More powerful than Kyverno's YAML, but with a real learning cost. The question is always: which tool fits your team?**

---

## The law book vs the checklist

Imagine two ways to train a bouncer:

**Kyverno way — the checklist**: "Here's a laminated card. Shirts must have collars. No sneakers. No hats. Check each item. If the card says yes, let them in."
Fast to write, easy to read, covers 90% of cases.

**OPA Gatekeeper way — the law library**: "Here's a complete legal system. You can encode any rule, any cross-reference, any conditional. You can write rules that check one resource against another. The sky's the limit — but you need to understand the law to write it."
Harder to write, much more expressive.

For most Kubernetes policy needs (block root, require labels, require signed images), the checklist is enough. Gatekeeper's power is for complex cases — multi-resource rules, custom audit logic, policies that span outside Kubernetes.

---

## The Gatekeeper model: ConstraintTemplate + Constraint

Gatekeeper splits policy into two objects:

**`ConstraintTemplate`** — defines the _type_ of rule and its Rego logic. Think of this as the law itself.

**`Constraint`** — applies an instance of that law to a specific scope. Think of this as "enforce this law in the 'production' namespace."

```
ConstraintTemplate (K8sNoRoot) → defines what "no root" means in Rego
Constraint (K8sNoRoot/require-non-root-prod) → apply K8sNoRoot to namespace=production
```

This separation lets you define a rule once and apply it in many contexts with different parameters.

---

## The Rego cost of entry

Kyverno policy looks like a K8s YAML you already know.
Gatekeeper policy looks like this:

```rego
package k8snoroot

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container %v must set runAsNonRoot: true", [container.name])
}
```

This is Rego. It's a logic programming language. If your team knows Rego (or uses OPA outside K8s for Terraform/HTTP API policy), Gatekeeper is a natural fit. If nobody on your team knows Rego, Kyverno wins on operational cost.

---

## When to pick Gatekeeper over Kyverno

| Situation | Pick |
|---|---|
| Your team already uses OPA for Terraform or API policy (one language, everywhere) | Gatekeeper |
| You need to write policies that cross-reference multiple resources | Gatekeeper |
| You need to distribute policies across many clusters with a central policy repo | Gatekeeper (better multi-cluster tooling) |
| Your team is K8s-native and knows YAML well | Kyverno |
| You need image verification / SBOM attestation | Kyverno (verifyImages is richer) |
| Simpler policies, faster onboarding | Kyverno |

Senior answer: "I've used Kyverno in production. I'd pick Gatekeeper when the team has Rego investment or needs cross-resource rules that Kyverno's YAML can't express."

---

## Self-test (the killer question)

Out loud:

> **"OPA Gatekeeper vs Kyverno — when would you pick each?"**

**Reference answer (intuitive version):**

"They solve the same problem — enforcing policy at Kubernetes admission time — but from different angles. Kyverno is YAML-native: policies look like Kubernetes resources, no new language. It's easier to adopt for a team that lives in YAML, and its verifyImages is excellent for supply-chain enforcement. OPA Gatekeeper uses Rego — a proper policy language. The upside is expressiveness: you can write rules that are impossible in Kyverno's YAML, like checking a resource against data from another resource, or enforcing complex multi-tenant rules. The downside is the Rego learning curve.

I'd pick Kyverno for a team that's primarily K8s-focused and wants fast onboarding. I'd pick Gatekeeper if the org already uses OPA for other policy surfaces — Terraform, API authorization, CI — because Rego becomes a shared language across the whole platform instead of a one-off K8s tool."

---

## Further reading

- [OPA Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/)
- [OPA / Rego intro](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gatekeeper library](https://github.com/open-policy-agent/gatekeeper-library) — community-maintained ConstraintTemplates

---

## Next: the deep-dive

When the law-library analogy feels obvious, jump to [`opa-gatekeeper.md`](./deep-dive.md). The deep-dive covers:

- ConstraintTemplate + Constraint in full
- Rego basics applied to K8s policy
- Audit mode
- Multi-cluster policy distribution
- Performance considerations
- The full Kyverno vs Gatekeeper trade-off matrix
- 3 self-test drills + 4-dimensions framing
