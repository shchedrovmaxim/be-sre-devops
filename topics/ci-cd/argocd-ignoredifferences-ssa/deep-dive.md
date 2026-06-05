# ArgoCD ignoreDifferences + Server-Side Apply — the deep-dive

> **Goal**: by the end you can answer the killer question — **"I have an HPA modifying replica count on a Deployment ArgoCD is managing. How do I keep ArgoCD from constantly trying to revert?"** — naming ignoreDifferences with jsonPointers/JQ, SSA's managedFields ownership model, `respectIgnoreDifferences`, and the common controller-fight patterns.

> Start with [argocd-ignoredifferences-ssa-simple.md](./simple.md) if you haven't.

---

## The senior framing — field ownership is a design concern

Mid-level: "ArgoCD keeps reverting my HPA. I set ignoreDifferences."
Senior: "the root cause is that multiple controllers claim ownership of the same field. The right fix is SSA — which makes field ownership explicit and tracked by Kubernetes. ignoreDifferences is a workaround for cases SSA can't fully solve."

The interview signal is explaining managedFields and why SSA is architecturally better, not just which YAML to write.

---

## The controller-fight pattern — why it happens

ArgoCD syncs by computing a diff between:
- **Desired state**: the resource manifest in Git
- **Live state**: the resource in the cluster

If the diff is non-empty, ArgoCD (with `selfHeal: true`) applies the desired state, overwriting the live state.

The HPA fight:

1. Git has `spec.replicas: 2`.
2. ArgoCD applies this. Deployment has `replicas: 2`.
3. Load increases. HPA updates `replicas: 8`.
4. ArgoCD computes diff: Git says 2, cluster says 8. Drift detected.
5. ArgoCD applies `replicas: 2` (because `selfHeal: true`).
6. HPA immediately updates to 8 again. Loop.

In practice, ArgoCD's sync interval is 3 minutes. So the HPA changes are reverted every 3 minutes, causing erratic scaling behavior. Under load, this is a production incident.

---

## ignoreDifferences — the targeted fix

`ignoreDifferences` tells ArgoCD to exclude specific fields from the diff computation. It doesn't stop ArgoCD from applying the resource; it just ignores those fields when deciding whether there's drift.

```yaml
# In the Application spec
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      name: my-app             # optional: scope to a specific resource name
      namespace: my-namespace  # optional: scope to a namespace
      jsonPointers:
        - /spec/replicas       # JSON Pointer format (RFC 6901)

    - group: ""
      kind: Secret
      jsonPointers:
        - /data                # ignore Secret data changes (e.g., externally managed secrets)

    - group: apps
      kind: StatefulSet
      jqPathExpressions:
        - ".spec.volumeClaimTemplates[].spec.storageClassName"  # JQ expression for complex paths
```

### jsonPointers vs jqPathExpressions

**JSON Pointer** (RFC 6901): simple path notation. `/spec/replicas` = the `replicas` field under `spec`. Good for simple scalar paths.

**JQ path expression**: full JQ query syntax. Required for:
- Array element matching: `.spec.containers[] | select(.name == "app") | .env`
- Conditional paths
- Deeply nested or dynamic structures

For most cases, JSON Pointer is sufficient and clearer. Use JQ when you need to ignore a field in a specific array element.

### respectIgnoreDifferences — apply-time behavior

By default, `ignoreDifferences` only affects **diff display** — ArgoCD shows no drift for ignored fields. But when ArgoCD applies (syncs), it still includes the field value from Git in the applied manifest.

This means: ArgoCD ignores the drift for display, but the next sync still applies `replicas: 2` from Git, briefly overwriting whatever the HPA set.

The fix:

```yaml
syncOptions:
  - RespectIgnoreDifferences=true
```

With `RespectIgnoreDifferences=true`, ArgoCD omits ignored fields entirely from the applied manifest. The HPA's value survives the sync. This is what you almost always want when using `ignoreDifferences` for controller-managed fields.

---

## Server-Side Apply — the structural fix

Server-Side Apply (SSA) is a Kubernetes API mode that tracks **field ownership** in `managedFields` metadata. Every field in a resource has an owner — the manager that last applied it.

```yaml
metadata:
  managedFields:
    - manager: argocd-application-controller
      operation: Apply
      fields:
        f:spec:
          f:template: ...
          # f:replicas is NOT here — ArgoCD didn't set it
    - manager: horizontal-pod-autoscaler
      operation: Update
      fields:
        f:spec:
          f:replicas: {}   # HPA owns this field
```

With SSA:
- ArgoCD applies only the fields it knows about (from Git). It doesn't set `spec.replicas` if it's not in the Git manifest.
- The HPA sets `spec.replicas`. It owns that field.
- No conflict — each manager owns separate fields. No fight.

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

### The SSA conflict case — when two managers claim the same field

SSA prevents the fight by requiring exclusive field ownership. If ArgoCD and the HPA both try to set `spec.replicas`, SSA raises a conflict error.

The resolution: use `forceApply` (ArgoCD takes ownership, overrides the other manager) or remove the field from the Git manifest so ArgoCD never claims ownership of it.

For HPA: remove `spec.replicas` from your Deployment manifest in Git entirely. Let the HPA own it from the start. ArgoCD will apply the Deployment without `replicas`; the HPA sets it.

```yaml
# In Git: Deployment without spec.replicas
spec:
  # DO NOT set replicas: here when using HPA
  selector: ...
  template: ...
```

If `spec.replicas` is absent from the manifest, SSA never creates an ArgoCD ownership entry for it. Clean from the start.

---

## Common controller-fight patterns and solutions

### 1. HPA + Deployment replicas

**Problem**: HPA modifies `spec.replicas`. ArgoCD reverts on sync.

**Fix A** (ignoreDifferences): 
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
syncOptions:
  - RespectIgnoreDifferences=true
```

**Fix B** (SSA + remove from Git): enable `ServerSideApply=true`, remove `spec.replicas` from the Deployment manifest.

Fix B is cleaner architecturally. Fix A is quicker to implement if SSA rollout is complex.

### 2. cert-manager CA injection

**Problem**: cert-manager's cainjector mutates Webhooks and CRD `spec.conversion` fields to inject the CA bundle. ArgoCD reverts on sync.

**Fix**:
```yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    jsonPointers:
      - /webhooks/0/clientConfig/caBundle
  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    jsonPointers:
      - /webhooks/0/clientConfig/caBundle
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jqPathExpressions:
      - ".spec.conversion.webhook.clientConfig.caBundle"
```

This is the most common `ignoreDifferences` usage in practice.

### 3. Istio sidecar injection

**Problem**: Istio's mutating webhook injects the `istio-proxy` container. ArgoCD sees an extra container that isn't in Git.

**Fix**:
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jqPathExpressions:
      - ".spec.template.spec.containers[] | select(.name == \"istio-proxy\")"
      - ".spec.template.spec.initContainers[] | select(.name == \"istio-init\")"
      - ".spec.template.metadata.annotations[\"kubectl.kubernetes.io/last-applied-configuration\"]"
```

### 4. CSI driver annotations

**Problem**: CSI drivers annotate PVCs or Secrets with driver-specific metadata after provisioning.

**Fix**: ignore the specific annotation keys that CSI adds:
```yaml
ignoreDifferences:
  - group: ""
    kind: PersistentVolumeClaim
    jqPathExpressions:
      - ".metadata.annotations[\"volume.beta.kubernetes.io/storage-provisioner\"]"
```

### 5. ExternalSecret / SealedSecret controller modifying Secret data

**Problem**: ExternalSecrets Operator or Sealed Secrets writes the decrypted Secret data. ArgoCD sees the data as drift from Git (which has an empty or template Secret).

**Fix**:
```yaml
ignoreDifferences:
  - group: ""
    kind: Secret
    jsonPointers:
      - /data
```

Or: don't manage Secrets via ArgoCD at all — only manage the ExternalSecret CRD (which ArgoCD applies), and let the controller produce the Secret. The Secret itself is not in Git and not an ArgoCD-managed resource.

---

## The interview answer in 60 seconds

> "The HPA-Deployment fight happens because ArgoCD computes a diff between Git (replicas: 2) and the cluster (replicas: 8) and reverts on selfHeal. Two ways to fix it.
>
> Quick fix: `ignoreDifferences` with a JSON pointer to `/spec/replicas`, plus `RespectIgnoreDifferences=true` in sync options. ArgoCD then excludes that field from the diff and doesn't apply it during syncs.
>
> Better fix: enable Server-Side Apply (`ServerSideApply=true`), and remove `spec.replicas` from the Deployment manifest in Git. With SSA, Kubernetes tracks field ownership via `managedFields`. ArgoCD owns the fields it applies; the HPA owns `spec.replicas` because it's the one setting it. No conflict — they manage separate fields. SSA is architecturally cleaner because field ownership is explicit and enforced by the API server, not by an ArgoCD workaround.
>
> The same pattern covers cert-manager CA injection and Istio sidecar injection — those also mutate fields ArgoCD manages. ignoreDifferences with JQ path expressions handles them."

---

## Self-test drills

### 1. What's the difference between ignoreDifferences and RespectIgnoreDifferences?

**Reference answer:**
- `ignoreDifferences` tells ArgoCD to exclude specific fields from the drift display. But by default, ArgoCD still applies the Git value for those fields during a sync.
- `RespectIgnoreDifferences=true` (in syncOptions) makes ArgoCD also omit those fields during sync apply. The live value survives the sync.
- Without `RespectIgnoreDifferences`: `ignoreDifferences` is just cosmetic — the display looks clean but the sync still overwrites the HPA's replica count every 3 minutes.
- The combination is almost always required when `ignoreDifferences` is used for controller-managed fields.

### 2. How does SSA managedFields prevent the controller fight?

**Reference answer:**
- With SSA, every field in a resource has an owner tracked in `managedFields`.
- ArgoCD claims ownership of the fields it submits from Git. If `spec.replicas` is absent from the Git manifest, ArgoCD never claims ownership of it.
- The HPA (or whichever controller) claims ownership when it writes `spec.replicas`.
- SSA enforces that only the field's owner can update it without conflict. If another manager tries to update a field owned by someone else, the API server raises a 409 Conflict.
- Resolution: `forceApply` (override) or remove the field from your manifest. For HPA, remove `spec.replicas` from Git. For Istio-injected containers, don't manage the container list yourself.

### 3. When would you use a JQ path expression instead of a JSON Pointer in ignoreDifferences?

**Reference answer:**
- JSON Pointer: `/spec/replicas` — simple path to a scalar or object. Correct when the field is at a predictable path.
- JQ path expression: needed when you need to match inside arrays by element value, not by index. Example: ignore the container named `istio-proxy` in a container list, regardless of its position index. JSON Pointer would need `/spec/template/spec/containers/2` (index-based), which breaks if the injection index changes. JQ: `.spec.template.spec.containers[] | select(.name == "istio-proxy")` is stable regardless of position.
- Rule of thumb: JSON Pointer for scalar fields. JQ for "find this element in an array by value."

---

## Further reading

- [ArgoCD diffing customization docs](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/)
- [ArgoCD Server-Side Apply docs](https://argo-cd.readthedocs.io/en/stable/user-guide/server-side-apply/)
- [K8s Server-Side Apply overview](https://kubernetes.io/docs/reference/using-api/server-side-apply/) — the managedFields model
- [JSON Pointer spec (RFC 6901)](https://datatracker.ietf.org/doc/html/rfc6901) — the path format used in ignoreDifferences

---

## The 4 dimensions (senior framing)

- **Tech**: `ignoreDifferences` with JSON Pointer (scalars) or JQ path (array elements) + `RespectIgnoreDifferences=true`. SSA (`ServerSideApply=true`) + remove controller-owned fields from Git manifests. Common cases: HPA/replicas, cert-manager/caBundle, Istio sidecar, ExternalSecrets.
- **People**: the HPA fight is often misdiagnosed as "ArgoCD is broken." The first instinct is to turn off `selfHeal`. Wrong — selfHeal is the correct GitOps behavior. Explain the field ownership model to the team. Once they understand SSA + managedFields, they stop fighting the tooling and start designing manifests correctly.
- **CI/CD**: run `argocd app diff` in CI to catch ignoreDifferences gaps before they hit production. If a controller-managed field appears in the diff output, it means `RespectIgnoreDifferences` isn't set. Add a smoke test: after a sync, verify the HPA's replica count was preserved (not reset to the Git value).
- **Operations**: the symptom of the HPA fight: erratic pod counts in the scaling history. If you see `spec.replicas` ping-ponging in `kubectl describe deployment`, it's the ArgoCD/HPA conflict. Fix: add ignoreDifferences immediately to stop the fight, then plan the proper SSA migration. Monitor ArgoCD sync frequency — excessive syncs (more than once per 3 min) on a stable app indicate selfHeal is fighting a controller.
