# Renovate vs Dependabot — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Pick Renovate vs Dependabot for a polyglot monorepo. Walk me through the trade-offs."** — naming Dependabot's limits, Renovate's manager model, grouping/scheduling/auto-merge, the self-hosted option, and the Image Updater coordination problem.

> Start with [renovate-vs-dependabot-simple.md](./renovate-vs-dependabot-simple.md) if you haven't.

---

## The senior framing — dependency update automation is ops work

Mid-level: "we use Dependabot for security updates."
Senior: "we treat dependency updates as a recurring operational concern — not just security patches, but all dependency drift. Renovate's grouping and auto-merge policies reduce the PR noise from 50 PRs/week to 5, while keeping us up to date. The goal is a strategy, not a tool."

The interview signal is explaining the operational model, not just which YAML to write.

---

## Dependabot — scope and mechanics

Dependabot is GitHub-native — zero installation, works automatically when enabled via `.github/dependabot.yml`.

### What Dependabot does

- Opens PRs to bump outdated dependencies
- Labels PRs with severity (for security advisories)
- Supports security updates independently of version updates (GitHub Dependabot security updates run on top of vulnerability alerts)
- Merges automatically with `auto-merge` enabled on the PR (requires branch protection + CI passing)

### Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
      timezone: "Europe/Kyiv"
    reviewers:
      - platform-team
    labels:
      - "dependencies"
    ignore:
      - dependency-name: "some-legacy-dep"
        versions: ["*"]
    open-pull-requests-limit: 10

  - package-ecosystem: docker
    directory: "/"
    schedule:
      interval: weekly

  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly

  - package-ecosystem: gomod
    directory: "/cmd/api"
    schedule:
      interval: daily
```

### Dependabot limits

| Limitation | Impact |
|---|---|
| No Terraform module/provider support | Terraform drift not covered |
| No Kubernetes manifest support | kustomization image tags not covered |
| No Helm chart dependency support (beyond basic) | Helm repos not fully covered |
| No custom regex managers | Can't handle internal version files |
| Limited grouping | Each dependency gets its own PR by default |
| No cross-repo config inheritance | Each repo has its own config |
| No auto-merge policy by update type | Auto-merge is PR-level, not rule-based |

For a simple Node app + GitHub Actions: Dependabot is sufficient. For a polyglot monorepo with Terraform, Helm, and 50+ npm packages: these limits become friction.

---

## Renovate — the manager model

Renovate is organized around **managers** — each manager understands one or more package ecosystems and knows how to detect, evaluate, and update dependencies in that ecosystem.

```
Manager         → What it detects
npm             → package.json, package-lock.json, yarn.lock
pip             → requirements.txt, Pipfile, pyproject.toml
poetry          → pyproject.toml (poetry format)
gomod           → go.mod, go.sum
maven           → pom.xml
gradle          → build.gradle, build.gradle.kts
cargo           → Cargo.toml
dockerfile      → FROM instructions
docker-compose  → image: lines
helm-values     → image.tag values in Helm values files
kustomize       → images: in kustomization.yaml
argocd          → Application image annotations
terraform       → required_providers, source in module blocks
github-actions  → uses: in workflow files
regex           → any file matching a custom regex pattern
```

Renovate is installed as:
- **Mend Renovate** (hosted): a GitHub App; zero self-hosting. Works like Dependabot from a configuration perspective. Paid above a certain number of repos.
- **Self-hosted**: run Renovate as a container, pointing it at your GitHub/GitLab org. Fully under your control. Free.

---

## Configuration — key concepts

### The `extends` inheritance model

```json
{
  "extends": [
    "config:base",              // Renovate's recommended baseline
    "group:allNonMajor",        // group all non-major updates into one PR
    ":automergeMinor",          // auto-merge minor updates if CI passes
    ":pinVersions"              // pin dependencies to exact versions (opinionated)
  ]
}
```

Renovate has a rich preset library. `config:base` is the recommended starting point — it enables major ecosystems and sets sensible defaults. You build on top with specific overrides.

For monorepos: the root `renovate.json` applies to all packages. You can also have per-package config:

```json
{
  "packageRules": [
    {
      "matchPaths": ["services/billing/**"],
      "enabled": false   // don't update dependencies in the billing service (frozen for compliance)
    }
  ]
}
```

### Grouping

```json
{
  "packageRules": [
    {
      "groupName": "AWS SDK",
      "matchPackagePatterns": ["^@aws-sdk/", "^aws-"],
      "automerge": true
    },
    {
      "groupName": "TypeScript tooling",
      "matchPackagePrefixes": ["typescript", "@typescript-eslint/"],
      "schedule": ["on the first day of the month"]
    },
    {
      "groupName": "all non-major dependencies",
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true,
      "automergeType": "pr"
    }
  ]
}
```

Instead of 40 PRs for AWS SDK patch updates, one grouped PR. This is the biggest operational win over Dependabot for large dependency trees.

### Scheduling

```json
{
  "schedule": ["after 9am and before 5pm on every weekday"],
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "schedule": ["on the first monday of the month"]
    }
  ]
}
```

Renovate respects schedules globally and per rule. In practice: patch updates daily (auto-merged), minor updates weekly, major updates monthly with human review. This manages the PR noise calendar effectively.

### Auto-merge

```json
{
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["ci/build", "ci/test"]
    },
    {
      "matchUpdateTypes": ["minor"],
      "automerge": false   // needs human review
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "reviewers": ["platform-team"],
      "labels": ["major-dependency-update"]
    }
  ]
}
```

Renovate auto-merges patch PRs that pass CI. Minor and major still require review. This is the sustainable model: fully automated for safe updates, human eyes on risky ones.

### Custom managers — the killer feature

```json
{
  "customManagers": [
    {
      "customType": "regex",
      "fileMatch": ["\\.yaml$"],
      "matchStrings": [
        "# renovate: datasource=(?<datasource>[a-z-]+?) depName=(?<depName>[^\\s]+)\n.*?: (?<currentValue>[^\\s]+)"
      ],
      "datasourceTemplate": "{{datasource}}"
    }
  ]
}
```

This custom manager looks for a comment in YAML files:

```yaml
# renovate: datasource=github-releases depName=kubernetes/kubernetes
kubernetesVersion: "1.30.2"
```

Renovate sees this comment, queries the GitHub releases API for `kubernetes/kubernetes`, and opens a PR when a new release is available. You can version-manage anything — internal version strings, CloudFormation parameter files, arbitrary config files.

---

## The Image Updater coordination problem

If your setup includes:
- ArgoCD Image Updater (writes new image tags to kustomization.yaml)
- Renovate with the `kustomize` manager (also reads kustomization.yaml to update image tags)

You'll get conflicts: both tools update the same lines in `kustomization.yaml`. Renovate might open a PR proposing `v1.2.9` while Image Updater has already committed `v1.3.0`.

**The fix — partition ownership clearly:**

Option A: configure Renovate to ignore kustomization files that Image Updater manages:
```json
{
  "packageRules": [
    {
      "matchPaths": ["**/kustomization.yaml"],
      "enabled": false
    }
  ]
}
```
Image Updater owns deployed image tags. Renovate owns everything else.

Option B: use only Renovate for container images; disable Image Updater. Renovate creates PRs for new image tags, and you merge them. Slower (no auto-commit from Image Updater), but unified tooling.

**Senior framing**: the boundary is "what gets auto-committed to the deployment config vs what goes through PR review." Image Updater = auto-commit for K8s-deployed images. Renovate = PR for everything else (base images in Dockerfiles, npm packages, Terraform). Don't overlap.

---

## Renovate vs Dependabot — the complete trade-off matrix

| Dimension | Dependabot | Renovate |
|---|---|---|
| **Setup** | Zero (GitHub-native) | Install GitHub App or self-host |
| **Terraform** | No | Yes |
| **Helm** | Partial | Full |
| **K8s manifests** | No | Yes (kustomize manager) |
| **Custom version files** | No | Yes (regex manager) |
| **PR grouping** | Limited | Rich |
| **Scheduling** | Basic | Full cron-like syntax |
| **Auto-merge policies** | PR-level only | Per update-type rules |
| **Config inheritance** | Per-repo only | Presets + global config |
| **Monorepo support** | Basic (per-directory configs) | Excellent (packageRules + matchPaths) |
| **Self-hosted** | No | Yes |
| **Cost** | Free | Free (self-hosted) or paid (Mend hosted) |

---

## The interview answer in 60 seconds

> "For a polyglot monorepo, I'd pick Renovate. Dependabot has real gaps: no Terraform, limited Helm, no custom managers for internal version files. In a polyglot setup, those gaps mean you still have manual updates for a significant chunk of your dependency surface.
>
> Renovate's winning features for a monorepo: manager model (one config handles npm, Go, Terraform, Helm, GitHub Actions, Docker simultaneously), grouping (50 AWS SDK patch updates become one PR), and auto-merge policies (patch updates that pass CI merge automatically, major updates require a human reviewer).
>
> The trade-off: Renovate requires installation (GitHub App or self-hosted) and a non-trivial `renovate.json`. For a small team with a single Node repo, Dependabot is the right answer — zero friction. For a platform team managing a polyglot monorepo, Renovate pays for itself in reduced PR noise within a week.
>
> One operational gotcha: if you're also using ArgoCD Image Updater, configure Renovate to ignore the kustomization files Image Updater manages. Otherwise both tools race to update the same lines."

---

## Self-test drills

### 1. What's the difference between Dependabot security updates and version updates?

**Reference answer:**
- **Security updates**: triggered by GitHub's vulnerability database. When a dependency has a known CVE and a fixed version exists, Dependabot automatically opens a PR to bump to the fixed version. This runs regardless of your `dependabot.yml` schedule.
- **Version updates**: scheduled bumps to the latest version (not necessarily security-related). Configured in `dependabot.yml` with a schedule (daily, weekly, monthly). These are the routine "keep up-to-date" PRs.
- Both generate PRs. Security updates are higher priority and run independently. In Renovate, both are handled by the same mechanism — the `vulnerabilityAlerts` config option controls whether security PRs are prioritized.

### 2. What is a Renovate custom manager and when would you use one?

**Reference answer:**
- A custom manager uses a regex pattern to extract version strings from any file type, then queries a configurable datasource (GitHub releases, Docker Hub, npm, etc.) for updates.
- Use when you have a non-standard version file: an internal `versions.yaml` that pins tool versions, a CloudFormation parameter file with an AMI version, a shell script that hard-codes a CLI version.
- Example: a `.tool-versions` file (asdf) or a `Makefile` with `KUBECTL_VERSION := 1.30.2`. Without a custom manager, Renovate ignores these. With a custom manager + a `# renovate: datasource=...` comment, Renovate updates them like any other dependency.
- The payoff: you stop having "we're on kubectl 1.27 in CI but 1.30 in production" drift, because Renovate PRs the update when a new release is published.

### 3. How would you structure Renovate to minimize PR noise in a large monorepo?

**Reference answer:**
- Group patch updates by ecosystem: "all AWS SDK patches → one PR," "all React patches → one PR." Reduces 50 PRs to 5.
- Auto-merge patch PRs that pass CI. These are overwhelmingly safe and don't need human review.
- Schedule minor updates weekly (Monday morning). Batch them by group. Human review required.
- Major updates: monthly, separate PRs per package, required reviewers from the owning team.
- Use `ignorePaths` for frozen directories (compliance-locked services that can't be updated without a change-management process).
- Add a dashboard PR (`renovate/dashboard` branch) — Renovate's "Dependency Dashboard" issue tracks all pending updates in one place, which reduces notification noise from individual PRs.

---

## Further reading

- [Renovate docs](https://docs.renovatebot.com/) — especially the configuration options and presets
- [Renovate config presets](https://docs.renovatebot.com/presets-config/) — start with `config:base`
- [Renovate self-hosted setup](https://docs.renovatebot.com/self-hosting/) — Docker/K8s deployment
- [Dependabot docs](https://docs.github.com/en/code-security/dependabot) — the GitHub reference
- [Renovate playground](https://app.renovatebot.com/dashboard) — test your config without affecting real repos

---

## The 4 dimensions (senior framing)

- **Tech**: Dependabot for simple/GitHub-only setups. Renovate for polyglot monorepos: manager model, grouping, scheduling, auto-merge policies by update type, custom managers for non-standard files. Partition ownership with Image Updater: Renovate → Dockerfiles/deps; Image Updater → kustomization/Helm deployed tags.
- **People**: dependency update PRs are a team habit issue, not a tooling issue. Even with auto-merge for patches, minor/major PRs pile up if the team doesn't review them. Add "review dependency PRs" to the weekly team ritual — 15 minutes, batch review. Use Renovate's Dependency Dashboard issue to give a single pane of glass instead of 30 notification emails.
- **CI/CD**: auto-merge only works if CI is trustworthy. Before enabling auto-merge for patch updates, verify: does your test suite actually catch regressions? If the answer is "mostly," add a canary deploy step (deploy the auto-merged change to staging for 24 hours before production). Don't enable auto-merge until you trust your CI.
- **Operations**: Renovate's self-hosted cron runs periodically. Monitor for silent failures (Renovate can fail to open PRs due to API rate limits, Git permission issues, or config errors). Add alerting on "Renovate hasn't opened a PR in 14 days" — that's the signal that it's broken. For Dependabot: check the "Insights → Dependency graph → Dependabot" tab monthly to verify it's still running and not stuck on a merge conflict.
