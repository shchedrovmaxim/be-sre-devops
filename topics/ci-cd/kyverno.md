# Kyverno — the deep-dive

> **Goal**: by the end you can answer the killer question — **"How would you use Kyverno to enforce 'no runAsRoot, no privileged, all images signed' across the cluster?"** — naming ClusterPolicy CRD, validate/mutate/generate/verifyImages modes, match/exclude selectors, background scanning, PolicyExceptions, and ClusterPolicyReport.

> You use Kyverno in production. This is your home tool. Use this doc to sharpen interview articulation, not to learn basics.

> Start with [kyverno-simple.md](./kyverno-simple.md) if you want the quick framing.

---

## The senior framing — policy as code, not configuration

Mid-level: "we have Kyverno policies that block runAsRoot."
Senior: "Kyverno is our policy-as-code layer. Policies live in Git alongside application manifests, reviewed in PRs, tested in the Kyverno Playground before merge. Enforcement is at admission time. Background scanning surfaces existing drift. PolicyExceptions make the escape hatch explicit and auditable — no more undocumented one-offs in the cluster."

The interview signal is articulating the governance model, not just the YAML.

---

## The ClusterPolicy CRD — anatomy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy        # cluster-scoped; use Policy for namespace-scoped
metadata:
  name: pod-security-baseline
  annotations:
    policies.kyverno.io/category: "Pod Security"
    policies.kyverno.io/severity: "high"
    policies.kyverno.io/description: "Enforce baseline pod security: no root, no privileged, require signed images."
spec:
  validationFailureAction: Enforce  # or Audit
  background: true                   # run background scans on existing resources
  rules:
    - name: <rule-name>
      match:
        any:
          - resources:
              kinds: ["Pod"]         # resource kinds this rule applies to
              namespaces: []         # empty = all namespaces
      exclude:
        any:
          - resources:
              namespaces: ["kube-system"]  # exempt kube-system
      # then one of: validate | mutate | generate | verifyImages
```

`ClusterPolicy` applies across all namespaces. `Policy` (namespace-scoped) applies only within its namespace — useful for tenant-level policies in a multi-tenant cluster.

---

## Rule type: validate

Checks that a resource matches (or doesn't match) a pattern. Blocks or logs violations.

```yaml
    - name: no-run-as-root
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Pods must set runAsNonRoot: true in the pod securityContext."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true

    - name: no-privileged-containers
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Privileged containers are not allowed."
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[].securityContext.privileged | contains(@, true) }}"
                operator: Equals
                value: true
```

Two validation styles:
- **`pattern`**: the resource must match this structure. Missing fields or non-matching values → violation.
- **`deny`**: use JMESPath expressions to evaluate conditions. More expressive — lets you check across arrays (every container's securityContext).

### The JMESPath pattern gotcha

When checking container-level settings across all containers, `pattern` only checks the first container. For arrays you need `deny` with a JMESPath query:

```yaml
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[*].securityContext.privileged }}"
                operator: AnyIn
                value: [true]
```

This checks every container, not just the first.

---

## Rule type: mutate

Modifies resources before they're admitted. Use sparingly — implicit mutation makes debugging harder.

```yaml
    - name: add-run-as-non-root
      match:
        any:
          - resources:
              kinds: ["Pod"]
      mutate:
        patchStrategicMerge:
          spec:
            securityContext:
              +(runAsNonRoot): true   # + prefix: only add if not already present
```

The `+(field)` syntax means "set this only if the field doesn't already exist." Without `+`, you'd overwrite any existing value — usually not what you want.

Common mutation use cases:
- Add default `securityContext` fields
- Inject labels or annotations (e.g., team ownership label)
- Set `imagePullPolicy: Always` for `:latest` tags
- Add a `tolerations` entry for infrastructure nodes

**The mutate-vs-validate debate**: mutate is convenient but hides problems. If a developer submits a pod without `runAsNonRoot` and mutate adds it silently, they don't learn their Dockerfile/manifest is wrong. A validate rule with a helpful message teaches. Use mutate for operational concerns (adding labels, pull policy), not for security-critical fields.

---

## Rule type: generate

Creates new resources when a matching resource is created.

```yaml
    - name: create-default-networkpolicy
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-ingress
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true   # keep the generated resource in sync; delete it if the source is deleted
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress"]
```

`synchronize: true` means Kyverno owns the generated resource — it will recreate or update it if someone deletes or modifies it directly. This is powerful: it means the policy is self-healing, not just one-time.

Use cases:
- Default NetworkPolicy on Namespace creation
- Default ResourceQuota / LimitRange
- Default RBAC bindings for new namespaces (e.g., give the team's service account view access)

---

## Rule type: verifyImages

Validates that images in a pod are signed with a trusted Cosign signature. This is the bridge to your production Cosign setup.

```yaml
    - name: require-signed-images
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/myorg/*"
          mutateDigest: true       # rewrite image tag to digest for immutability
          verifyDigest: true       # ensure the digest hasn't changed since signing
          required: true           # pod is denied if no signature found
          attestors:
            - count: 1             # at least 1 of the following must verify
              entries:
                - keyless:
                    subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
          attestations:
            - type: https://cyclonedx.org/bom
              required: true
              attestors:
                - entries:
                    - keyless:
                        subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
                        issuer: "https://token.actions.githubusercontent.com"
```

`mutateDigest: true` is a hidden gem: Kyverno rewrites `myapp:v1.2.3` to `myapp@sha256:...` in the pod spec. This prevents a supply-chain attack where a tag is overwritten — the pod always runs exactly the image that was signed.

---

## match/exclude selectors in depth

Kyverno's `match`/`exclude` logic is flexible but has gotchas.

```yaml
      match:
        any:                         # OR across entries
          - resources:
              kinds: ["Pod"]
              namespaces: ["production", "staging"]
          - resources:
              kinds: ["Pod"]
              selector:
                matchLabels:
                  env: production    # also match by label
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "cert-manager"]  # exclude system namespaces
          - subjects:                # exclude when submitted by these service accounts
              - kind: ServiceAccount
                name: system-deployer
                namespace: kube-system
```

Key points:
- `any` in `match` is OR: "match if any of these is true."
- `any` in `exclude` is OR: "exclude if any of these is true."
- `all` (not shown) is AND: "match only if all of these are true."
- `subjects` lets you exclude specific users or service accounts — useful for platform-team tooling that legitimately needs to create privileged pods.

### The kube-system gotcha

Never enforce security policies against `kube-system` unless you know what you're doing. System components (`kube-proxy`, `coredns`, `node-local-dns`) often run as root by design. Excluding `kube-system` in `exclude` is standard practice.

---

## Background scanning and PolicyReport

Kyverno scans existing resources periodically (default: every 1 hour) against all policies with `background: true`. Results land in:

- `PolicyReport` (namespace-scoped): reports for resources in a specific namespace
- `ClusterPolicyReport` (cluster-scoped): reports for cluster-scoped resources

```bash
# See all violations across the cluster
kubectl get clusterpolicyreport -A
kubectl get policyreport -A

# Detailed view of a report
kubectl describe policyreport -n production
```

The `PolicyReport` CRD follows the [Policy Reports spec](https://github.com/kubernetes-sigs/wg-policy-prototypes) — same format used by other policy tools, so dashboards like Grafana or tools like Policy Reporter can consume them.

**Interview framing**: background scanning means you're not just reactive (blocking new violations) but proactive (auditing the whole cluster continuously). This is how you discover drift — something that was compliant when the policy was written but is now in violation due to a manual change or a chart upgrade.

---

## PolicyException — the auditable escape hatch

```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: legacy-root-exception
  namespace: legacy-namespace
  annotations:
    ticket: "PLAT-4321"
    review-date: "2026-09-05"
    owner: "platform-team"
spec:
  exceptions:
    - policyName: pod-security-baseline
      ruleNames: ["no-run-as-root"]
  match:
    any:
      - resources:
          kinds: ["Pod"]
          namespaces: ["legacy-namespace"]
          names: ["legacy-app-*"]
```

Key design points:
- **Scoped**: only applies to `legacy-app-*` pods in `legacy-namespace`. Not a blanket policy disable.
- **Documented**: the annotations carry the ticket reference and review date. When someone reviews the cluster's policy exceptions, they know why this exists and when to revisit.
- **Reviewable**: the `PolicyException` is a K8s resource, committed to Git, reviewed in a PR.
- **Audit trail**: `ClusterPolicyReport` still shows the exception matches — the violation is recorded even though admission was allowed.

The senior posture: "we have no policies disabled; we have `PolicyException` CRDs for documented legitimate cases, each with a review date."

---

## The full enforcement policy (comprehensive example)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pod-security-and-supply-chain
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: no-run-as-root
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "cert-manager", "monitoring"]
      validate:
        message: "Set runAsNonRoot: true in pod or container securityContext."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true

    - name: no-privileged
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system"]
      validate:
        message: "Privileged containers are not allowed."
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[*].securityContext.privileged }}"
                operator: AnyIn
                value: [true]
              - key: "{{ request.object.spec.initContainers[*].securityContext.privileged }}"
                operator: AnyIn
                value: [true]

    - name: require-signed-images
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["production", "staging"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/*"
          mutateDigest: true
          required: true
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/myorg/myrepo/.github/workflows/build.yaml@refs/heads/main"
```

---

## The interview answer in 60 seconds

> "I'd write a single ClusterPolicy with `validationFailureAction: Enforce` and three rules. First, a validate rule checking that `runAsNonRoot: true` is set in the pod's securityContext — with a helpful message explaining why. Second, a validate rule with a JMESPath deny condition checking every container's securityContext for `privileged: true` — the pattern approach only checks the first container, so you need JMESPath for array traversal. Third, a verifyImages rule requiring a valid Cosign keyless signature from our CI pipeline for any image from our registry — with `mutateDigest: true` to rewrite tags to digests, preventing tag-mutable supply-chain attacks.
>
> I'd exclude kube-system and system namespaces from the security rules — system components often legitimately run as root. For legitimate edge cases, I'd use PolicyException CRDs — scoped, documented with a ticket reference and review date, committed to Git.
>
> Background scanning is on by default, so existing resources are continuously audited. Violations appear in ClusterPolicyReport. We feed that into Policy Reporter for a dashboard view. The senior point: policy is code — reviewed in PRs, tested in the Kyverno Playground, audited in Git. No undocumented exceptions."

---

## Self-test drills

### 1. What's the difference between validate pattern and validate deny in Kyverno?

**Reference answer:**
- **`pattern`**: the resource must structurally match the given pattern. Good for simple field checks on scalar values.
- **`deny`**: evaluates JMESPath conditions and blocks if they evaluate to true. Required when you need to check array elements (e.g., every container's securityContext), logical combinations, or complex expressions.
- Gotcha: `pattern` on a container securityContext only checks the first container in the list. For "no container in the pod is privileged," you need a `deny` with `spec.containers[*].securityContext.privileged`.

### 2. What does mutateDigest: true do in a verifyImages rule, and why does it matter?

**Reference answer:**
- Kyverno rewrites the image reference in the pod spec from a tag (`myapp:v1.2.3`) to a digest (`myapp@sha256:abc123...`).
- This means the pod will always run exactly the image that was signed and verified, regardless of what the tag points to in the future.
- Without this, a supply-chain attacker who overwrites the tag in the registry after signing would cause pods to run a different, unsigned image under the same tag.
- It also makes rollbacks deterministic: the exact image digest is recorded in the pod spec.

### 3. How would you handle a legacy app that genuinely needs to run as root?

**Reference answer:**
- Create a `PolicyException` CRD, scoped to the specific namespace and pod name pattern.
- Include annotations: ticket reference (why it's needed), review date (when to reassess), owner team.
- The exception is committed to Git, reviewed in a PR — it's documented and auditable.
- The `ClusterPolicyReport` will still record the exception matches, so it's not invisible in dashboards.
- Set a calendar reminder for the review date: is this still necessary? Can the app be updated?
- Don't disable the policy. Don't use a broad `namespaces: ["*"]` exclude. Scope it tightly.

---

## Further reading

- [Kyverno docs](https://kyverno.io/docs/) — especially the policies reference and verifyImages
- [Kyverno Playground](https://playground.kyverno.io/) — test policies against sample resources without a cluster
- [Kyverno Policy Library](https://kyverno.io/policies/) — community-maintained policies; start here before writing from scratch
- [Policy Reporter](https://github.com/kyverno/policy-reporter) — dashboard and alerting for PolicyReport CRDs
- [wg-policy-prototypes](https://github.com/kubernetes-sigs/wg-policy-prototypes) — the PolicyReport spec used by Kyverno and other engines

---

## The 4 dimensions (senior framing)

- **Tech**: ClusterPolicy CRD; validate/mutate/generate/verifyImages modes; JMESPath for array traversal; `mutateDigest: true` for immutable image references; background scanning → ClusterPolicyReport. PolicyException for auditable escape hatches.
- **People**: rolling out Enforce to an existing cluster is risky — start in Audit mode for 2 weeks. The ClusterPolicyReport tells you what would have been blocked. Fix violations or document PolicyExceptions. Only then flip to Enforce. Communicate to dev teams: the Kyverno Playground lets them test their manifests before submitting.
- **CI/CD**: run `kyverno apply <policy.yaml> --resource <manifest.yaml>` in CI to test manifests against policies before they reach the cluster. This catches violations in PRs, not at deploy time. Add to the platform team's GitHub Actions workflow template. Also run dry-run or `kyverno test` against the policy repository itself.
- **Operations**: monitor Kyverno admission webhook latency (it's in the critical path of every pod create). Set reasonable `failurePolicy` — `Ignore` means policy errors allow the resource (availability over security); `Fail` means policy errors block the resource (security over availability). For core policies (security), `Fail`. For generate rules, `Ignore` is safer. Keep Kyverno updated — admission webhooks with bugs can block your entire cluster.
