# ArgoCD Application + AppProject — the simple version (the tenant office map)

> Read this first. Once the analogy clicks, the [deep-dive](./argocd-application-appproject.md) is easy.

This doc explains **one idea**:

> **An ArgoCD Application says "sync this Git path to this cluster namespace." An AppProject says "here's the boundary — which repos, which namespaces, which K8s resources are allowed for this tenant."**

---

## The office building analogy

Imagine a large office building (your Kubernetes cluster) with many tenants (teams).

- **The Application** is a **desk assignment**: "Team Alpha has desk 4B in room 203." It specifies where the team sits and what files are on their desk (source repo + path → destination namespace).
- **The AppProject** is the **lease agreement**: "Team Alpha is allowed in rooms 200-209 only, they may use the conference room but not the server room, and their files can only come from approved suppliers." It sets the boundary.

Without the lease (AppProject), every tenant has access to every room. That's the default `default` project — fine for development, never for multi-team production.

---

## The Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: team-alpha     # which AppProject this Application belongs to
  source:
    repoURL: https://github.com/myorg/infra-configs
    targetRevision: HEAD
    path: apps/my-app/production
  destination:
    server: https://kubernetes.default.svc   # target cluster
    namespace: team-alpha-production          # target namespace
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # revert manual changes to match Git
```

Three key pieces:
- **source**: where the manifests live (Git repo + path + revision)
- **destination**: where they go (cluster + namespace)
- **syncPolicy**: manual (you click Sync) or automated (ArgoCD syncs automatically)

The `selfHeal: true` gotcha: if you `kubectl edit` a resource that ArgoCD manages, ArgoCD will revert it on the next sync cycle. GitOps means Git is the source of truth. Manual edits don't stick.

---

## The AppProject CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: "Team Alpha's applications"

  # Which source repos are allowed
  sourceRepos:
    - https://github.com/myorg/team-alpha-*
    - https://github.com/myorg/shared-charts

  # Which destination clusters + namespaces are allowed
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "team-alpha-*"   # wildcard: any namespace starting with team-alpha-

  # Which Kubernetes resources they can manage
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace    # they can create Namespaces
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota  # but they cannot modify ResourceQuotas (platform team controls those)
```

The AppProject is the multi-tenant boundary:
- **sourceRepos**: team Alpha can only pull from their own repos (not another team's).
- **destinations**: team Alpha can only deploy to `team-alpha-*` namespaces.
- **resource allowlist/blocklist**: team Alpha cannot touch cluster-level or quota resources.

---

## The team pattern: platform owns AppProject, app teams own Applications

```
Platform team owns:
  AppProject/team-alpha  ← defines the boundary

Team Alpha owns:
  Application/my-app-production   ← points into the boundary
  Application/my-app-staging
```

Platform team controls the boundaries (which repos, which namespaces, which K8s kinds). App teams work within them. This is the senior pattern for a multi-team GitOps setup.

---

## The auto-sync gotcha

Without `syncPolicy.automated`, ArgoCD shows drift but doesn't fix it. An out-of-sync resource is just a yellow icon in the UI. Teams often miss this in production.

With `automated`: ArgoCD syncs every 3 minutes (default). Any drift is reverted. Any resource deleted from Git is pruned from the cluster.

The question is: do you trust auto-sync to not break things? Answer: use it with `selfHeal: true` in production only after your GitOps workflows are solid. Without it, drift accumulates silently.

---

## Self-test (the killer question)

Out loud:

> **"Walk me through what an ArgoCD Application + AppProject actually do, and how they map to a multi-team setup."**

**Reference answer (intuitive version):**

"An Application CRD is a sync rule: 'take this Git path and reconcile it into this cluster namespace.' It defines source, destination, and sync policy. An AppProject is the tenant boundary: which source repos are allowed, which destination namespaces and clusters, which Kubernetes resource types. Platform team owns the AppProjects — they define the fence. App teams own their Applications — they work inside the fence. If a team tries to deploy to a namespace outside their AppProject's `destinations`, ArgoCD denies it. If they try to pull from an unauthorized repo, same result. Auto-sync with selfHeal means ArgoCD continuously reconciles Git → cluster. Without it, drift just shows as yellow icons that nobody looks at."

---

## Further reading

- [ArgoCD Application CRD reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/application-crd/)
- [ArgoCD AppProject reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)
- [ArgoCD multi-tenancy docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/multitenancy/)

---

## Next: the deep-dive

When the office-building analogy feels obvious, jump to [`argocd-application-appproject.md`](./argocd-application-appproject.md). The deep-dive covers:

- Application CRD in full (source types, destination by name vs server, ignoreDifferences)
- AppProject resource allow/block lists in depth
- The platform-team-owns-AppProject pattern
- Auto-sync gotchas (prune, selfHeal, when not to auto-sync)
- RBAC within ArgoCD (who can sync what)
- 3 self-test drills + 4-dimensions framing
