# Pod Security Standards — the deep-dive

> **Goal**: by the end you can answer the killer question — **"PSS replaced PodSecurityPolicy. Walk me through how the new model works and how I'd enforce 'restricted' across the cluster."** — naming the three levels, three modes, namespace label API, PSP removal story, and the Kyverno layer for custom rules.

> Start with the [simple version](./simple.md) if you haven't read it. The nightclub bouncer analogy is the spine.

---

## The senior framing — PSP's failure was complexity, not intent

PSP was good in theory: gate every pod through a set of security checks. But it had two fatal flaws:

1. **RBAC coupling was confusing**: a PSP only applied if the pod's ServiceAccount had `use` permission on that PSP. Teams would create a catch-all PSP that allowed everything and bind it to `system:authenticated` just to stop the noise. The result was a security control that looked like it was active but wasn't enforcing anything.

2. **Mutation was surprising**: PSP could mutate pods (auto-fill fields). An engineer thought they were running a privileged container; PSP silently changed it. This made debugging very hard.

PSS strips away the mutation and the RBAC coupling. It's simpler, harder to misconfigure into "effectively off," and built into the apiserver with no extra components.

The gap it leaves — custom rules — is intentionally punted to Kyverno or OPA Gatekeeper. That's a feature: PSS does the 80% case cleanly; you bring your own tool for the 20%.

---

## The three levels — what they actually restrict

### Privileged

No restrictions. Equivalent to no PSP or a fully permissive PSP. Used for:
- System namespaces: `kube-system`, `kube-public`
- Monitoring agents that need node-level access (Falco, Datadog agent, Prometheus node-exporter)
- CNI plugins

```yaml
# Namespace for system stuff — no PSS enforcement
labels:
  pod-security.kubernetes.io/enforce: privileged
```

### Baseline

Prevents known privilege escalations. Most application workloads that just want to "run my app" should clear baseline:

| What it blocks | Example of non-compliant spec |
|---|---|
| `hostProcess: true` (Windows) | Windows-only; blocks escape to host |
| `hostNetwork: true` | Pod shares node's network namespace |
| `hostPID: true` | Pod sees all host processes |
| `hostIPC: true` | Pod shares host IPC namespace |
| `privileged: true` | Privileged container (root on the host) |
| Capabilities: adding beyond a safe set | `capabilities.add: ["NET_ADMIN", "SYS_ADMIN"]` |
| Host path volumes | `hostPath:` volumes (can read anything on the node) |
| `allowPrivilegeEscalation` not explicitly false | Allows `sudo`-style escalation |
| Specific `seccompProfile.type` violations | `Unconfined` seccomp |
| `runAsUser: 0` explicitly | Root user |
| Unsafe `sysctl` values | `sysctl: net.core.somaxconn=65535` if kernel.* or net.* not allowed |

### Restricted

Everything in baseline, plus:

| Additional requirement | What it means |
|---|---|
| `runAsNonRoot: true` | Container must not run as UID 0. |
| `allowPrivilegeEscalation: false` | Must be explicitly set to false. |
| Capabilities: `drop: [ALL]` required | Must drop all capabilities. Can add a tiny subset (`NET_BIND_SERVICE` is allowed). |
| `seccompProfile.type: RuntimeDefault` or `Localhost` | Must specify a seccomp profile. |
| `volumes`: only safe volume types | ConfigMap, Secret, projected, serviceAccountToken, emptyDir, ephemeral, CSI, image, PVC. No hostPath, no NFS, no direct CSI driver volumes. |

The restricted level is the right target for any internet-facing service.

---

## The three modes

| Mode | K8s API behavior | Pod outcome |
|---|---|---|
| `enforce` | Admission webhook **rejects** the pod object | Pod never created |
| `audit` | Admission allows the pod; violation written to **audit log** | Pod runs; violation logged |
| `warn` | Admission allows the pod; violation written to **admission warnings** | Pod runs; warning shown in `kubectl` output |

Modes are per-level, per-namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    # Enforce: block anything that's not at least baseline
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.28

    # Audit and warn: also check against restricted and log/warn violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
```

The `-version` label pins the PSS check to a specific K8s version's definition of the level. This prevents a K8s upgrade from silently tightening your `restricted` definition and breaking existing pods. Set it; don't leave it at `latest`.

---

## Exemptions

Some pods are exempt from PSS by default (or can be configured as exempt):

```yaml
# kube-apiserver configuration (for self-managed clusters)
--admission-plugins=PodSecurity
--pod-security-admission-config-file=/etc/k8s/psa-config.yaml
```

```yaml
# /etc/k8s/psa-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: restricted           # cluster default
      enforce-version: latest
      audit: restricted
      warn: restricted
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system                 # system namespace is exempt
      - monitoring                  # monitoring namespace is exempt (node-exporter needs privileged)
```

Managed clusters (EKS, GKE, AKS) manage this config for you — but they often set the default level to `privileged` for `kube-system` and leave user namespaces at `privileged` by default. You have to explicitly label your namespaces.

---

## The PSP → PSS migration path

If you're on K8s 1.24 and need to migrate before 1.25 removes PSP:

1. **Inventory your PSPs**: `kubectl get psp` — list all, note which are actually being enforced vs permissive catch-alls.

2. **Map PSP rules to PSS levels**: for each PSP, determine which PSS level it maps to. A PSP that disallows `privileged` and `hostNetwork` is roughly baseline. A PSP that also requires `runAsNonRoot` is roughly restricted.

3. **Label namespaces in audit/warn mode first**: set `audit: restricted` and `warn: restricted` on all non-system namespaces. Let the cluster run for a sprint; collect violations.

4. **Fix violations**: common fixes:
   - `runAsUser` not set → add `securityContext.runAsNonRoot: true` + `runAsUser: 1000`
   - `capabilities` not dropped → add `capabilities: { drop: [ALL] }`
   - `seccompProfile` missing → add `seccompProfile: { type: RuntimeDefault }`

5. **Switch to enforce**: once audit logs are clean, flip `enforce: restricted` namespace by namespace, starting with the least critical.

6. **Disable PSP**: after migration, remove the `PodSecurityPolicy` admission plugin from the apiserver config.

---

## Kyverno as the custom-rules layer

PSS can't express rules like:
- "Images must come from `my-registry.example.com`"
- "All pods must have `app.kubernetes.io/team` label"
- "No `latest` tags"
- "Resources must have CPU/memory limits"

This is where Kyverno comes in — and where your production Kyverno experience is directly applicable.

```yaml
# Kyverno ClusterPolicy: require non-root securityContext (supplements PSS)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce   # or Audit
  background: true
  rules:
  - name: check-non-root
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Containers must not run as root."
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
```

```yaml
# Kyverno ClusterPolicy: require image from approved registries
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allowed-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-registry
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Images must come from approved registries."
      pattern:
        spec:
          containers:
          - image: "my-registry.example.com/* | ecr.amazonaws.com/*"
```

The production pattern: **PSS handles the standard security baseline** (namespace labels, no extra moving parts), and **Kyverno handles organization-specific rules** (registries, labels, resource limits, naming conventions). This replaces everything PSP was trying to do, and does it more clearly.

In Wiz terms: Kyverno policies are your preventive control layer in the cluster; Wiz is your detective control layer (scanning what actually ran, finding drift). They're complementary.

---

## A worked cluster-wide rollout

```bash
# Step 1: label all user namespaces with warn/audit at restricted
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}' \
  | tr ' ' '\n' | grep -v '^kube-'); do
  kubectl label namespace "$ns" \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/warn=restricted \
    --overwrite
done

# Step 2: collect violations over a sprint
# kubectl get events -A --field-selector reason=FailedCreate | grep "violates PodSecurity"

# Step 3: after fixing violations, enforce baseline everywhere first
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=baseline \
  --overwrite

# Step 4: tighten to restricted after validating
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.28 \
  --overwrite
```

---

## The interview answer in 60 seconds

> "PSP was deprecated in 1.21 and removed in 1.25 because it was too complex — the RBAC coupling confused teams into setting up catch-all PSPs that allowed everything. PSS replaced it with a simpler model built into the apiserver: three levels (privileged, baseline, restricted), three modes (enforce, audit, warn), applied via namespace labels. No extra components.
>
> To enforce restricted cluster-wide, I'd start by labeling all user namespaces with `audit: restricted` and `warn: restricted` — pods still run, but I collect the violations in audit logs and dev terminals. I'd fix them over a sprint: add `runAsNonRoot`, drop all capabilities, add a seccomp profile. Then I'd flip `enforce: restricted` namespace by namespace, starting with least critical. I'd also pin `-version: v1.28` to prevent a K8s upgrade from tightening the definition silently.
>
> For custom rules — required registries, label conventions, resource limits — I'd layer Kyverno ClusterPolicies on top. PSS handles the 80% standard baseline; Kyverno handles the organization-specific 20%. Together they replace everything PSP was trying to do."

---

## Self-test drills

### 1. PSS replaced PodSecurityPolicy. Walk me through the new model and how you'd enforce restricted.

**Reference answer**: three levels, three modes, namespace labels, built into apiserver. Migration path: warn/audit first, fix violations, enforce last, pin version. Kyverno for custom rules. See full rollout above.

### 2. What does the `restricted` PSS level actually require?

**Reference answer**: non-root (`runAsNonRoot: true`), `allowPrivilegeEscalation: false`, `capabilities: { drop: [ALL] }`, `seccompProfile: { type: RuntimeDefault }`, only safe volume types. Everything in baseline (no `hostNetwork`, `hostPID`, `hostIPC`, no privileged containers) plus these additions.

### 3. What's the difference between `enforce`, `audit`, and `warn` modes?

**Reference answer**: `enforce` rejects the pod — it never starts. `audit` lets the pod run but writes a violation to the audit log. `warn` lets the pod run but shows a warning in `kubectl` output. You use all three together during migration: audit and warn to see what would be blocked, enforce once you've fixed the violations.

### 4. Why use Kyverno alongside PSS rather than just PSS?

**Reference answer**: PSS only has three fixed levels — it can't express custom rules. You can't use it to require images from a specific registry, mandate team labels, ban `latest` tags, or enforce resource limits. Kyverno (or OPA Gatekeeper) fills that gap: you define ClusterPolicies that validate or mutate pods against organization-specific rules. PSS is the built-in baseline; Kyverno is the extensible policy layer.

---

## Further reading

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [PSP to PSS migration guide](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/)
- [Kyverno docs](https://kyverno.io/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)

---

## The 4 dimensions (senior framing)

- **Tech**: three levels (privileged, baseline, restricted) + three modes (enforce, audit, warn) + namespace labels = the entire API. No webhooks, no CRDs. `restricted` requires: non-root, drop ALL caps, seccomp RuntimeDefault, no hostPath volumes. Kyverno/OPA Gatekeeper for custom rules. Pin `-version` label to prevent surprise tightening on upgrades.
- **People**: the migration from PSP to PSS is primarily a people problem — finding the teams that own the pods with violations and getting them to fix the securityContext. Run warn mode in prod first so developers see the warnings in their own terminals; they're more motivated to fix it when it's their `kubectl apply` that shows the warning. Don't flip enforce until they've had a sprint to address it.
- **CI/CD**: your Kyverno ClusterPolicies run in `Audit` mode first (set `validationFailureAction: Audit`), generating Policy Reports you can review in CI. Once clean, flip to `Enforce`. Gate PRs on Kyverno admission — a Kyverno-Enforce cluster with a pre-apply dry-run in CI catches violations before they reach prod. Pair with PSS `warn` in your staging namespace so engineers see both signals.
- **Operations**: use `kubectl get events --field-selector reason=FailedCreate` to see PSS enforcement events in real time. Collect PSS audit violations from the apiserver audit log — aggregate them in your SIEM or Loki. Alert on any new `kube-system` pod that's not exempted (a supply-chain attack landing a pod in kube-system is a high-signal event). Quarterly: review Kyverno Policy Reports for residual audit violations; tighten enforce where audit is clean.
