# ArgoCD Image Updater — the simple version (the automatic price tag updater)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **ArgoCD Image Updater polls your container registry for new image tags matching a policy, then commits the new tag back to Git — so ArgoCD can pick it up and deploy it. The whole loop is automatic, Git-based, and auditable.**

---

## The price tag updater analogy

Imagine a store where every price tag must be approved and filed in the official price ledger (Git) before it goes on the shelf (cluster). Someone has to:

1. Check the supplier's catalog for new prices (poll the container registry)
2. If there's a new price matching our update policy, write it in the ledger (commit to Git)
3. The store manager (ArgoCD) sees the ledger update and puts the new price tag on the shelf (deploys the image)

That's Image Updater. It closes the loop between "a new image was pushed" and "Git is updated" and "ArgoCD deploys it."

Without Image Updater, you'd have to manually update the image tag in Git every time a new build is pushed. That's manual, error-prone, and slow.

---

## The three components of the loop

```
CI builds image → pushes my-app:v1.3.0 to registry
                           ↓
Image Updater polls registry every 2 minutes
Sees v1.3.0 matches the semver policy
                           ↓
Image Updater commits "image: my-app:v1.3.0" to Git
                           ↓
ArgoCD sees the Git change
Syncs the new image to the cluster
```

Every step is traceable: the new tag is in the registry (audit), the Git commit is in history (audit), the ArgoCD sync is in the UI (audit).

---

## The update strategies

Image Updater decides "should I update?" based on a strategy:

| Strategy | Rule | Example |
|---|---|---|
| `semver` | Update to the highest tag matching a constraint | `>= 1.0.0, < 2.0.0` → picks `v1.9.5` |
| `latest` | Always update to the most recently pushed tag | Useful for pre-release / dev images |
| `digest` | Update to the latest digest of a specific tag (e.g., `:main`) | For mutable tags |
| `name` | Alphabetically latest tag matching a pattern | Less common |

`semver` is the most production-appropriate strategy — you define the acceptable range, and Image Updater picks the highest matching tag.

---

## The write-back modes

Image Updater has two ways to write back the updated tag:

| Mode | How it works | When to use |
|---|---|---|
| **Annotation write-back** | Stores the tag in ArgoCD Application annotations (no Git commit) | Fast, simple, but not in Git history |
| **Git write-back** | Commits the new tag to a Git branch (a `parameters/` or `kustomization.yaml` file) | True GitOps: tag is in Git history |

For true GitOps — use **Git write-back**. The Git commit is the deployment record. Annotation write-back is convenient but the tag change isn't in Git, so rollback requires a separate mechanism.

---

## Self-test (the killer question)

Out loud:

> **"Walk me through how Image Updater detects a new image and commits the bump back to Git."**

**Reference answer (intuitive version):**

"Image Updater is a separate component that polls your container registry on a configurable interval — usually every 2-5 minutes. You annotate your ArgoCD Application to enable it and specify the image and update policy. When Image Updater finds a new tag that matches the policy (for example, a semver range), it commits the new image tag to the Git repository using Git write-back mode. The commit lands in the configured branch. ArgoCD sees the Git change and syncs the new image to the cluster.

The key design point: Image Updater doesn't deploy anything directly. It only updates Git. ArgoCD does the actual deployment. This keeps the full GitOps property — Git is the source of truth, every deployment is a Git commit, and rollbacks are just Git reverts."

---

## Further reading

- [ArgoCD Image Updater docs](https://argocd-image-updater.readthedocs.io/)
- [Git write-back configuration](https://argocd-image-updater.readthedocs.io/en/stable/configuration/applications/#git-write-back-method)

---

## Next: the deep-dive

When the price-tag-updater analogy feels obvious, jump to [`argocd-image-updater.md`](./deep-dive.md). The deep-dive covers:

- The polling model and how to configure it
- All four update strategies in detail
- Git write-back vs annotation write-back (with YAML examples)
- The Git credentials and branch setup
- Per-Application annotation configuration
- The trade-off vs Renovate ("K8s-aware but K8s-only vs handles everything")
- 3 self-test drills + 4-dimensions framing
