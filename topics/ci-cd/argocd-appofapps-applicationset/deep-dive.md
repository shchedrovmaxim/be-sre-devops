# ArgoCD app-of-apps + ApplicationSet — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through app-of-apps vs ApplicationSet — when would you pick each?"** — naming the generators (list, git, cluster, matrix), the template model, the "GitOps for a fleet" pattern, and the deletion-propagation gotcha.

> Start with [argocd-appofapps-applicationset-simple.md](./simple.md) if you haven't.

---

## The senior framing — fleet management at scale

Mid-level: "app-of-apps creates child Applications."
Senior: "app-of-apps is a manual pattern that doesn't scale past ~20 apps. ApplicationSet is the scalable approach — it treats Application creation as a function of a generator. The generator + template model means you can bootstrap a new cluster by adding one line, or onboard a new tenant by creating a directory."

The interview signal is describing the fleet-management pattern, not just the YAML.

---

## App-of-apps — mechanics and gotchas

### Structure

```
infra-configs/
  bootstrap/
    apps-parent.yaml            ← the one Application you apply manually
  argocd-apps/
    my-app-production.yaml      ← child Application (ArgoCD manages this)
    another-service-staging.yaml
    shared-infra.yaml
```

The parent Application (`apps-parent.yaml`) syncs the `argocd-apps/` directory into the `argocd` namespace. ArgoCD then creates each child Application, which in turn syncs its own workload.

### The bootstrap chicken-and-egg

The parent Application itself must be created manually (or via Helm/Terraform). After that, everything else is GitOps. This is the bootstrap problem — every GitOps setup has one.

```bash
# One-time bootstrap: create the parent
kubectl apply -f bootstrap/apps-parent.yaml
# Now ArgoCD manages everything else from Git
```

### The deletion propagation gotcha

If you delete a child Application YAML from the `argocd-apps/` directory, the parent Application's `prune: true` will delete the ArgoCD Application object. If the child Application has the cascade-delete finalizer, this also deletes all the workload resources. Deleting an Application YAML from Git deletes the workload.

This is correct GitOps behavior — but it surprises teams the first time. Communicate this clearly: "removing a service from Git removes it from the cluster."

### When to stop using app-of-apps

You've outgrown app-of-apps when:
- You have more than one cluster and you're copying Application YAMLs with only the cluster/namespace changed
- You're onboarding a new tenant by copy-pasting an existing tenant's Application files
- Adding a new environment means creating 10+ new YAML files

These are the signals to migrate to ApplicationSet.

---

## ApplicationSet — generators in depth

An ApplicationSet produces Applications via a template. The generator provides the variables the template expands.

### Generator 1: list

Fixed, explicit list of environments.

```yaml
generators:
  - list:
      elements:
        - env: production
          cluster: https://prod.cluster.example.com
          namespace: my-app-prod
          replicas: "5"
        - env: staging
          cluster: https://staging.cluster.example.com
          namespace: my-app-staging
          replicas: "2"
        - env: dev
          cluster: https://dev.cluster.example.com
          namespace: my-app-dev
          replicas: "1"
```

Best for: a fixed set of environments (prod/staging/dev) that rarely changes.

### Generator 2: git — directory

Scans a Git repo for directories matching a pattern. One Application per matching directory.

```yaml
generators:
  - git:
      repoURL: https://github.com/myorg/tenants
      revision: HEAD
      directories:
        - path: "tenants/*"
        # Generates an Application for each directory: tenants/team-alpha, tenants/team-beta, etc.
```

```
tenants/
  team-alpha/         ← one Application: my-app-team-alpha
    kustomization.yaml
    deployment.yaml
  team-beta/          ← one Application: my-app-team-beta
    kustomization.yaml
    deployment.yaml
```

Adding a new tenant: create a new directory in `tenants/`. The ApplicationSet detects it automatically and creates the Application. Zero platform-team involvement after the initial setup.

### Generator 2b: git — files

Instead of directories, use JSON/YAML config files in the repo:

```yaml
generators:
  - git:
      repoURL: https://github.com/myorg/tenants
      revision: HEAD
      files:
        - path: "tenants/**/config.json"
```

Each matching file is parsed, and its fields are available as template variables. Useful for per-tenant configuration beyond just "which directory."

### Generator 3: cluster

Generates one Application per cluster registered in ArgoCD.

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: production   # only clusters labeled env=production
```

Template variables available: `{{name}}`, `{{server}}`, `{{metadata.labels.env}}` (any cluster label).

Best for: platform-level applications that must run on every cluster (cert-manager, monitoring, Kyverno policies). Adding a new cluster to ArgoCD automatically triggers deployment of all platform services.

### Generator 4: matrix

Cross-product of two generators.

```yaml
generators:
  - matrix:
      generators:
        - git:
            repoURL: https://github.com/myorg/apps
            revision: HEAD
            directories:
              - path: "apps/*"
        - clusters:
            selector:
              matchLabels:
                type: production
```

This produces one Application per (app directory × production cluster). If you have 10 apps and 5 clusters, you get 50 Applications. Each app is deployed to every production cluster.

Best for: truly large fleet management. The power comes with complexity — a matrix of 10 × 5 means 50 Applications to debug.

### Generator 5: scmProvider

Scans GitHub/GitLab/Bitbucket for repos matching a filter.

```yaml
generators:
  - scmProvider:
      github:
        organization: myorg
        tokenRef:
          secretName: github-token
          key: token
      filters:
        - repositoryMatch: "^my-app-.*"   # repos starting with my-app-
```

Best for: "deploy every repo in our GitHub org that matches this pattern." Useful for self-service platforms where teams create repos and the platform deploys them automatically.

---

## The full ApplicationSet example — git directory generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
  namespace: argocd
spec:
  syncPolicy:
    preserveResourcesOnDeletion: false  # if the ApplicationSet is deleted, delete Applications and resources

  generators:
    - git:
        repoURL: https://github.com/myorg/tenants
        revision: HEAD
        directories:
          - path: "tenants/*"

  template:
    metadata:
      name: "{{path.basename}}"          # directory name (e.g., "team-alpha")
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: "{{path.basename}}"      # AppProject with same name as tenant
      source:
        repoURL: https://github.com/myorg/tenants
        targetRevision: HEAD
        path: "{{path}}"               # full path (e.g., "tenants/team-alpha")
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"  # namespace = directory name
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Template variables from the git directory generator:
- `{{path}}` — full path (`tenants/team-alpha`)
- `{{path.basename}}` — last segment (`team-alpha`)
- `{{path[0]}}`, `{{path[1]}}` — path segments by index

---

## ApplicationSet deletion behavior — the dangerous gotcha

If you delete the ApplicationSet object, by default ArgoCD also deletes all the Applications it created (and with cascade-delete finalizer, the workloads too).

Control this with:

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true   # Applications survive ApplicationSet deletion
```

With `preserveResourcesOnDeletion: true`: deleting the ApplicationSet leaves orphaned Applications. You manage them manually. Useful for "migrate away from ApplicationSet" transitions.

With `false` (default): deleting the ApplicationSet deletes everything. A production incident if done accidentally.

Protect against accidental deletion: use Kyverno to block deletion of the ApplicationSet without a label:

```yaml
# Kyverno policy: require 'safe-to-delete: "true"' label to delete ApplicationSets
```

---

## The "GitOps for a fleet of clusters" pattern

The full fleet management setup:

```
platform-team-repo/
  bootstrap/
    applicationset-platform.yaml    ← deploys platform services to all clusters
    applicationset-tenants.yaml     ← deploys tenant apps from tenant repos
  platform-apps/
    cert-manager/
    kyverno/
    monitoring/
  cluster-configs/
    production-us-east-1/
      config.json   ← {"cluster": "...", "region": "us-east-1", "env": "production"}
    production-eu-west-1/
      config.json
    staging/
      config.json
```

The `applicationset-platform.yaml` uses the git-files generator to read `cluster-configs/*/config.json`. It creates one Application per cluster config file for each platform service.

Adding a new cluster:
1. Register the cluster in ArgoCD: `argocd cluster add <context>`
2. Create a new directory in `cluster-configs/` with `config.json`.
3. Commit and push.
4. The ApplicationSet detects the new file and creates Applications for cert-manager, Kyverno, monitoring on the new cluster.

No manual ArgoCD Application authoring. Pure GitOps.

---

## The interview answer in 60 seconds

> "App-of-apps is a parent Application whose source is a directory of child Application YAMLs. You author each child Application manually. Simple and explicit — great for getting started or for a small platform with 10-15 apps. The pattern breaks when you're managing many clusters or tenants and copying Application YAMLs with only cluster/namespace changing.
>
> ApplicationSet solves this with generators. You write one template and choose a generator: list (fixed environments), git directory (one-app-per-directory — good for tenant self-service), cluster (deploy to every registered cluster), or matrix (cross-product of apps × clusters). The ApplicationSet manages the Application lifecycle — creates, updates, and deletes them as the generator output changes.
>
> The fleet pattern: git-files generator reads per-cluster config files. Adding a new cluster = create a config file. The ApplicationSet automatically deploys all platform services to it. That's the 'GitOps for a fleet' model — no manual ArgoCD commands, just Git.
>
> The gotcha: deleting an ApplicationSet deletes all its Applications (and with cascade-delete finalizer, the workloads). Protect with `preserveResourcesOnDeletion: true` on any ApplicationSet that manages production workloads."

---

## Self-test drills

### 1. What's the difference between the git directory generator and the git files generator?

**Reference answer:**
- **git directory**: generates one Application per directory matching the path glob. Template variables include the path and path.basename.
- **git files**: generates one Application per JSON/YAML file matching the path glob. Template variables include the file's parsed fields.
- Use directory when the directory structure itself encodes the tenant/env (each tenant has a directory). Use files when you need per-Application metadata beyond just the path (e.g., a config.json with region, cluster URL, replica count).
- They can be combined in a matrix generator: e.g., git-files (cluster configs) × git-directories (app directories) → one Application per (cluster × app).

### 2. What happens when you add a new directory to the git repo scanned by a git generator?

**Reference answer:**
- The ApplicationSet controller polls the Git repo (by default every 3 minutes, same as ArgoCD's sync interval).
- When it detects the new directory, it generates a new Application using the template with the new directory's path variables.
- ArgoCD creates the Application and immediately starts syncing it.
- No manual action required.
- Deletion is symmetric: if you remove the directory from Git, the ApplicationSet controller deletes the Application it generated (and with cascade-delete finalizer, the workload).

### 3. Why is the matrix generator powerful but also dangerous?

**Reference answer:**
- Power: cross-product generation. 10 apps × 5 clusters = 50 Applications managed by one ApplicationSet. Adding one cluster means instantly deploying all 10 apps to it.
- Danger: the same cross-product. If you add a cluster accidentally, all 10 apps deploy to it instantly. If the template has an error, all 50 Applications have the same error. A git generator path change affects all generated Applications simultaneously.
- Mitigation: gate matrix generators with careful cluster selectors (label-based). Separate platform ApplicationSets (system components) from tenant ApplicationSets (workloads). Use `preserveResourcesOnDeletion: true` on matrix-based ApplicationSets. Test changes in a dry-run mode before applying.

---

## Further reading

- [ArgoCD ApplicationSet docs](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [ApplicationSet generators reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/)
- [App-of-apps pattern docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

---

## The 4 dimensions (senior framing)

- **Tech**: app-of-apps = parent Application + child Application YAMLs in Git. ApplicationSet = generators (list/git/cluster/matrix/scmProvider) + template. Template variables from generator. Fleet pattern: git-files per cluster config + cluster generator. `preserveResourcesOnDeletion: true` for production ApplicationSets.
- **People**: the app-of-apps → ApplicationSet migration is a platform-team initiative, not an app-team one. App teams don't need to understand the generator model — they just create a directory (git generator) or a config file and the Application appears. Expose this as a self-service workflow with a PR template: "create a file in `tenants/your-team/config.json` with these required fields." The ApplicationSet handles the rest.
- **CI/CD**: add validation in CI for AppProject + ApplicationSet consistency: "every directory in `tenants/` must have a matching AppProject." This prevents orphaned Applications (app deployed but no AppProject boundary set). `argocd-autopilot` is a tool that codifies app-of-apps + ApplicationSet setup — worth knowing the name even if you use a manual setup.
- **Operations**: monitor the ApplicationSet controller's own health (it's a separate deployment from the ArgoCD application controller). Alert on ApplicationSet status conditions. When a generator error occurs (Git auth failure, cluster unreachable), the ApplicationSet controller logs it — these errors don't surface as ArgoCD Application errors, they're in the controller's own logs. Know how to `kubectl logs -n argocd deployment/argocd-applicationset-controller` to debug.
