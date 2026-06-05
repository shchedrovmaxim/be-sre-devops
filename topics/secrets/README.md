# Secrets management

Storing and delivering secrets safely in a K8s + GitOps world. You can't commit raw secrets to Git, but you also can't manually kubectl-apply them.

## Recommended order

1. **The problem** — why K8s `Secret` resources need a layer above them in GitOps
2. **Sealed Secrets** (Bitnami) — encrypt with cluster's public key; only the controller can decrypt
3. **External Secrets Operator (ESO)** — sync from AWS Secrets Manager / Vault into K8s Secrets
4. **SOPS + age/KMS** — file-level encryption (helm-secrets, kustomize-sops)
5. **Vault** as source of truth + ESO as the sync mechanism (common enterprise pattern)
6. **When to pick each** — trade-offs table

## Files

| File | Type | Topic | Date |
|---|---|---|---|
| [sealed-secrets-simple.md](./sealed-secrets/simple.md) | Simple companion | Sealed Secrets — wax-seal analogy, multi-cluster dealbreaker | 2026-06-05 |
| [sealed-secrets.md](./sealed-secrets/deep-dive.md) | Deep-dive | Sealed Secrets — RSA+AES mechanics, scope binding, key rotation, backup, multi-cluster failure | 2026-06-05 |
| [external-secrets-operator-simple.md](./external-secrets-operator/simple.md) | Simple companion | ESO — personal shopper analogy, SecretStore vs ClusterSecretStore, refresh model | 2026-06-05 |
| [external-secrets-operator.md](./external-secrets-operator/deep-dive.md) | Deep-dive | ESO — full reconciliation loop, IRSA auth chain, generators, failure modes, Kyverno integration | 2026-06-05 |
| [sops-simple.md](./sops/simple.md) | Simple companion | SOPS + age/KMS — redacted-document analogy, per-value encryption, diff-friendliness | 2026-06-05 |
| [sops.md](./sops/deep-dive.md) | Deep-dive | SOPS + age/KMS — DEK+KEK hybrid, `.sops.yaml` creation rules, ArgoCD+KSOPS integration | 2026-06-05 |
| [vault-eso-pattern-simple.md](./vault-eso-pattern/simple.md) | Simple companion | Vault + ESO — bank vault analogy, secret zero problem, dynamic secrets overview | 2026-06-05 |
| [vault-eso-pattern.md](./vault-eso-pattern/deep-dive.md) | Deep-dive | Vault + ESO — HA Raft, auto-unseal, K8s auth, dynamic DB creds, 50-service design | 2026-06-05 |
| [secrets-comparison-simple.md](./secrets-comparison/simple.md) | Simple companion | Decision matrix — in-Git vs runtime-pull axis, multi-cluster dealbreaker, rotation comparison | 2026-06-05 |
| [secrets-comparison.md](./secrets-comparison/deep-dive.md) | Deep-dive | Decision matrix — full 8-dimension trade-off matrix, decision tree, Kyverno enforcement, Wiz bridge | 2026-06-05 |

## Quick comparison

| Tool | Storage | Decryption | Best for |
|---|---|---|---|
| Sealed Secrets | Encrypted YAML in Git | Controller (cluster's private key) | Small teams, single cluster |
| External Secrets | External KMS (AWS SM, Vault) | ESO controller fetches | Multi-cluster, existing KMS |
| SOPS | Encrypted YAML in Git | Apply-time (operator's keys) | Self-hosted, no controller |

## Why this matters

Secrets in GitOps is the question every interviewer asks once you say "we use ArgoCD." There's no single right answer — but you need a defensible opinion.
