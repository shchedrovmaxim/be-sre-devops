# RBAC — the deep-dive

> **Goal**: by the end you can answer the killer question — **"A pod's ServiceAccount can read Secrets across the cluster — walk me through how I'd audit and lock that down."** — naming the four RBAC objects, the audit commands, the smell patterns, and the minimum-permission replacement.

> Start with the [simple version](./simple.md) if you haven't read it. The keycard analogy is the spine of this whole topic.

---

## The senior framing — RBAC is your blast-radius control

Mid-level engineers think RBAC is about "giving the right permissions." Senior engineers know RBAC is about **limiting blast radius when credentials are compromised**. A compromised pod token with `secrets: *` in a ClusterRoleBinding is an immediate cluster-wide credential harvest. A compromised pod token scoped to `get` on two specific ConfigMaps in one namespace is a minor incident.

Every over-permission is a latent incident waiting for a compromise event to reveal it.

---

## The four objects

### Role and ClusterRole — the "what"

A Role defines a set of permissions (rules). Rules are **additive** — K8s RBAC has no deny.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: payments
rules:
- apiGroups: [""]               # "" = core group: Pods, Secrets, Services, ConfigMaps, etc.
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["stripe-api-key", "db-password"]  # optional: limit to specific names
- apiGroups: ["apps"]           # apps group: Deployments, StatefulSets, DaemonSets
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

A **ClusterRole** is identical in structure but has no `namespace` field and governs:
- Non-namespaced resources: Nodes, PersistentVolumes, Namespaces, ClusterRoles
- Cross-namespace or cluster-wide access when bound with a ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-health-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
```

### RoleBinding and ClusterRoleBinding — the "who gets it"

A **RoleBinding** grants a Role (or ClusterRole) to subjects **within a single namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-secret-reader
  namespace: payments
subjects:
- kind: ServiceAccount
  name: payments-api
  namespace: payments
- kind: User                    # human or external identity
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:                        # immutable after creation — delete and recreate to change
  kind: Role                    # or ClusterRole — RoleBinding can reference either
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

A **ClusterRoleBinding** grants a ClusterRole to subjects **cluster-wide** — across all namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-node-reader
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-health-reader
  apiGroup: rbac.authorization.k8s.io
```

**The subtle power move**: you can bind a ClusterRole with a *RoleBinding* (not ClusterRoleBinding) to scope it to one namespace. ClusterRoles become **reusable permission templates** this way. The same `secret-reader` ClusterRole bound via RoleBinding in `payments` only gives secret access in `payments`.

---

## The verb / resource / apiGroup model

Every rule has three axes:

| Axis | Values | Notes |
|---|---|---|
| `apiGroups` | `""` (core), `"apps"`, `"batch"`, `"networking.k8s.io"`, `"rbac.authorization.k8s.io"`, etc. | Match the API group in `kubectl api-resources -o wide` |
| `resources` | `"pods"`, `"secrets"`, `"deployments"`, `"nodes"`, etc. | Plural form. Also supports subresources: `"pods/log"`, `"pods/exec"` |
| `verbs` | `"get"`, `"list"`, `"watch"`, `"create"`, `"update"`, `"patch"`, `"delete"`, `"deletecollection"`, `"escalate"`, `"bind"`, `"impersonate"` | `"*"` is the wildcard |

```bash
# Discover what API group a resource belongs to
kubectl api-resources --verbs=list --namespaced=true -o wide | grep secrets
# NAME      SHORTNAMES  APIVERSION  NAMESPACED  KIND    VERBS
# secrets               v1          true        Secret  [create delete get list patch update watch]
```

**The subresource gotcha**: `pods/exec` is a separate resource. Granting `pods: ["get"]` does NOT grant exec. Many least-privilege roles forget to enumerate subresources needed by their consumers.

---

## aggregationRule — composing ClusterRoles

Instead of one giant ClusterRole, you can build it from smaller ones using label selectors:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-engineer
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to: platform-engineer
rules: []   # populated automatically by the aggregation controller
---
# Each sub-role opts in with the label:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-engineer-nodes
  labels:
    rbac.example.com/aggregate-to: platform-engineer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

K8s ships several built-in aggregated ClusterRoles (`admin`, `edit`, `view`) that use this pattern. Custom controllers add permissions by creating a new labeled ClusterRole — the aggregate is updated automatically.

---

## The system: prefixed built-in roles — don't touch them

K8s ships many built-in ClusterRoles and ClusterRoleBindings prefixed with `system:`:

| Name | What it is |
|---|---|
| `cluster-admin` | Superuser. Full access everywhere. |
| `system:masters` | Group that maps to cluster-admin via ClusterRoleBinding. Usually used for kubeadm bootstrap. |
| `system:node` | What kubelets use to talk to the API server. |
| `system:kube-scheduler` | What the scheduler uses. |
| `admin` | Full access within a namespace. |
| `edit` | Read/write within a namespace, but can't modify RBAC objects. |
| `view` | Read-only within a namespace. |

**Golden rule**: never bind `system:masters` to a real person or a ServiceAccount. It bypasses all RBAC admission checks — there are no deny rules that apply to it. If you grant it via SSO group membership, every person in that SSO group is effectively root on your cluster.

**Never edit the `system:` ClusterRoles** — they get overwritten on every K8s upgrade.

---

## The audit flow for an over-permissioned ServiceAccount

This is the exact workflow to answer the killer Q.

### Step 1 — confirm the symptom

```bash
# "Can the 'orders-api' SA in 'default' get secrets anywhere?"
kubectl auth can-i get secrets \
  --as=system:serviceaccount:default:orders-api \
  --namespace=production
# yes  ← bad
```

### Step 2 — find every binding that grants it anything

```bash
# ClusterRoleBindings — lists subjects for every CRB
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(
      .subjects[]? |
      select(.kind == "ServiceAccount" and
             .name == "orders-api" and
             .namespace == "default")
    ) | .metadata.name + " -> " + .roleRef.name'

# RoleBindings — must repeat per namespace, or loop
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get rolebindings -n "$ns" -o json 2>/dev/null \
    | jq -r --arg ns "$ns" '.items[] | select(
        .subjects[]? |
        select(.kind == "ServiceAccount" and
               .name == "orders-api" and
               .namespace == "default")
      ) | $ns + "/" + .metadata.name + " -> " + .roleRef.name'
done
```

### Step 3 — inspect the role it's bound to

```bash
kubectl describe clusterrole <name>   # or role -n <ns>
# Look for:
#   verbs: ["*"]  — wildcard verbs (automatic flag)
#   resources: ["*"]  — wildcard resources (automatic flag)
#   resources: ["secrets"]  — secrets access (check if really needed)
```

### Step 4 — check what the pod actually uses

If logs are available, grep for API calls. Tools like `kubectl-who-can`, `audit2rbac`, and `rbac-tool` can mine audit logs to produce a minimum-permission Role.

```bash
# rbac-tool (OSS) — generates minimum Role from audit log
rbac-tool policy-rules -e orders-api

# audit2rbac — generates Role from kube-apiserver audit log
audit2rbac --filename audit.log --serviceaccount default:orders-api
```

### Step 5 — lock it down

```yaml
# Replace the wildcard ClusterRoleBinding with a narrow namespaced Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orders-api
  namespace: orders
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["db-password", "stripe-key"]   # only the exact secrets it needs
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orders-api
  namespace: orders
subjects:
- kind: ServiceAccount
  name: orders-api
  namespace: orders
roleRef:
  kind: Role
  name: orders-api
  apiGroup: rbac.authorization.k8s.io
```

Then delete the over-broad ClusterRoleBinding.

---

## Common smell patterns

| Smell | Why it's bad | Fix |
|---|---|---|
| `verbs: ["*"]` | Grants delete, patch, escalate — everything. One compromised token = destructive access. | Enumerate: `["get", "list", "watch"]` |
| `resources: ["*"]` | Includes Secrets, RBAC objects. Often copied from `cluster-admin` templates. | Enumerate exactly the resources needed. |
| `ClusterRoleBinding` for a namespaced workload | A pod that only needs ConfigMaps in `staging` should not have cluster-wide ClusterRoleBinding. | Convert to RoleBinding in that namespace. |
| `cluster-admin` bound to a CI/CD ServiceAccount | CI bot with cluster admin = anyone who can inject a pipeline step owns the cluster. | Grant only what CI needs: apply specific Deployments, not cluster-admin. |
| Granting K8s `cluster-admin` to everyone in an SSO group | "Everyone in engineering" effectively owns the cluster. | Grant `view` at cluster level; `edit` per namespace per team. |

---

## Cloud IAM integration gotchas

In EKS, K8s RBAC is layered on top of AWS IAM via the `aws-auth` ConfigMap (legacy) or Access Entries (EKS 1.29+):

```yaml
# aws-auth ConfigMap (legacy) — maps IAM roles to K8s groups
mapRoles:
- rolearn: arn:aws:iam::123:role/devs
  username: "{{SessionName}}"
  groups:
  - system:masters           # ← BAD: gives full cluster-admin to all devs
  - developers               # ← GOOD: map to a limited K8s group
```

The most common EKS RBAC mistake: mapping a developer IAM role to `system:masters` because it's the fastest way to get `kubectl` working. Every developer in that role is now cluster-admin.

Fix: map the IAM role to a custom group (`developers`), then create a ClusterRoleBinding from `developers` to the `edit` ClusterRole — or to a custom ClusterRole that limits what namespaces they can touch.

In GKE, the equivalent is Workload Identity → K8s ServiceAccount → RBAC. Same principle: tight RBAC on the K8s side regardless of how broad the GCP IAM role is.

---

## kubectl auth can-i — the audit command to memorize

```bash
# Basic: can I do X?
kubectl auth can-i list pods -n production

# Impersonate: can ServiceAccount X do Y?
kubectl auth can-i get secrets \
  --as=system:serviceaccount:payments:payments-api \
  -n payments

# List all permissions for a subject (requires kubectl 1.26+)
kubectl auth can-i --list \
  --as=system:serviceaccount:payments:payments-api \
  -n payments

# Who can do something (kubectl-who-can plugin)
kubectl who-can get secrets -n payments
```

`kubectl auth can-i --list` is the single most useful command for RBAC audits. It shows every verb/resource combination the subject has, in that namespace.

---

## The interview answer in 60 seconds

> "I'd start with `kubectl auth can-i get secrets --as=system:serviceaccount:<ns>:<sa>` to confirm the scope. Then I'd find every ClusterRoleBinding and RoleBinding pointing at that ServiceAccount — using `kubectl get clusterrolebindings -o json | jq` to grep subjects. The smell is usually a wildcard verb or wildcard resource in a ClusterRole, or a ClusterRoleBinding where a namespaced RoleBinding would suffice.
>
> To lock it down: identify what the pod actually needs — API audit logs and tools like `audit2rbac` are useful here — then create a narrow Role in the pod's own namespace with only the required verbs on the specific resources, ideally with `resourceNames` to limit to exact Secret names. Delete the over-broad ClusterRoleBinding. Add a Kyverno policy to block wildcard verbs and wildcard resources in new Roles going forward, so this doesn't come back.
>
> Senior point: the biggest RBAC blast-radius mistake isn't usually a clever attack — it's CI/CD ServiceAccounts mapped to `cluster-admin` because it was 'easiest to get the pipeline working.' That's the first thing I'd check in a new cluster."

---

## Self-test drills

### 1. A pod's ServiceAccount can read Secrets across the cluster. Walk me through auditing and locking it down.

**Reference answer**: confirm with `kubectl auth can-i`; find bindings via `kubectl get clusterrolebindings -o json | jq`; inspect the role for wildcard smells; replace ClusterRoleBinding with a namespaced RoleBinding and a narrow Role with `resourceNames`; add Kyverno policy to prevent recurrence. See the full audit flow above.

### 2. What's the difference between a RoleBinding that references a ClusterRole vs a ClusterRoleBinding?

**Reference answer**: both grant the ClusterRole's permissions, but RoleBinding scopes it to a single namespace. ClusterRoleBinding grants it across the entire cluster. Use ClusterRoles as reusable templates; use the binding type to control scope.

### 3. What does `kubectl auth can-i --list` show and when do you use it?

**Reference answer**: it shows every verb/resource combination the current user (or impersonated identity) has in the current namespace. Use it to audit what a ServiceAccount can actually do — much faster than tracing bindings manually. Combined with `--as=system:serviceaccount:<ns>:<name>`, it's the complete audit tool.

### 4. Why should you never grant `system:masters` to a real user or ServiceAccount?

**Reference answer**: `system:masters` is a special group that bypasses RBAC entirely — no deny rules apply, and it's equivalent to having unrestricted cluster-admin access. It's not a Role or ClusterRole; it's hardcoded in the apiserver authorizer. Even if you later add Kyverno or OPA policies, they won't stop `system:masters` subjects. It exists for bootstrap scenarios (kubeadm) and should never be granted to operational identities.

---

## Further reading

- [K8s RBAC docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [rbac-tool](https://github.com/alcideio/rbac-tool) — visualize and audit RBAC policies
- [audit2rbac](https://github.com/liggitt/audit2rbac) — generate minimum RBAC from audit logs
- [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can) — reverse lookup: who can do X?
- [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) — the modern replacement for aws-auth ConfigMap

---

## The 4 dimensions (senior framing)

- **Tech**: four objects (Role, RoleBinding, ClusterRole, ClusterRoleBinding); verb/resource/apiGroup model; `kubectl auth can-i` for auditing; `aggregationRule` for composable ClusterRoles; `resourceNames` for maximum restriction; subresource gotcha (`pods/exec` is separate).
- **People**: RBAC sprawl comes from "just give it cluster-admin for now" shortcuts. Establish a review process where new ServiceAccounts require peer review of their Role. Make least-privilege the default in your Helm chart scaffolding. Don't rely on individuals to remember to tighten — automate it.
- **CI/CD**: if you have Kyverno in production (which you do), add policies that block: `verbs: ["*"]`, `resources: ["*"]`, and ClusterRoleBindings for namespaced ServiceAccounts. Gate ClusterRoleBinding creation behind a higher approval bar. Run `rbac-tool` or `audit2rbac` as a periodic CI job to flag permission drift.
- **Operations**: quarterly RBAC audit: list all ClusterRoleBindings, flag any that grant `cluster-admin` or contain wildcard patterns, remove any that belong to deleted ServiceAccounts. Alert on new ClusterRoleBindings that reference `cluster-admin`. Keep an inventory of which ServiceAccounts own which Secrets access so a compromise has a defined blast-radius.
