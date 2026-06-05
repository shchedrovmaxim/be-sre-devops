# ArgoCD ignoreDifferences + Server-Side Apply — the simple version (the shared whiteboard)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **Sometimes a controller (HPA, cert-manager, a webhook) legitimately modifies a field ArgoCD owns. ignoreDifferences tells ArgoCD: "don't treat this field as drift — it's expected to be managed by someone else."**

---

## The shared whiteboard problem

Imagine two people are allowed to write on the same whiteboard:
- **ArgoCD** owns the whiteboard. It writes and maintains everything.
- **The HPA controller** occasionally writes the current replica count.

Every time the HPA changes the replica count, ArgoCD says: "this doesn't match Git — let me fix it!" And it writes `replicas: 2` back. Then the HPA updates it to 4. ArgoCD reverts it to 2. Fight.

This is the **controller conflict** problem. It happens when a Kubernetes controller legitimately mutates a field that ArgoCD is also managing.

The fix is to tell ArgoCD: "don't touch the replica count field — let the HPA own it."

```yaml
# In the Application CRD
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas   # ArgoCD will not treat replica-count changes as drift
```

---

## When does this come up?

Three classic cases:

| Who mutates | What they mutate | ignoreDifferences target |
|---|---|---|
| HPA | `spec.replicas` on Deployment/StatefulSet | `/spec/replicas` |
| cert-manager | `spec.template.spec.volumes` (injected CA cert) | `/spec/template/spec/volumes` |
| Admission webhook (e.g., Istio sidecar injector) | Container list (adds `istio-proxy`) | `/spec/template/spec/containers` |

In all cases: the mutation is legitimate, expected, and managed by another controller. ArgoCD should observe it without reverting it.

---

## Server-Side Apply — the modern upgrade

The "ignoreDifferences" problem is less severe with Server-Side Apply (SSA). With SSA, Kubernetes tracks which **manager** owns which **field**. ArgoCD only owns the fields it applies. The HPA owns `spec.replicas`. No conflict — they each write their own fields without interfering.

```yaml
syncOptions:
  - ServerSideApply=true   # in the Application's syncPolicy.syncOptions
```

SSA doesn't eliminate the need for `ignoreDifferences` entirely — some edge cases still need it — but it reduces the frequency dramatically.

---

## Self-test (the killer question)

Out loud:

> **"I have an HPA modifying replica count on a Deployment ArgoCD is managing. How do I keep ArgoCD from constantly trying to revert?"**

**Reference answer (intuitive version):**

"Two options. The simpler one: add `ignoreDifferences` to the Application spec, pointing at `/spec/replicas` with a JSON pointer. ArgoCD will observe the value but not treat changes to it as drift that needs reverting.

The more modern approach: enable Server-Side Apply in the Application's sync options. With SSA, Kubernetes tracks field ownership per manager. ArgoCD owns the fields it submitted. The HPA owns `spec.replicas` because it's the one updating that field. No conflict — they're working in parallel without stepping on each other. SSA is the cleaner long-term solution; `ignoreDifferences` is the quick fix for specific fields."

---

## Further reading

- [ArgoCD ignoreDifferences docs](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/)
- [ArgoCD Server-Side Apply](https://argo-cd.readthedocs.io/en/stable/user-guide/server-side-apply/)

---

## Next: the deep-dive

When the shared-whiteboard analogy feels obvious, jump to [`argocd-ignoredifferences-ssa.md`](./deep-dive.md). The deep-dive covers:

- ignoreDifferences with jsonPointers vs JQ paths
- `respectIgnoreDifferences` (apply-time behavior)
- SSA's managedFields in depth
- The `application.kubernetes.io/app-name` manager conflict
- All common controller-fight cases (HPA, cert-manager, Istio, CSI drivers)
- 3 self-test drills + 4-dimensions framing
