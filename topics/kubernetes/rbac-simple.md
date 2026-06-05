# RBAC — the simple version (the office keycard analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./rbac.md) becomes easy.

This doc only explains **one idea**:

> **RBAC answers "who can do what to which resource." It is four objects wired together: Role (what), RoleBinding (who gets it), ClusterRole (what, cluster-wide), ClusterRoleBinding (who gets it, cluster-wide).**

That's the whole model. Everything else is just precision on top.

---

## Your office has keycards

Imagine a big office building with multiple floors (namespaces). Rooms on each floor contain filing cabinets (K8s resources: Pods, Secrets, ConfigMaps…).

| Office world | K8s RBAC world |
|---|---|
| A keycard permission ("can open rooms on floor 3") | **Role** |
| Giving that keycard to Alice | **RoleBinding** |
| A master keycard ("can open rooms on ANY floor") | **ClusterRole** |
| Giving the master keycard to the security team | **ClusterRoleBinding** |
| Alice | A **ServiceAccount** (or user, or group) |
| "Can open" vs "can read files" vs "can shred files" | K8s **verbs**: get, list, create, update, delete, watch, patch |

Key insight: **a Role exists on one floor. A ClusterRole exists across all floors.** If you give Alice a ClusterRole via a *RoleBinding* (not ClusterRoleBinding), she only gets it on that floor. The ClusterRole is a template; the binding decides the scope.

---

## The four objects

```yaml
# 1. Role — defines what can be done, scoped to a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]          # "" = core API group (Pods, Secrets, ConfigMaps, Services)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 2. RoleBinding — grants the Role to someone, in the same namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-reads-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: alice
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# 3. ClusterRole — same as Role, but no namespace (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---
# 4. ClusterRoleBinding — grants a ClusterRole cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-reads-nodes
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## The one command you use for everything

```bash
# "Can the 'ci-bot' ServiceAccount in 'staging' list pods?"
kubectl auth can-i list pods \
  --as=system:serviceaccount:staging:ci-bot \
  --namespace=staging
# yes / no

# "Can it read Secrets?"
kubectl auth can-i get secrets \
  --as=system:serviceaccount:staging:ci-bot \
  --namespace=staging
```

Use this constantly. It is the RBAC audit tool. No guessing, no manually tracing bindings.

---

## The two smells to spot immediately

**1. Wildcard verbs**: `verbs: ["*"]` — grants every verb, including delete and escalate. Almost always wrong. Ask "does this pod actually need to delete nodes?"

**2. Wildcard resources**: `resources: ["*"]` — grants access to every resource in the apiGroup, including Secrets. The most common over-permission in production.

Both patterns typically come from "I'll tighten it later" shortcuts that live forever.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Role vs ClusterRole? | Role is namespaced. ClusterRole is cluster-wide (used for nodes, namespaces, PVs — non-namespaced resources). |
| RoleBinding vs ClusterRoleBinding? | RoleBinding restricts to one namespace. ClusterRoleBinding applies everywhere. |
| Can I grant a ClusterRole to just one namespace? | Yes — use a RoleBinding that references a ClusterRole. Scopes it to that namespace. |
| What's a ServiceAccount? | A K8s identity for a pod. Every pod gets one (default or named). |
| What is `system:masters`? | A built-in group that bypasses RBAC entirely. Cluster-admin equivalent. Don't grant it. |
| What's `kubectl auth can-i`? | Your best friend for auditing. Use it before every RBAC question. |

---

## Self-test (one question — the killer one)

Out loud:

> **"A pod's ServiceAccount can read Secrets across the cluster. Walk me through how you'd audit and lock that down."**

**Reference answer (intuitive version):**

"First I'd confirm what the ServiceAccount can actually do — `kubectl auth can-i get secrets --as=system:serviceaccount:<ns>:<name>` across a few namespaces, then list all ClusterRoleBindings and RoleBindings pointing at it. The smell is usually a wildcard verb or wildcard resource in a ClusterRole. To lock it down: if the pod only needs Secrets in its own namespace, I'd drop the ClusterRoleBinding and create a narrow Role in that namespace — `get` on `secrets`, limited to exactly the names it needs if K8s resource-name restrictions apply. If it needs nothing, I'd set `automountServiceAccountToken: false` on the pod or the ServiceAccount so the token doesn't even mount."

---

## Further reading

- [K8s RBAC docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [kubectl auth can-i reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#auth)

---

## Next: the deep-dive

When the keycard analogy feels obvious, jump to [`rbac.md`](./rbac.md). The deep-dive covers:

- The verb/resource/apiGroup model in detail
- `aggregationRule` for composing ClusterRoles
- The `system:` prefixed built-in roles (don't edit them)
- The full audit flow for over-permissioned ServiceAccounts
- Cloud IAM integration gotchas
- 4 self-test drills + 4-dimensions framing
