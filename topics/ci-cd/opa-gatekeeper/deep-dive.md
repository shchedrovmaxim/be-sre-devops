# OPA Gatekeeper — the deep-dive

> **Goal**: by the end you can answer the killer question — **"OPA Gatekeeper vs Kyverno — when would you pick each?"** — naming ConstraintTemplate + Constraint, Rego trade-offs, audit mode, multi-cluster distribution, and the "Kyverno is YAML-native and simpler for K8s; OPA is more powerful and works outside K8s" framing.

> Start with [opa-gatekeeper-simple.md](./simple.md) for the quick framing.

---

## The senior framing — the Rego investment

The Gatekeeper vs Kyverno decision is fundamentally about your team's Rego investment. OPA and Rego appear in multiple places in a modern platform:

- Kubernetes admission (Gatekeeper)
- Terraform plan validation (Conftest + Rego)
- HTTP API authorization (OPA sidecar)
- CI pipeline policy checks (Conftest)

If your org uses Rego in multiple layers, Gatekeeper is a natural extension — one language, one mental model, one policy library. If Kubernetes is the only policy surface, Kyverno's YAML is almost always the lower-cost choice.

---

## The model: ConstraintTemplate + Constraint

Gatekeeper's split into two CRDs is the key architectural distinction:

### ConstraintTemplate — the law

Defines a new CRD type (the constraint kind) and the Rego logic that enforces it.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoroot
spec:
  crd:
    spec:
      names:
        kind: K8sNoRoot     # this creates a new CRD called K8sNoRoot
      validation:
        openAPIV3Schema:
          type: object
          properties:
            parameters:
              type: object   # define any parameters the constraint can pass to Rego
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoroot

        violation[{"msg": msg}] {
          # iterate every container
          container := input.review.object.spec.containers[_]
          # check if runAsNonRoot is not set or false
          not container.securityContext.runAsNonRoot == true
          msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
        }

        violation[{"msg": msg}] {
          # also check init containers
          container := input.review.object.spec.initContainers[_]
          not container.securityContext.runAsNonRoot == true
          msg := sprintf("Init container '%v' must set runAsNonRoot: true", [container.name])
        }
```

The `violation` rule is the contract: if it produces any messages, the resource is rejected (or logged in audit mode).

### Constraint — the applied law

Creates an instance of the `K8sNoRoot` kind and specifies where it applies.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoRoot
metadata:
  name: require-non-root-pods
spec:
  enforcementAction: deny    # or: warn, dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
      matchExpressions:
        - key: gatekeeper-policy
          operator: NotIn
          values: ["exempt"]   # namespaces labeled gatekeeper-policy=exempt are excluded
  parameters: {}               # passed to input.parameters in Rego
```

`enforcementAction`:
- `deny` — blocks the resource. Default secure behavior.
- `warn` — admits the resource but adds a warning annotation. Good for rollout.
- `dryrun` — audit only, no enforcement. Equivalent to Kyverno's `Audit` mode.

---

## Rego basics for K8s policy

The `input` object in a Gatekeeper Rego policy:

```
input.review.object      → the K8s resource being admitted
input.review.userInfo    → the user/service account submitting the request
input.parameters         → parameters from the Constraint spec
```

A simple `no-privileged` rule:

```rego
package k8sprivileged

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Privileged container '%v' is not allowed", [container.name])
}
```

The `_` in `containers[_]` is Rego's "iterate all" — equivalent to a for-each. `violation` collects all messages; if the set is non-empty, the resource is rejected.

### Cross-resource rules — Gatekeeper's killer feature

Kyverno can't easily check "does this service exist in the namespace before allowing the pod?" Gatekeeper can, via **external data** (OPA's `data` store, or via Gatekeeper's `ExternalData` provider):

```rego
package k8srequiredservice

violation[{"msg": msg}] {
  # Check that the namespace has a required "billing-label" service
  service := data.inventory.namespace[input.review.object.metadata.namespace]["v1"]["Service"]
  not service["billing-label"]
  msg := "Namespace must have a 'billing-label' Service before pods are admitted."
}
```

`data.inventory` is Gatekeeper's cache of existing cluster resources (synced via the `Config` CRD). This cross-resource check is impossible in Kyverno without a custom webhook.

---

## Audit mode and constraint reporting

Gatekeeper runs an audit loop (default: every 60 seconds) that evaluates all `dryrun` and `deny` constraints against existing cluster resources.

```bash
# Check audit results
kubectl get k8snoroot require-non-root-pods -o jsonpath='{.status.violations}'
```

Results appear in the constraint's `.status.violations` field:

```json
[
  {
    "kind": "Pod",
    "name": "legacy-app-abc",
    "namespace": "legacy",
    "message": "Container 'app' must set runAsNonRoot: true"
  }
]
```

This is surfaced differently from Kyverno's `PolicyReport` CRD — Gatekeeper embeds violations in the constraint itself, while Kyverno uses a separate report CRD. Both approaches are queryable; the format differs.

---

## Multi-cluster policy distribution

This is one area where Gatekeeper has historically been stronger.

### OPA Bundle Server

OPA (and by extension Gatekeeper) natively supports a **bundle server** pattern: a central server serves a bundle of Rego policies + data, and each OPA instance polls for updates. For Kubernetes: a Gatekeeper instance polls your policy server and syncs the latest ConstraintTemplates.

### GitOps approach (common in practice)

More commonly: store ConstraintTemplates + Constraints in a Git repo. Use ArgoCD or Flux to sync them to each cluster. The same ApplicationSet generator (see `argocd-appofapps-applicationset.md`) that deploys apps can deploy policies.

For Kyverno, the exact same approach applies — policies are K8s resources, so GitOps handles them identically. Kyverno doesn't have a disadvantage here in practice.

---

## Performance considerations

Gatekeeper is an admission webhook — it's in the critical path of every resource create/update.

- **Rego evaluation time**: complex Rego with large `data.inventory` queries can be slow. OPA benchmarks individual rule evaluations; keep them under 50ms.
- **Webhook timeout**: if Gatekeeper doesn't respond within the webhook timeout, K8s falls back to `failurePolicy` (Fail or Ignore). For security-critical policies: `failurePolicy: Fail`. For less critical: `Ignore`.
- **Memory footprint**: `data.inventory` caches existing cluster resources in memory. In large clusters (10k+ pods), this can be significant. Configure `Config` to sync only the resource kinds you need.

Kyverno has similar performance characteristics — it's also an admission webhook. The difference: Rego's evaluation model (unification/backtracking) can be slower than Kyverno's YAML pattern matching for simple cases, but faster for complex cross-resource queries.

---

## The Gatekeeper library

The community maintains a library of pre-built ConstraintTemplates: [github.com/open-policy-agent/gatekeeper-library](https://github.com/open-policy-agent/gatekeeper-library).

Covers all the standard baselines:
- Pod Security Standards (baseline + restricted)
- Required labels
- Allowed repositories
- Replica limits
- Resource quota enforcement

This dramatically lowers the Rego cost: you might never need to write Rego from scratch for standard K8s policies. Just use the library's ConstraintTemplates and write Constraints.

---

## Kyverno vs Gatekeeper — the full trade-off matrix

| Dimension | Kyverno | OPA Gatekeeper |
|---|---|---|
| **Policy language** | YAML (K8s-native) | Rego (general-purpose logic language) |
| **Learning curve** | Low (if you know K8s YAML) | High (Rego is a different paradigm) |
| **Cross-resource rules** | Limited (some via JMESPath lookups) | Native (data.inventory + ExternalData) |
| **Image signing enforcement** | Excellent (verifyImages built-in) | Limited (no native Cosign integration) |
| **SBOM attestation** | Excellent (verifyImages attestations) | Limited |
| **Mutation** | Full-featured | Limited (Gatekeeper is primarily validating) |
| **Generate** | Yes (generate policies) | No |
| **Audit / background scan** | ClusterPolicyReport CRD | constraint.status.violations |
| **Multi-cluster** | GitOps (ArgoCD/Flux); same as apps | Bundle server + GitOps |
| **Policy library** | Large community library | Large community library (gatekeeper-library) |
| **Works outside K8s** | No (K8s-only) | Yes (OPA + Rego for Terraform, API authz, CI) |
| **Image verification** | Built-in, excellent | Manual (custom Rego + webhook) |
| **Ecosystem fit** | Pure K8s shops | Orgs using OPA across multiple layers |

---

## The interview answer in 60 seconds

> "OPA Gatekeeper and Kyverno both do Kubernetes admission policy, but from different angles. Kyverno is YAML-native: policies look like K8s resources, validate/mutate/generate/verifyImages all in the same tool. Low barrier to entry, excellent supply-chain enforcement with verifyImages. The limit is expressiveness — complex cross-resource rules are awkward.
>
> Gatekeeper uses Rego — a proper logic language. The killer feature is cross-resource rules: checking whether a namespace has a required resource before admitting a pod, for example. And the bigger win for an org that already uses OPA is a shared language: one policy language for K8s admission, Terraform validation, API authorization, and CI checks. You build Rego skills once and apply them everywhere.
>
> I've used Kyverno in production and it covers 95% of what we need. I'd recommend Gatekeeper when the org has existing OPA/Rego investment or has complex cross-resource rule requirements that Kyverno can't express cleanly."

---

## Self-test drills

### 1. Explain the ConstraintTemplate + Constraint split. Why two objects?

**Reference answer:**
- `ConstraintTemplate` defines the rule type and Rego logic — it's the policy definition. It creates a new CRD kind.
- `Constraint` is an instance of that type — it says "apply this rule here, with these parameters."
- The split allows: one policy definition reused across many contexts. Example: one `K8sNoRoot` ConstraintTemplate, but three Constraints applying it to `production`, `staging`, and `dev` with different parameters (e.g., different enforcement actions per environment).
- It also mirrors how RBAC works: ClusterRole (defines permissions) + ClusterRoleBinding (applies to who/where). Same conceptual model.

### 2. What's the Rego `_` operator and how does it apply to container policy?

**Reference answer:**
- `_` in Rego is the anonymous variable — "for each element in this array." `containers[_]` iterates every element.
- In K8s policy: `container := input.review.object.spec.containers[_]` binds each container in turn.
- The `violation` rule fires for each container that matches the condition. All messages are collected; if any exist, the resource is rejected.
- This is how you enforce "every container in the pod must be non-root" — you iterate all containers and collect violations for any that don't comply.

### 3. When would you use Gatekeeper's dryrun enforcementAction instead of deny?

**Reference answer:**
- `dryrun` audits existing resources without blocking new ones. Use it to:
  - Assess blast radius before enabling enforcement: "how many existing pods would be blocked if I set this to deny?"
  - Onboard a new cluster to existing policies: see what's currently non-compliant before blocking.
- The equivalent in Kyverno: `validationFailureAction: Audit`.
- Standard rollout: `dryrun` → fix violations or write exceptions → `warn` (admit with warning) → `deny`. Never flip directly to `deny` on a production cluster without the dry-run phase.

---

## Further reading

- [OPA Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/)
- [OPA Rego language reference](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gatekeeper library](https://github.com/open-policy-agent/gatekeeper-library) — pre-built ConstraintTemplates
- [Conftest](https://www.conftest.dev/) — OPA + Rego for CI policy checks (Terraform plans, K8s manifests, Helm charts)
- [OPA playground](https://play.openpolicyagent.org/) — test Rego policies without a cluster

---

## The 4 dimensions (senior framing)

- **Tech**: ConstraintTemplate defines Rego + new CRD; Constraint applies it. `deny` / `warn` / `dryrun` enforcement modes. Cross-resource rules via `data.inventory`. Audit loop surfaces existing violations in constraint `.status.violations`. Gatekeeper library for standard policies.
- **People**: Rego is the barrier. Before choosing Gatekeeper, assess: does anyone on the team know Rego? If not, the Gatekeeper library reduces the write-from-scratch cost, but debugging policy failures still requires reading Rego. Run a 1-hour Rego workshop before adopting. If the team bounces off Rego, Kyverno is the better choice — a policy engine that nobody uses is worse than no policy.
- **CI/CD**: use Conftest in CI for shift-left policy validation (Terraform plans, K8s manifests). Conftest uses the same Rego packages — write the policy once, use it in CI and in the cluster. Store ConstraintTemplates + Constraints in Git, deploy via ArgoCD to every cluster. Gate policy PR merges on Conftest tests (`conftest test` in CI).
- **Operations**: `failurePolicy: Fail` for security-critical constraints — if Gatekeeper is down, K8s blocks the request. Set Gatekeeper's PodDisruptionBudget to ensure HA of the webhook pods. Monitor webhook latency — it's in the critical path. Alert on Gatekeeper pod restarts. For large clusters, tune `data.inventory` to sync only needed resource kinds; don't sync everything or memory usage spikes.
