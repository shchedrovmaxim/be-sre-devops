# ArgoCD app-of-apps + ApplicationSet — the simple version (the manager vs the template)

> Read this first. Once the analogy clicks, the [deep-dive](./argocd-appofapps-applicationset.md) is easy.

This doc explains **one idea**:

> **App-of-apps is a parent Application that creates child Applications — manual fan-out. ApplicationSet is a generator that creates Applications automatically from a template + list of clusters/tenants/paths. Use app-of-apps for small setups. Use ApplicationSet when you're managing many things that follow the same pattern.**

---

## The manager vs the staffing agency

**App-of-apps** is like a **project manager** who personally assigns each team member to a task. For 5 tasks — fine. For 500 tasks — they'd have a full-time job just writing assignments.

**ApplicationSet** is like a **staffing agency**: you give them a template ("I need a developer with skill X") and a list of requirements ("one for each of our 20 offices"). They generate all the assignments automatically.

---

## App-of-apps — the manual approach

A parent Application whose source is a directory of Application YAMLs.

```
infra-configs/
  argocd-apps/
    my-app-production.yaml   ← Application YAML
    my-app-staging.yaml      ← Application YAML
    another-app.yaml         ← Application YAML
```

The parent:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-parent
spec:
  source:
    path: argocd-apps   # this directory contains Application YAMLs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD sees Application YAMLs in that directory and applies them to the `argocd` namespace — creating child Applications. Each child Application then syncs its own workload.

**When to use**: 5-20 Applications. Manual control is fine. You want to explicitly author each Application YAML.

---

## ApplicationSet — the generator approach

An ApplicationSet uses **generators** to produce Application objects automatically from a data source.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
spec:
  generators:
    - list:
        elements:
          - cluster: production
            url: https://prod.cluster.example.com
            namespace: my-app-production
          - cluster: staging
            url: https://staging.cluster.example.com
            namespace: my-app-staging

  template:
    metadata:
      name: "my-app-{{cluster}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/infra-configs
        path: apps/my-app/{{cluster}}
      destination:
        server: "{{url}}"
        namespace: "{{namespace}}"
```

This produces two Applications: `my-app-production` and `my-app-staging`. Each configured with the right cluster and namespace.

**When to use**: you're deploying the same app to many clusters, or many tenants, and you'd be writing identical Application YAMLs with only the cluster/namespace changing.

---

## The three most useful generators

| Generator | What it uses as input | Example use case |
|---|---|---|
| `list` | A hard-coded list of environments/clusters | Fixed set of 3 environments |
| `git` | Directories in a Git repo | Each subdirectory = one tenant's app |
| `cluster` | All clusters registered in ArgoCD | Deploy to every cluster automatically |
| `matrix` | Combination of two other generators | All apps × all clusters |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through app-of-apps vs ApplicationSet — when would you pick each?"**

**Reference answer (intuitive version):**

"App-of-apps is a parent Application pointing to a directory of child Application YAMLs. You author each Application manually. It's simple and explicit — great for a small platform with 10-15 apps. The downside: adding a new cluster means duplicating a bunch of Application files.

ApplicationSet is a generator. You write a template and a generator — 'list generator: here are my 3 clusters' or 'git generator: scan this directory, one Application per subdirectory.' The ApplicationSet automatically creates and manages the Applications. Adding a new cluster means adding one line to the list. Adding a new tenant's app means creating a new subdirectory.

I'd use app-of-apps to start — it's simpler to understand and debug. I'd migrate to ApplicationSet when I notice I'm copy-pasting Application YAMLs with only the cluster/namespace changing, or when I'm managing more than one cluster."

---

## Further reading

- [ArgoCD app-of-apps pattern docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD ApplicationSet docs](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [ApplicationSet generators reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/)

---

## Next: the deep-dive

When the manager-vs-staffing-agency analogy feels obvious, jump to [`argocd-appofapps-applicationset.md`](./argocd-appofapps-applicationset.md). The deep-dive covers:

- App-of-apps mechanics and gotchas
- ApplicationSet generators in depth (list, git, cluster, matrix, SCM provider)
- Template variables and how they expand
- The "GitOps for a fleet of clusters" pattern
- ApplicationSet sync policy and the deletion risk
- 3 self-test drills + 4-dimensions framing
