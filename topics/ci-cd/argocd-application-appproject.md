# ArgoCD Application + AppProject — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through what an ArgoCD Application + AppProject actually do, and how they map to a multi-team setup"** — naming the CRD fields, the multi-tenant boundary model, the auto-sync gotcha, and the platform-team-owns-AppProject pattern.

> Start with [argocd-application-appproject-simple.md](./argocd-application-appproject-simple.md) if you haven't.

---

## The senior framing — GitOps as a contract

Mid-level: "ArgoCD syncs Git to Kubernetes."
Senior: "ArgoCD is the enforcement layer that makes Git the single source of truth. The Application defines what should exist. The AppProject defines who is allowed to say so and where. RBAC and AppProject together mean no team can deploy to another team's namespace, and no manual cluster change survives a sync cycle."

The interview signal is explaining the governance model, not just the sync mechanics.

---

## Application CRD — anatomy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd              # Applications live in the argocd namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # cascade-delete: deleting the Application deletes managed resources
spec:
  project: team-alpha            # AppProject membership (required)

  source:
    repoURL: https://github.com/myorg/infra-configs
    targetRevision: HEAD         # branch, tag, or commit SHA
    path: apps/my-app/production # directory in the repo

    # For Helm charts:
    # chart: my-app
    # helm:
    #   valueFiles:
    #     - values-production.yaml
    #   parameters:
    #     - name: image.tag
    #       value: "v1.2.3"

    # For Kustomize:
    # kustomize:
    #   images:
    #     - my-app=registry.example.com/myapp:v1.2.3

  destination:
    server: https://kubernetes.default.svc   # the in-cluster API server
    # OR:
    # name: production-cluster               # ArgoCD cluster name (registered via argocd cluster add)
    namespace: team-alpha-production

  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true     # revert manual changes within 3 minutes
    syncOptions:
      - CreateNamespace=true   # create the destination namespace if it doesn't exist
      - ServerSideApply=true   # use SSA instead of client-side apply (see argocd-ignoredifferences-ssa.md)
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:          # don't treat these fields as drift (see argocd-ignoredifferences-ssa.md)
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas      # HPA manages replicas; ArgoCD should ignore

  revisionHistoryLimit: 10   # how many sync histories to keep in ArgoCD
```

### destination: name vs server

```yaml
destination:
  server: https://kubernetes.default.svc  # in-cluster, by API server URL
```

vs

```yaml
destination:
  name: production-cluster   # by ArgoCD cluster name (registered via `argocd cluster add`)
```

`name:` is the modern preference. It's stable if the cluster's API server URL changes (e.g., EKS endpoint rotation). `server:` requires updating every Application if the URL changes. Use `name:` for external clusters, `server: https://kubernetes.default.svc` for the in-cluster case (it never changes).

### The finalizer — cascade delete

The `resources-finalizer.argocd.argoproj.io` finalizer means: when you delete the ArgoCD Application object, ArgoCD also deletes all the Kubernetes resources it was managing. Without it, deleting the Application leaves orphaned resources in the cluster. Add it for production Applications. Be careful: deleting the Application then deletes the workload.

---

## AppProject CRD — anatomy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: "Team Alpha — checkout service"

  sourceRepos:
    - https://github.com/myorg/team-alpha-*    # wildcard: any team-alpha repo
    - https://github.com/myorg/shared-platform  # shared charts (platform team repo)
    - "*"                                         # allow all (only for the default project)

  destinations:
    - server: https://kubernetes.default.svc
      namespace: "team-alpha-*"    # any team-alpha-prefixed namespace
    - server: https://prod.cluster.example.com
      namespace: "team-alpha-*"    # same constraint on the external cluster

  # Cluster-level resources team-alpha can manage
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace   # can create namespaces (for their own namespace provisioning)
    - group: rbac.authorization.k8s.io
      kind: ClusterRole   # if they need ClusterRoles for their CRDs

  # Namespace-level resources they CANNOT manage (blacklist wins over whitelist)
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange
    - group: networking.k8s.io
      kind: NetworkPolicy   # platform team controls NetworkPolicies

  # RBAC within this AppProject
  roles:
    - name: deployers
      description: "Can sync applications in this project"
      policies:
        - "p, proj:team-alpha:deployers, applications, sync, team-alpha/*, allow"
      groups:
        - github-team:team-alpha-devs   # SSO group → role binding

  orphanedResources:
    warn: true   # alert if resources in the destination namespace aren't tracked by any Application
```

### sourceRepos — the supply chain control

`sourceRepos` is the key supply-chain boundary. If team-alpha can only pull from their own GitHub repos, they can't introduce manifests from an external source (a compromised third-party repo). The platform team controls which repos are trusted.

### namespaceResourceBlacklist + clusterResourceWhitelist

The default is: namespace-level resources are allowed (teams can deploy Deployments, Services, etc.) but cluster-level resources are not (no ClusterRoles, no Namespaces, no ClusterPolicies). These defaults are sensible for multi-tenant setups.

Customize with:
- `clusterResourceWhitelist`: explicitly allow specific cluster-level kinds (e.g., Namespace if teams provision their own namespaces via GitOps)
- `namespaceResourceBlacklist`: block specific namespace-level kinds (e.g., ResourceQuota — platform team owns quotas)

---

## The multi-team platform pattern

```
Git layout:
  infra-configs/
    projects/
      team-alpha.yaml    ← AppProject (owned by platform team)
      team-beta.yaml
    apps/
      team-alpha/
        my-app/
          production/    ← Application source path (owned by team-alpha)
          staging/
      team-beta/
        their-app/
          ...
```

ArgoCD itself manages its own configuration via GitOps — an Application called `argocd-projects` syncs the `projects/` directory, and `argocd-apps` syncs the `apps/` directory (or use app-of-apps — see `argocd-appofapps-applicationset.md`).

**The key governance point**: the `projects/` directory has a `CODEOWNERS` rule pointing to the platform team. App teams can open PRs to add Applications in their own directory but cannot modify their AppProject boundaries without a platform team review. Git PR review is the access control mechanism.

---

## Auto-sync — the critical decision

`syncPolicy.automated` with `prune: true` and `selfHeal: true` is the full GitOps position.

| Setting | Effect | When to use |
|---|---|---|
| No `automated` | ArgoCD shows drift but doesn't fix it | Teams that want human approval for every sync |
| `automated`, no options | Syncs new resources but doesn't prune or heal | Migration period |
| `automated.prune: true` | Deletes resources removed from Git | Required for full GitOps |
| `automated.selfHeal: true` | Reverts manual cluster changes | Required for full GitOps |
| Both | True GitOps: Git is law | Production standard |

**The hidden gotcha without auto-sync**: teams think they're doing GitOps ("we have ArgoCD!") but if `selfHeal` is off, someone does `kubectl edit` in production, drift accumulates, and the next deployment overwrites it inconsistently. Without `selfHeal: true`, the out-of-sync resource is never fixed unless someone manually hits Sync. Many teams run in this state for months.

**When NOT to auto-sync**: for Applications managing shared infrastructure where a bad sync could affect multiple teams (e.g., cert-manager, ingress controllers). These often have a human approval step in the sync. Use ArgoCD sync windows or require a manual sync on the Application.

---

## ArgoCD RBAC — who can sync what

ArgoCD has its own RBAC model (separate from Kubernetes RBAC):

```yaml
# argocd-rbac-cm ConfigMap
policy.csv: |
  # Admins can do anything
  g, role:admin, admin

  # SSO group → project-scoped role
  g, github-team:team-alpha-devs, role:team-alpha-deployer

  p, role:team-alpha-deployer, applications, get,    team-alpha/*, allow
  p, role:team-alpha-deployer, applications, sync,   team-alpha/*, allow
  p, role:team-alpha-deployer, applications, create, team-alpha/*, allow
  # Note: no 'applications, delete' → teams can't delete Applications in their project

  # Platform team can manage everything
  g, github-team:platform, role:admin
```

The AppProject `roles` field is a shorthand for project-scoped policies. The `argocd-rbac-cm` ConfigMap is for org-wide RBAC.

---

## The interview answer in 60 seconds

> "An ArgoCD Application is a sync rule: source repo + path + revision maps to a destination cluster + namespace. The Application is what does the work — it computes the diff between Git and the cluster and reconciles them.
>
> An AppProject is the boundary. It says: this Application is allowed to pull from these repos and deploy to these namespaces, and only manage these K8s resource kinds. Without AppProject, every team can deploy anywhere. With it, team-alpha can only touch `team-alpha-*` namespaces and can only pull from their approved repos.
>
> The multi-team pattern: platform team owns AppProject CRDs (the fences), app teams own Application CRDs (within the fence). Git `CODEOWNERS` enforces this — only the platform team can approve changes to the `projects/` directory.
>
> The auto-sync gotcha: without `selfHeal: true`, ArgoCD shows drift but doesn't fix it. Teams do `kubectl edit` in production, the change survives until the next manual sync, and then gets overwritten inconsistently. Full GitOps means `prune: true` and `selfHeal: true` — Git is law."

---

## Self-test drills

### 1. What's the difference between destination.server and destination.name?

**Reference answer:**
- `destination.server` is the Kubernetes API server URL (e.g., `https://prod.cluster.example.com`). Stable until the URL changes — which can happen (EKS endpoint rotation, cluster recreation).
- `destination.name` is ArgoCD's registered cluster name (set when you run `argocd cluster add`). Stable even if the API server URL changes, because the name is a label in ArgoCD's own records.
- Use `name:` for external clusters (multi-cluster setups). Use `server: https://kubernetes.default.svc` for in-cluster (it's always the same).

### 2. What happens when you delete an ArgoCD Application that has the cascade-delete finalizer?

**Reference answer:**
- ArgoCD deletes all Kubernetes resources it was managing in the destination namespace before removing the Application object.
- The finalizer is `resources-finalizer.argocd.argoproj.io`. With it: delete Application → ArgoCD prunes all managed resources. Without it: delete Application → managed resources remain as orphans.
- Practical implication: deleting an Application in production deletes the workload. Always double-check the finalizer is intentional. For "undeploy everything," this is the right behavior. For "I want to stop managing this via ArgoCD but keep the resources running," remove the finalizer first.

### 3. Why might a team say "we use ArgoCD but it keeps reverting our changes"?

**Reference answer:**
- They have `selfHeal: true` enabled and are making manual cluster changes (`kubectl edit`, `kubectl scale`).
- With `selfHeal: true`, ArgoCD periodically (every 3 min by default) reconciles the cluster against Git. Any cluster state that differs from Git is reverted.
- The fix is not to turn off `selfHeal` — it's to make changes through Git. If they need to scale a deployment temporarily, do it by editing the Git manifest and committing. If they need a longer-term override, use `ignoreDifferences` for that specific field (e.g., replica count managed by HPA).
- The underlying insight: they're treating ArgoCD as a deployment tool but still using the cluster as the source of truth. GitOps means Git is the source of truth. The "problem" is actually the correct behavior.

---

## Further reading

- [ArgoCD Application CRD spec](https://argo-cd.readthedocs.io/en/stable/operator-manual/application-crd/)
- [ArgoCD AppProject spec](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)
- [ArgoCD RBAC configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [ArgoCD sync options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)

---

## The 4 dimensions (senior framing)

- **Tech**: Application CRD (source/destination/syncPolicy/ignoreDifferences); AppProject CRD (sourceRepos, destinations, resource allow/blocklists, roles); `destination.name` over `destination.server`; cascade-delete finalizer; `selfHeal: true` for true GitOps.
- **People**: the multi-team pattern requires explicit ownership — platform team owns AppProject, app teams own Applications. This boundary lives in Git (CODEOWNERS). Onboarding a new team = PR to create their AppProject (platform review) + PR in their repo to create their Applications (their review). No manual cluster access needed.
- **CI/CD**: the Application itself is deployed via GitOps (the "app-of-apps" pattern, or ApplicationSet). Don't deploy Applications manually with `argocd app create`. Manage them in Git like everything else. Add a CI check that validates Application/AppProject YAML (schema + AppProject boundary consistency) before merge.
- **Operations**: monitor ArgoCD's `argocd_app_info` metric (sync status) and `argocd_app_sync_total` (sync frequency). Alert on `SyncFailed` status. The ArgoCD UI is useful but treat it as read-only for production — all actions should trace back to a Git commit. For incident response: emergency manual sync is fine; document it as a post-incident Git commit ("emergency sync at T+30: revert bad deploy").
