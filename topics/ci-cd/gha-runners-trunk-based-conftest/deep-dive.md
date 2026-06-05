# GHA self-hosted runners + trunk-based dev + Conftest — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through the modern CI/CD architecture you'd recommend for a small SRE team: GHA runners, branching strategy, policy checks"** — naming ARC, trunk-based with feature flags, immutable promotion, and Conftest in CI with Rego.

> Start with [gha-runners-trunk-based-conftest-simple.md](./simple.md) if you haven't.

---

## The senior framing — CI as architecture

Mid-level: "CI runs the tests. CD deploys."
Senior: "CI is the architecture enforcement layer. It's where branching strategy, security policy, artifact immutability, and deployment gates are expressed as code. The quality of your CI design predicts the quality of your deployments."

The interview signal is articulating CI as a deliberate architectural choice, not a collection of scripts.

---

## GHA self-hosted runners — the decision

### When GitHub-hosted runners are not enough

GitHub-hosted runners (`ubuntu-latest`, `windows-latest`) are ephemeral, managed VMs with outbound internet access. They cannot:

- Access resources inside your VPC (internal registries, private databases, internal API endpoints)
- Provide a persistent local cache (each job is a fresh VM; S3-based caching is possible but adds latency)
- Run on custom hardware (GPU nodes for ML CI, arm64 for cross-compile, FPGA for hardware testing)
- Use managed identities (IRSA, Workload Identity) without network access to the K8s cluster

The decision tree:

```
Does the job need VPC access? → YES → self-hosted runner
Does the job need a persistent local cache? → YES → self-hosted runner (or evaluate GitHub cache actions)
Does the job need custom hardware? → YES → self-hosted runner
Everything else? → GitHub-hosted runner
```

### Actions Runner Controller (ARC)

ARC is the production approach for self-hosted runners on Kubernetes. It creates ephemeral runner pods — each job gets a fresh pod, which is deleted after the job completes. No persistent runner VMs to patch, no "runner has accumulated state" bugs.

```yaml
# HorizontalRunnerAutoscaler: auto-scale based on pending jobs
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: my-runner-pool
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: my-runners
  minReplicas: 0          # scale to zero when no jobs are pending
  maxReplicas: 10
  metrics:
    - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
      repositoryNames:
        - myorg/myrepo
```

```yaml
# RunnerDeployment: defines the runner pod template
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: my-runners
spec:
  template:
    spec:
      repository: myorg/myrepo
      labels:
        - self-hosted
        - linux
        - vpc-private
      serviceAccountName: ci-runner   # IRSA or Workload Identity for cloud API access
      image: summerwind/actions-runner:latest
```

The `serviceAccountName` is the key for K8s-based CI: the runner pod gets the SA's credentials, which grants it IRSA permissions to push to ECR, pull from private registries, etc. — without embedding secrets in the runner.

### Self-hosted runner security

The self-hosted runner executes arbitrary code from your CI workflows. This means: anyone who can open a PR and trigger a CI workflow could potentially exfiltrate secrets from the runner's environment.

Mitigations:
- **Only use self-hosted runners for trusted repositories** (private repos with restricted PR access). Never for public repos — a malicious PR could run code in your VPC.
- **Ephemeral runners** (ARC's default): each job is a fresh pod. No persistence between jobs means no state accumulation.
- **Minimal IRSA permissions**: the runner's service account should have only the permissions needed (e.g., ECR push to one specific repo, not the whole registry).
- **Network segmentation**: run runners in a dedicated subnet with egress control. They should reach GitHub's API and your specific resources, not your entire VPC.

---

## Trunk-based development — the mechanics

### Core rules

1. **One integration branch** (`main` / `trunk`). All developers integrate here.
2. **Short-lived branches**: if you use feature branches, they must merge to `main` within 24 hours. No week-long branches.
3. **Every commit to `main` is potentially releasable**: the CI gate ensures this. If a commit breaks the build or tests, it's reverted or fixed immediately.
4. **Feature flags for incomplete work**: code ships to production turned off. The feature is released by enabling the flag.

### Feature flags vs feature branches

| | Feature branches | Feature flags |
|---|---|---|
| Incomplete work isolation | In the branch | In production, behind a flag |
| Integration with other features | Only at merge time | Continuous (both features in main, flags off) |
| Rollback | Revert the branch | Flip the flag |
| Merge complexity | High (long-lived = big diff) | Low (tiny commits) |
| Production incidents | "This branch is not in prod" | "Flag is off; this code is in prod but inactive" |

Feature flags are more operationally complex to manage (you need a flag management system — LaunchDarkly, Unleash, or home-grown) but produce a safer, more stable main branch.

### Immutable promotion — the artifact model

```
CI builds → Image tagged: myapp:sha256-abc123 (digest-based, immutable)
                ↓
Deploy to staging: argocd sync (same image, different namespace)
                ↓ (after staging passes)
Deploy to production: argocd sync (SAME IMAGE, different namespace)
```

The key: **the image is never rebuilt between environments.** The same artifact that passed staging tests goes to production. Rebuilding for production is a common mistake — you change the environment variable and suddenly you're deploying code that was never tested.

Immutable means: tag by digest or by commit SHA, not by `latest`. The image tag in the kustomization or Helm values is pinned to the immutable digest.

### The promotion gate

Options for controlling staging → production promotion:

1. **Time-based soak**: staging runs for 24 hours with no alerts → auto-promote
2. **Manual approval**: GitHub Actions environment protection rules; a reviewer approves the production deploy
3. **Metric-based**: check SLO dashboards before promoting (automated or manual)
4. **Canary first**: deploy 10% production traffic to new version → check error rate → promote to 100%

```yaml
# GitHub Actions: production deploy requires approval
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: argocd app sync my-app-staging

  deploy-production:
    environment: production   # environment with required reviewers configured
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: argocd app sync my-app-production
```

The `environment: production` with required reviewers creates a manual approval gate in GitHub Actions. The run pauses until a reviewer approves.

---

## Conftest — policy testing for configuration

Conftest runs OPA Rego policies against structured files. It doesn't test your application code — it tests your **configuration** (K8s manifests, Terraform plans, Helm values, Dockerfiles).

### Installation and basic usage

```bash
# Install (macOS)
brew install conftest

# Test K8s manifests
conftest test deployment.yaml --policy policies/

# Test a Terraform plan (JSON output)
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policies/terraform/

# Test Helm-rendered manifests
helm template my-chart > rendered.yaml
conftest test rendered.yaml --policy policies/k8s/
```

### Rego policy structure for K8s

```rego
# policies/k8s/security.rego
package kubernetes.security

# Deny containers running as root
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.securityContext.runAsNonRoot == true
  msg := sprintf("Container '%s' must set runAsNonRoot: true", [container.name])
}

# Deny latest tag
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Container '%s' must not use ':latest' tag", [container.name])
}

# Deny missing resource limits
warn[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.resources.limits.cpu
  msg := sprintf("Container '%s' is missing CPU limits", [container.name])
}
```

`deny` fails the test. `warn` passes but prints a warning. This matches the severity model of other policy tools.

### Rego policy for Terraform

```rego
# policies/terraform/s3.rego
package terraform.aws.s3

# Deny public S3 buckets
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.after.block_public_acls == false
  msg := sprintf("S3 bucket '%s' must have block_public_acls enabled", [resource.address])
}

# Deny S3 buckets without encryption
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not resource.change.after.server_side_encryption_configuration
  msg := sprintf("S3 bucket '%s' must have server-side encryption enabled", [resource.address])
}
```

### CI integration

```yaml
# .github/workflows/ci.yaml
jobs:
  conftest-k8s:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install conftest
        run: |
          VERSION=0.51.0
          curl -L "https://github.com/open-policy-agent/conftest/releases/download/v${VERSION}/conftest_${VERSION}_Linux_x86_64.tar.gz" | tar xzf -
          sudo mv conftest /usr/local/bin/

      - name: Render Helm charts
        run: |
          helm template my-app ./charts/my-app \
            --values ./charts/my-app/values-production.yaml \
            > /tmp/manifests.yaml

      - name: Run Conftest
        run: |
          conftest test /tmp/manifests.yaml \
            --policy ./policies/k8s/ \
            --output github   # GitHub annotation format
```

The `--output github` flag produces GitHub check annotations pointing at specific lines in PRs. Combined with the `conftest` run on every PR, developers see policy violations as PR check failures with line annotations.

### Linking Conftest and Kyverno policies

The senior insight: run the same policy logic in two places.

- **Conftest in CI**: catches violations before the manifest reaches the cluster. Fast feedback in PRs.
- **Kyverno in cluster**: catches violations at admission time. Defense-in-depth for things that bypass CI (direct kubectl, broken CI config).

Write the Rego policy once, use it in Conftest. Convert the same logic to Kyverno YAML for cluster enforcement. They're different syntaxes for the same rule. When you update the policy (new security requirement), update both.

Some teams use [Kyverno CLI](https://kyverno.io/docs/kyverno-cli/) in CI instead of Conftest — it validates against the same ClusterPolicy YAMLs that the cluster enforces. This removes the Rego-vs-YAML translation step entirely.

---

## The complete CI/CD architecture for a small SRE team

```
Developer pushes to main (trunk-based, short-lived branches only)
          ↓
GitHub Actions (github-hosted runner):
  - Build image (multi-stage, distroless)
  - Run unit tests + linting
  - Generate manifests (helm template or kustomize build)
  - Conftest: policy check on manifests + Terraform plan
  - Trivy: image scan (--ignore-unfixed --severity HIGH,CRITICAL)
  - Syft: generate SBOM
  - Push image to ECR (self-hosted runner with IRSA, if ECR is VPC-private)
  - Cosign: sign image (keyless OIDC)
  - Cosign: attest SBOM
  - Commit image tag to Git (via Image Updater or direct commit step)
          ↓
ArgoCD detects Git change → auto-sync to staging
          ↓
Staging soak (24h or metric-based gate)
          ↓
Manual approval (GitHub environment protection)
          ↓
ArgoCD syncs SAME IMAGE to production
```

Every step is traceable. The image digest is the deployment ID. The Git commit history is the deployment log. ArgoCD's sync history is the rollback menu.

---

## The interview answer in 60 seconds

> "The architecture I'd recommend: GitHub-hosted runners for 90% of CI — ephemeral, managed, no maintenance. Self-hosted runners via ARC (ephemeral pods on K8s) only for jobs that need VPC access or IRSA — specifically image push to ECR and integration tests against private resources.
>
> Branching: trunk-based development. Everyone works on `main`, short-lived branches that merge within a day. Feature flags for incomplete work. CI must pass before merge — branch protection enforces this. Every commit to main is potentially deployable.
>
> Policy in CI: Conftest runs against rendered K8s manifests and Terraform plans on every PR. Same rules as Kyverno in the cluster — defense-in-depth. Catches the `latest` tag, missing resource limits, public S3 buckets before they reach the cluster.
>
> Immutable promotion: the image is tagged by commit SHA, never rebuilt. The same image artifact that passed staging testing is what deploys to production — via ArgoCD syncing both environments from Git. Production deploy requires manual approval via GitHub environment protection rules.
>
> The senior framing: CI is architecture. It's where you encode the team's quality bar — security policy, deployment gates, immutability invariants. If CI is just 'the thing that runs tests,' you're missing most of its value."

---

## Self-test drills

### 1. What's the security risk of using self-hosted runners on public repositories?

**Reference answer:**
- A PR from a fork (untrusted external contributor) can contain malicious workflow changes that run arbitrary code on your self-hosted runner.
- The runner has access to your VPC, your secrets (via IRSA or environment variables), and potentially your production systems.
- A malicious actor opens a PR, triggers CI, exfiltrates secrets or moves laterally in the VPC.
- Mitigations: never use self-hosted runners on public repos. On private repos with controlled PR access, use ephemeral runners (ARC) so each job starts clean. Apply minimal-privilege IRSA. Run runners in a separate network segment.

### 2. What's the difference between a feature flag and a feature branch?

**Reference answer:**
- Feature branch: the incomplete code lives in a separate branch. Integration with other developers' work only happens at merge time. Long-lived branches accumulate merge debt and conflict.
- Feature flag: the incomplete code is in `main` (and therefore in production), but disabled for users. Integration is continuous — every day, the code is tested with all other changes. The feature is "released" by enabling the flag.
- The operational difference: with a feature flag, rollback is flipping a flag (seconds). With a feature branch, rollback is reverting the branch merge (minutes to hours, depending on what merged since).
- The cost: you need a flag management system and discipline to remove old flags (flag debt accumulates). Worth it for teams that deploy frequently.

### 3. How does Conftest complement Kyverno? Why run policy checks in both places?

**Reference answer:**
- Conftest in CI: catches violations at PR time, before the manifest is committed to Git or reaches the cluster. Fast feedback (seconds), in the PR check UI. Developers fix the issue before it propagates.
- Kyverno in cluster: catches violations at admission time. Defense-in-depth — handles cases that bypass CI (direct `kubectl apply`, broken CI config, ArgoCD syncing a manifest that slipped through).
- Together: shift-left (CI) + defense-in-depth (cluster). If Conftest catches 95% of violations in PRs, Kyverno's admission denials become rare events — but they're the safety net.
- Maintenance note: keep the policies in sync. When you add a new Kyverno rule, add the equivalent Conftest Rego. When you remove a policy, remove it from both. The Kyverno CLI (`kyverno apply`) can run Kyverno policies in CI against manifest files, removing the "two syntaxes for the same rule" burden.

---

## Further reading

- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller) — K8s-based ephemeral runners
- [Trunk-based development guide](https://trunkbaseddevelopment.com/) — the canonical reference, with patterns and antipatterns
- [Conftest docs](https://www.conftest.dev/) — policy testing for configuration
- [Kyverno CLI](https://kyverno.io/docs/kyverno-cli/) — run Kyverno policies in CI without a cluster
- [Feature flags with Unleash](https://www.getunleash.io/) — OSS flag management system
- [LaunchDarkly](https://launchdarkly.com/) — hosted flag management, the industry standard

---

## The 4 dimensions (senior framing)

- **Tech**: GitHub-hosted runners for standard CI; ARC ephemeral pods for VPC/IRSA needs. Trunk-based: `main` always deployable, feature flags for incomplete work, short-lived branches (<1 day). Conftest in CI against rendered manifests and Terraform plans. Immutable image promotion: digest-tagged, never rebuilt. ArgoCD for both staging and production from Git.
- **People**: trunk-based development is a cultural shift. Teams used to feature branches resist it. The pitch: "you spent 2 days last sprint resolving merge conflicts — that's avoidable." Start with a protected main branch + automated CI + very short branch lifetimes. Feature flags are the harder sell — they require organizational trust that incomplete code in production (flagged off) is safe.
- **CI/CD**: the CI pipeline is the team's architecture document — readable as a sequence of steps. Keep workflows short (< 5 min for the PR gate) by running expensive tests in parallel. Run Conftest + Trivy in parallel, not sequential. Cache the Trivy DB daily. Push image only after scans pass — don't push and then scan; scan before push.
- **Operations**: monitor CI pipeline duration as an SLI. A slow CI pipeline is a velocity blocker — teams start cutting corners (skipping the slow step). Target: PR gate < 5 min. Alert if median CI duration exceeds 10 min. Self-hosted runner autoscaling (ARC): monitor pending job queue depth — if jobs are queuing more than 2 min, scale out. The ARC `HorizontalRunnerAutoscaler` handles this, but verify the scaling metrics are wired correctly.
