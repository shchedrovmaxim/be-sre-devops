# CI/CD + DevOps ecosystem

The pipeline from code commit to production. Senior signal: you treat CI as architecture, not "the thing that runs tests."

## Recommended order

### Containers + images (⚠️ rejection gap)
1. **Multi-stage Docker builds** + BuildKit cache mounts
2. **Distroless / scratch** base images
3. **Image scanning** — Trivy, Grype
4. **SBOM** — Syft
5. **Image signing** — Cosign / Sigstore (keyless OIDC)
6. **Admission policy enforcement** — Kyverno / OPA Gatekeeper

### GitOps (⚠️ rejection gap)
7. **ArgoCD: Application** + AppProject (multi-team)
8. **ArgoCD: sync waves, sync hooks** (PreSync / Sync / PostSync / SyncFail)
9. **ArgoCD: ignoreDifferences + Server-Side Apply** (handling webhook mutations)
10. **ArgoCD: app-of-apps** pattern
11. **ApplicationSet** — fan-out across clusters

### Image / dep automation (⚠️ rejection gap)
12. **ArgoCD Image Updater** — when, how, where it commits back
13. **Renovate** — config, when over Dependabot
14. **Dependabot** — language deps, GitHub Actions
15. The comparison: which tool for which job

### CI itself
16. **GitHub Actions** runners (self-hosted patterns)
17. **Trunk-based development** + immutable promotion across envs
18. **Conftest / OPA** in CI

## Files

| Sub-topic | File | Date covered |
|---|---|---|
| Multi-stage Docker builds + BuildKit cache mounts, distroless base images, security framing | [`multi-stage-builds.md`](./multi-stage-builds.md) | 2026-06-04 |

## Why this matters for senior SRE interviews

> "Significant gaps were identified regarding key tools and concepts, such as Let's Encrypt, multi-stage builds, and automation solutions like ArgoCD image-updater, Renovate, or Dependabot."

This was the exact rejection quote. Closing this gap is non-negotiable.

> "Unconvincing architectural approach … neglecting … CI-related aspects."

Senior architects describe **how code reaches prod**, not just what runs in prod.

## Hands-on environment

- Local `kind` cluster + ArgoCD install via Helm
- A free GitHub repo to test Actions + Renovate
- `docker buildx` for BuildKit features
- `cosign`, `syft`, `trivy` installable locally
