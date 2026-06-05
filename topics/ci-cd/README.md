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
| Multi-stage Docker builds — simple version (kitchen vs dining-room analogy) | [`multi-stage-builds-simple.md`](./multi-stage-builds-simple.md) | 2026-06-04 |
| Image scanning (Trivy, Grype) — deep-dive: layer-walking, NVD/GHSA, false positives, CI integration, OSS vs Wiz | [`image-scanning.md`](./image-scanning.md) | 2026-06-05 |
| Image scanning — simple version (health inspector analogy) | [`image-scanning-simple.md`](./image-scanning-simple.md) | 2026-06-05 |
| SBOM (Syft) — deep-dive: formats, component discovery, Cosign attestation, EO 14028 | [`sbom-syft.md`](./sbom-syft.md) | 2026-06-05 |
| SBOM (Syft) — simple version (ingredient label analogy) | [`sbom-syft-simple.md`](./sbom-syft-simple.md) | 2026-06-05 |
| Cosign / Sigstore keyless signing — deep-dive: Fulcio, Rekor, OIDC trust chain, Kyverno verifyImages | [`cosign-sigstore.md`](./cosign-sigstore.md) | 2026-06-05 |
| Cosign / Sigstore — simple version (wax seal analogy) | [`cosign-sigstore-simple.md`](./cosign-sigstore-simple.md) | 2026-06-05 |
| Kyverno — deep-dive: ClusterPolicy, validate/mutate/generate/verifyImages, PolicyException, ClusterPolicyReport | [`kyverno.md`](./kyverno.md) | 2026-06-05 |
| Kyverno — simple version (bouncer + auto-fixer analogy) | [`kyverno-simple.md`](./kyverno-simple.md) | 2026-06-05 |
| OPA Gatekeeper — deep-dive: ConstraintTemplate, Rego, audit mode, multi-cluster, Kyverno vs Gatekeeper | [`opa-gatekeeper.md`](./opa-gatekeeper.md) | 2026-06-05 |
| OPA Gatekeeper — simple version (law library analogy) | [`opa-gatekeeper-simple.md`](./opa-gatekeeper-simple.md) | 2026-06-05 |
| ArgoCD Application + AppProject — deep-dive: CRD anatomy, multi-tenant pattern, auto-sync gotchas | [`argocd-application-appproject.md`](./argocd-application-appproject.md) | 2026-06-05 |
| ArgoCD Application + AppProject — simple version (tenant office map analogy) | [`argocd-application-appproject-simple.md`](./argocd-application-appproject-simple.md) | 2026-06-05 |
| ArgoCD sync waves + hooks — deep-dive: PreSync/PostSync/SyncFail, deletion policies, migration pattern | [`argocd-sync-waves-hooks.md`](./argocd-sync-waves-hooks.md) | 2026-06-05 |
| ArgoCD sync waves + hooks — simple version (construction sequence analogy) | [`argocd-sync-waves-hooks-simple.md`](./argocd-sync-waves-hooks-simple.md) | 2026-06-05 |
| ArgoCD ignoreDifferences + SSA — deep-dive: controller fights, jsonPointers, managedFields, common cases | [`argocd-ignoredifferences-ssa.md`](./argocd-ignoredifferences-ssa.md) | 2026-06-05 |
| ArgoCD ignoreDifferences + SSA — simple version (shared whiteboard analogy) | [`argocd-ignoredifferences-ssa-simple.md`](./argocd-ignoredifferences-ssa-simple.md) | 2026-06-05 |
| ArgoCD app-of-apps + ApplicationSet — deep-dive: generators, fleet pattern, deletion gotchas | [`argocd-appofapps-applicationset.md`](./argocd-appofapps-applicationset.md) | 2026-06-05 |
| ArgoCD app-of-apps + ApplicationSet — simple version (manager vs staffing agency analogy) | [`argocd-appofapps-applicationset-simple.md`](./argocd-appofapps-applicationset-simple.md) | 2026-06-05 |
| ArgoCD Image Updater — deep-dive: polling, strategies, Git write-back, Renovate comparison | [`argocd-image-updater.md`](./argocd-image-updater.md) | 2026-06-05 |
| ArgoCD Image Updater — simple version (price tag updater analogy) | [`argocd-image-updater-simple.md`](./argocd-image-updater-simple.md) | 2026-06-05 |
| Renovate vs Dependabot — deep-dive: manager model, grouping, auto-merge, monorepo config | [`renovate-vs-dependabot.md`](./renovate-vs-dependabot.md) | 2026-06-05 |
| Renovate vs Dependabot — simple version (junior vs senior assistant analogy) | [`renovate-vs-dependabot-simple.md`](./renovate-vs-dependabot-simple.md) | 2026-06-05 |
| GHA runners + trunk-based dev + Conftest — deep-dive: ARC, feature flags, immutable promotion, CI as architecture | [`gha-runners-trunk-based-conftest.md`](./gha-runners-trunk-based-conftest.md) | 2026-06-05 |
| GHA runners + trunk-based dev + Conftest — simple version (assembly line analogy) | [`gha-runners-trunk-based-conftest-simple.md`](./gha-runners-trunk-based-conftest-simple.md) | 2026-06-05 |

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
