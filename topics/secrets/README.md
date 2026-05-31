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

(none yet — to be written)

## Quick comparison

| Tool | Storage | Decryption | Best for |
|---|---|---|---|
| Sealed Secrets | Encrypted YAML in Git | Controller (cluster's private key) | Small teams, single cluster |
| External Secrets | External KMS (AWS SM, Vault) | ESO controller fetches | Multi-cluster, existing KMS |
| SOPS | Encrypted YAML in Git | Apply-time (operator's keys) | Self-hosted, no controller |

## Why this matters

Secrets in GitOps is the question every interviewer asks once you say "we use ArgoCD." There's no single right answer — but you need a defensible opinion.
