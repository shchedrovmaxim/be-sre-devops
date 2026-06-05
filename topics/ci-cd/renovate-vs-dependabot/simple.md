# Renovate vs Dependabot — the simple version (the junior vs senior assistant)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **Dependabot is GitHub's built-in dependency updater — simple, zero config, narrow scope. Renovate is a powerful self-hosted alternative — more configuration, supports everything, smarter PR management. Dependabot to start; Renovate when you've outgrown it.**

---

## The junior vs senior assistant

**Dependabot** is like a good junior assistant: shows up automatically, follows a clear process, handles the obvious tasks without being asked, but has limited flexibility — "sorry, that's not in my job description."

**Renovate** is like a senior contractor: more setup to hire (install the app or self-host), but handles everything, has rich policies, can group updates, schedule them, auto-merge safe changes, and follow custom rules. Does exactly what you configure.

For a small team with a single GitHub repo using standard tooling: Dependabot wins — zero friction to start. For a polyglot monorepo with many managers, scheduling requirements, and complex auto-merge rules: Renovate wins.

---

## What each covers

| Ecosystem | Dependabot | Renovate |
|---|---|---|
| npm / Yarn / pnpm | Yes | Yes |
| pip / Poetry / pip-compile | Yes | Yes |
| Go modules | Yes | Yes |
| Maven / Gradle | Yes | Yes |
| Cargo (Rust) | Yes | Yes |
| GitHub Actions | Yes | Yes |
| Docker base images (Dockerfile) | Yes | Yes |
| Helm charts | Limited | Yes |
| Terraform modules + providers | No | Yes |
| Kubernetes manifests (kustomization) | No | Yes |
| Custom regex patterns | No | Yes (custom managers) |
| Grouping PRs together | Limited | Yes (rich grouping) |
| Schedule (only on weekdays, etc.) | Limited | Yes |
| Auto-merge policies | Limited | Yes |

---

## Configuration comparison

**Dependabot** — a `.github/dependabot.yml` file, ~10 lines:

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
```

Done. No installation required — it's built into GitHub.

**Renovate** — a `renovate.json` in the repo root, potentially complex:

```json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "groupName": "AWS dependencies",
      "matchPackagePatterns": ["^@aws-sdk/"]
    }
  ],
  "schedule": ["after 9am on Monday"]
}
```

More power, more config.

---

## Self-test (the killer question)

Out loud:

> **"Pick Renovate vs Dependabot for a polyglot monorepo. Walk me through the trade-offs."**

**Reference answer (intuitive version):**

"For a polyglot monorepo, I'd pick Renovate. Dependabot doesn't support Terraform modules or Kubernetes manifests — in a polyglot setup, those are probably in scope. Renovate also handles grouping: instead of 40 separate PRs for AWS SDK patch updates, one grouped PR. And auto-merge policies: patch updates that pass CI can be merged without a human reviewer, which keeps the noise manageable.

The cost is setup: Renovate requires installation (as a GitHub App or self-hosted). And the config can get complex — but you configure it once in `renovate.json` at the repo level, and every new dependency manager picks it up automatically.

One gotcha: if you're also using ArgoCD Image Updater for container images in K8s manifests, you need to configure Renovate to ignore those files. Otherwise both tools update the same lines and you get conflicts."

---

## Further reading

- [Renovate docs](https://docs.renovatebot.com/)
- [Dependabot docs](https://docs.github.com/en/code-security/dependabot)
- [Renovate config presets](https://docs.renovatebot.com/presets-config/) — start with `config:base`

---

## Next: the deep-dive

When the junior-vs-senior analogy feels obvious, jump to [`renovate-vs-dependabot.md`](./deep-dive.md). The deep-dive covers:

- Dependabot scope and configuration in detail
- Renovate's manager model and custom managers
- Grouping, scheduling, auto-merge policies
- Renovate self-hosted vs hosted (Mend Renovate)
- The Image Updater coordination problem
- The config inheritance model for monorepos
- 3 self-test drills + 4-dimensions framing
