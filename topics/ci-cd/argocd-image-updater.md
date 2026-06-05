# ArgoCD Image Updater — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through how Image Updater detects a new image and commits the bump back to Git"** — naming the polling model, update strategies, write-back modes, Git credential setup, and the trade-off vs Renovate.

> Start with [argocd-image-updater-simple.md](./argocd-image-updater-simple.md) if you haven't.

---

## The senior framing — closing the GitOps loop

Mid-level: "we use Image Updater to auto-update images."
Senior: "Image Updater closes the last gap in the GitOps loop: CI pushes an image; Image Updater detects it, commits to Git; ArgoCD deploys from Git. Every deployment is a Git commit. The full audit trail is preserved. The image-to-production latency is predictable and measurable."

The trade-off to articulate: Image Updater is K8s-aware but K8s-only. It has no concept of npm packages, Python dependencies, or Terraform modules. Renovate handles those but is less K8s-native. Senior framing: both tools, clearly bounded.

---

## The polling model

Image Updater runs as a Deployment in the `argocd` namespace. It polls at a configurable interval:

```yaml
# argocd-image-updater ConfigMap
data:
  registries.conf: |
    registries:
      - name: Docker Hub
        api_url: https://registry-1.docker.io
        prefix: docker.io
        ping: yes
      - name: ECR
        api_url: https://123456789.dkr.ecr.us-east-1.amazonaws.com
        prefix: 123456789.dkr.ecr.us-east-1.amazonaws.com
        aws_region: us-east-1   # uses IRSA or instance role for auth
```

Image Updater respects registry rate limits. For Docker Hub, it caches registry responses. For private registries (ECR, GCR, Artifact Registry), authentication uses the same credential mechanisms as ArgoCD itself.

For ECR: use IRSA on the Image Updater ServiceAccount. The Image Updater then calls ECR's `GetAuthorizationToken` API automatically.

---

## Per-Application annotation configuration

Image Updater is enabled per ArgoCD Application via annotations. Nothing is enabled globally — you opt in per Application.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    # Enable Image Updater and specify the image alias(es)
    argocd-image-updater.argoproj.io/image-list: |
      app=registry.example.com/myorg/my-app

    # Update strategy for the 'app' alias
    argocd-image-updater.argoproj.io/app.update-strategy: semver

    # Semver constraint (only update within 1.x)
    argocd-image-updater.argoproj.io/app.allow-tags: regexp:^v1\.[0-9]+\.[0-9]+$

    # Write-back mode
    argocd-image-updater.argoproj.io/write-back-method: git

    # Git write-back configuration
    argocd-image-updater.argoproj.io/git-branch: image-updates   # branch to write to
    argocd-image-updater.argoproj.io/write-back-target: kustomization
```

The `image-list` annotation maps aliases (short names like `app`) to full image references. Multiple images per Application are supported.

---

## Update strategies in detail

### semver

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: semver
argocd-image-updater.argoproj.io/app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
# No explicit constraint = latest matching semver
```

Image Updater fetches the tag list from the registry, filters by `allow-tags` regexp, parses as semver, and selects the highest version that matches the constraint.

For a constraint:
```yaml
argocd-image-updater.argoproj.io/app.helm.image-tag: "image.tag"
argocd-image-updater.argoproj.io/app.update-strategy: semver
# Implicit: pick highest semver tag. Add explicit constraint:
argocd-image-updater.argoproj.io/app.force-update-strategy: ">=1.0.0, <2.0.0"
```

### latest

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: latest
```

Updates to the most recently **pushed** tag (by registry timestamp, not by name). Use for dev/preview environments where you always want the newest build. Never use in production — "latest" is not reproducible.

### digest

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: digest
```

Tracks a specific mutable tag (e.g., `:main`) and updates when the digest behind that tag changes. Useful when you tag by branch name and push to it on every merge. The tag stays `:main`; the digest changes on each push. Image Updater detects the digest change and updates.

### name

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: name
```

Alphabetically latest tag matching the filter. Useful when tags encode dates (e.g., `20260605-abc1234`) or when strict semver isn't used. Less predictable than semver.

---

## Write-back modes — the critical design decision

### Annotation write-back (simpler but not true GitOps)

```yaml
argocd-image-updater.argoproj.io/write-back-method: argocd
```

Image Updater writes the new image tag into the ArgoCD Application's annotations:

```yaml
annotations:
  argocd-image-updater.argoproj.io/app.tag: v1.3.0
```

ArgoCD reads these annotations and overrides the image tag at sync time. **No Git commit.** The cluster state reflects the new tag, but Git doesn't.

Trade-off: fast, no Git credentials needed. But rolling back means removing the annotation, not reverting a commit. And the deployment history isn't in Git — it's in ArgoCD's own sync history. Loss of audit trail.

### Git write-back (true GitOps)

```yaml
argocd-image-updater.argoproj.io/write-back-method: git
argocd-image-updater.argoproj.io/git-branch: image-updates
argocd-image-updater.argoproj.io/write-back-target: kustomization
```

Image Updater commits to Git. The commit updates either:
- A `.argocd-source-<appname>.yaml` file (Image Updater's own parameter file)
- Or the Kustomization's `images` list
- Or a Helm values file

#### Kustomize write-back

Image Updater writes to the Kustomization's image override:

```yaml
# kustomization.yaml (before)
images:
  - name: my-app
    newTag: v1.2.0
```

After Image Updater commits:

```yaml
# kustomization.yaml (after)
images:
  - name: my-app
    newTag: v1.3.0   # committed by argocd-image-updater
```

#### Helm write-back

```yaml
argocd-image-updater.argoproj.io/write-back-target: helmvalues:image.tag
```

Image Updater writes to a Helm values file:

```yaml
# values-production.yaml (after)
image:
  tag: v1.3.0   # committed by argocd-image-updater
```

---

## Git credential setup for write-back

Image Updater needs write access to your Git repo. Configure via a Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
  namespace: argocd
stringData:
  username: argocd-image-updater
  password: github_token_here   # fine-grained PAT with repo-write scope
```

Reference it in the Image Updater ConfigMap:

```yaml
# argocd-image-updater ConfigMap
data:
  git.user: argocd-image-updater
  git.email: argocd-image-updater@example.com
  git.commit-message-template: |
    chore(image-updater): bump {{.AppName}} to {{.NewTag}}
```

The Git user/email appears in commit history. The commit message template is customizable — make it recognizable so you can filter the history.

### Branch strategy for write-back

Two approaches:

**Direct to default branch**: Image Updater commits directly to `main`. No PR, no review. Automated and fast. Risk: a bad tag bump goes directly to production GitOps.

**Write to a separate branch**: Image Updater commits to `image-updates` branch. A CI job creates a PR from `image-updates` to `main`. Team reviews and merges. Slower but reviewed. Can also configure auto-merge for minor/patch semver bumps.

Production recommendation: direct to main for patch bumps on trusted semver tags (your own builds). PR for major version bumps. Match the risk level to the review overhead.

---

## The commit-back loop — understanding the timing

```
T+0:   CI pushes my-app:v1.3.0 to registry
T+2m:  Image Updater polls, detects v1.3.0 (polling interval = 2 min)
T+2m:  Image Updater commits to Git: "bump my-app to v1.3.0"
T+5m:  ArgoCD polls Git, detects new commit (ArgoCD polling = 3 min)
T+5m:  ArgoCD syncs: pulls my-app:v1.3.0, rolling update begins
T+8m:  Rolling update complete
```

Total latency from image push to production: ~5-8 minutes. This is acceptable for most teams. If you need faster, use ArgoCD webhooks (GitHub webhook triggers immediate ArgoCD sync instead of polling). Image Updater itself can also be triggered by a webhook in newer versions.

---

## Image Updater vs Renovate — the trade-off

| Dimension | ArgoCD Image Updater | Renovate |
|---|---|---|
| **What it updates** | Container image tags in ArgoCD Applications | Container images + npm/pip/Go/Maven/Helm deps + GitHub Actions + Terraform |
| **Kubernetes awareness** | Deep — reads ArgoCD Applications, writes to kustomization/Helm values | Shallow — manages files but no K8s resource semantics |
| **Config surface** | ArgoCD Application annotations | `renovate.json` in each repo |
| **Registry support** | Docker Hub, ECR, GCR, Artifact Registry, Harbor | Same + more (including Docker Hub with private repos) |
| **PR/commit model** | Commits directly or to a branch | Always creates PRs (with configurable auto-merge) |
| **Update policy** | semver constraint, latest, digest, name | Semver, regex, npm ranges, etc. |
| **Grouping/scheduling** | Limited | Rich (group packages, schedule business-hours only, automerge minor) |
| **Multi-ecosystem** | No (containers only) | Yes (everything) |
| **Self-hosted** | Yes (K8s Deployment) | Yes (GitHub App or self-hosted) |

**The senior framing:**

Use Image Updater when:
- You're a K8s-centric shop and container image version control is the primary concern
- You want K8s-native tooling (ArgoCD annotations, kustomization write-back)
- Your GitOps setup is fully ArgoCD-based

Use Renovate when:
- You have a polyglot repo (npm + Go + Helm + GitHub Actions all need version bumps)
- You want rich PR management (grouping, scheduling, auto-merge policies)
- You want the same tool for containers and dependencies

Use both when:
- Image Updater handles container images in ArgoCD-managed apps
- Renovate handles everything else (npm, pip, Terraform modules, GitHub Actions)
- **Avoid double-bumps**: if Renovate also watches container images in the same files Image Updater writes to, you'll get conflicts. Configure Renovate to ignore the files Image Updater writes to.

---

## The interview answer in 60 seconds

> "Image Updater runs in the cluster and polls container registries every few minutes. You annotate your ArgoCD Application to opt in — specifying the image alias, registry URL, and update strategy (semver, digest, latest). When Image Updater finds a new tag matching the policy, it writes back to Git — committing the updated image tag to the kustomization or Helm values file. ArgoCD sees the Git commit and syncs the new image to the cluster.
>
> The write-back mode matters architecturally. Annotation write-back is faster but doesn't create a Git commit — the deployment isn't in Git history. Git write-back creates a commit, preserving the full audit trail. For production GitOps, Git write-back is the right choice.
>
> The trade-off vs Renovate: Image Updater is deep on K8s — it understands ArgoCD Applications and writes to kustomization files directly. But it only handles container images. Renovate handles containers plus npm, pip, Terraform, GitHub Actions — everything. In practice I'd use both: Image Updater for container images in ArgoCD-managed apps, Renovate for everything else. Carefully partition which files each tool manages to avoid commit conflicts."

---

## Self-test drills

### 1. What's the difference between digest and semver update strategies?

**Reference answer:**
- `semver`: compares the tag names parsed as semantic version numbers. Picks the highest tag within the constraint. Requires tags to follow semver format (v1.2.3). Reproducible: given the same tag list, always picks the same winner.
- `digest`: tracks the digest behind a specific mutable tag (e.g., `:main`). The tag name doesn't change; the content does. Image Updater detects when the digest changes and updates.
- Use semver for production (deterministic, versioned). Use digest for preview/dev environments where you want "latest build of this branch" semantics.
- Gotcha: `latest` strategy uses registry timestamp, not tag name — it picks the most recently pushed tag regardless of name. Non-reproducible and risky for production.

### 2. What's the risk of Image Updater committing directly to the default branch vs a PR branch?

**Reference answer:**
- **Direct to main**: automated and fast (no PR review overhead). Risk: a bad tag (failed tests in CI, broken image) goes directly to production GitOps. The protection is that CI should have already validated the image before pushing. If CI is thorough, direct is fine for patch/minor bumps.
- **PR branch**: requires a human (or auto-merge bot) to merge. Adds review opportunity. Risk: PR merges can pile up; auto-merge policies become complex to configure correctly.
- Production pattern: direct for patch semver bumps from trusted pipelines; PR for major version bumps or external base image updates (where a human review makes sense).

### 3. How do you avoid double-bump conflicts between Image Updater and Renovate?

**Reference answer:**
- Image Updater writes to the kustomization.yaml or Helm values file.
- Renovate also scans these files for image references and might create PRs updating the same line.
- Solution: configure Renovate to ignore the specific files Image Updater manages. In `renovate.json`:
  ```json
  { "ignorePaths": ["**/kustomization.yaml"] }
  ```
  Or: configure Renovate to manage base image dependencies (in Dockerfile) and Image Updater to manage the kustomization/Helm values (deployed tag). Clear boundary: Renovate = source; Image Updater = deployment.
- The deeper fix: establish clear ownership. Renovate owns Dockerfiles and dependency files. Image Updater owns deployment configuration. They write to different files.

---

## Further reading

- [ArgoCD Image Updater docs](https://argocd-image-updater.readthedocs.io/) — especially the write-back configuration and registry setup
- [ECR with Image Updater + IRSA](https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/#using-aws-ecr) — the production AWS setup
- [Renovate vs Dependabot vs Image Updater comparison](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/2.0-2.1/) — community comparison blog posts

---

## The 4 dimensions (senior framing)

- **Tech**: poll interval (default 2 min), update strategies (semver/digest/latest/name), write-back modes (git vs annotation), per-Application annotation config, Git credential Secret + ConfigMap. ECR uses IRSA. Commit message template for history clarity.
- **People**: Image Updater creates commits in your repo. Make the commit message distinctive (`chore(image-updater): ...`) and add it to the team's `.gitmessage` expectations. App teams need to understand that "a commit appeared in our repo" is normal — it's the automation working. Without upfront communication, these commits cause confusion ("who pushed this?").
- **CI/CD**: don't configure Image Updater to update staging automatically AND production automatically without a promotion gate. The pattern: Image Updater updates staging automatically (direct commit). To promote to production, the promotion workflow reads the staging digest and creates a PR or commits to the production kustomization. Image Updater updating both environments simultaneously means a bad build hits prod as fast as staging.
- **Operations**: monitor Image Updater's own health (its `argocd-image-updater` deployment). It logs every image check — high-cardinality logs on busy registries. Set log level to `warn` in production to reduce noise. Monitor for "registry rate limit" errors (Docker Hub has a 100-pull/6h limit on anonymous; use authenticated pulls). If Image Updater stops committing, check: registry auth, Git credentials, branch protection rules (branch protection on `image-updates` branch may block the bot's push).
