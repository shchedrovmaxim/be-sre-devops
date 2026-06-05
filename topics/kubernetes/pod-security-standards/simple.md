# Pod Security Standards — the simple version (the nightclub bouncer analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **PSS is a nightclub with three rooms. Each room has different rules. A namespace label tells the bouncer which room your pods are in — and what happens when they break the rules.**

That's the whole model. PodSecurityPolicy (PSP) was the old bouncer. It was fired in K8s 1.25. PSS replaced it.

---

## Three rooms, three rule sets

```
┌─────────────────────────────────────────────────────────┐
│  privileged room  — no rules. Run as root. Mount anything. │
│  (used only for system infrastructure pods)             │
├─────────────────────────────────────────────────────────┤
│  baseline room  — sane defaults. No host networking,    │
│  no hostPID, no privileged containers.                  │
│  (OK for most workloads)                                │
├─────────────────────────────────────────────────────────┤
│  restricted room  — strictest. Must run as non-root.    │
│  Read-only filesystem. Drop ALL capabilities. Seccomp.  │
│  (what you want for any internet-facing service)        │
└─────────────────────────────────────────────────────────┘
```

The **namespace label** decides which room a namespace belongs to.

---

## The three modes — what the bouncer does when a pod breaks the rules

| Mode | What happens | When to use |
|---|---|---|
| `enforce` | Pod is **rejected**. It never starts. | Production — block non-compliant pods outright. |
| `audit` | Pod starts, but a **warning is logged** in the audit log. | When you're rolling out PSS and want to see what would be blocked. |
| `warn` | Pod starts, but the user gets a **warning in their terminal**. | Developer feedback — they see the warning, nothing breaks yet. |

You can mix modes per level on the same namespace:

```yaml
# "Enforce baseline, warn about restricted violations"
metadata:
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

This is the recommended migration path: start with warn/audit, fix violations, then switch enforce to restricted.

---

## The three labels (the entire API)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted    # mode: level
    pod-security.kubernetes.io/enforce-version: v1.28 # pin to a K8s version
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

That's the entire PSS API. Three labels. No CRDs, no webhooks, no admission controllers to operate. It's built into the K8s apiserver.

---

## PSP is gone — what replaced it?

PodSecurityPolicy (PSP) was the old way. It was deprecated in K8s 1.21 and removed in 1.25.

| PSP | PSS |
|---|---|
| A cluster-level CRD + RBAC binding | Namespace labels only |
| Mutating (could auto-fix violations) | Non-mutating (enforces or warns) |
| Complex and confusing (many teams ran with `*` allowed everything) | Simple 3-level model |
| Removed in K8s 1.25 | Stable since K8s 1.25 |

The gap: PSS doesn't let you write *custom* rules. If you need "don't allow image from unverified registries" or "require specific labels," you need **Kyverno** or **OPA Gatekeeper**. PSS + Kyverno = the modern replacement for PSP.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What are the three PSS levels? | `privileged` (no restrictions), `baseline` (sane defaults), `restricted` (strictest) |
| What are the three modes? | `enforce` (block), `audit` (log), `warn` (terminal warning) |
| Where does PSS live? | In the K8s apiserver — namespace labels, no extra components needed. |
| What replaced PSP? | PSS for standard rules. Kyverno or OPA Gatekeeper for custom rules. |
| What does `restricted` require? | Non-root user, read-only filesystem (or explicit override), drop ALL capabilities, seccomp runtime default. |
| Can PSS auto-fix pods? | No. It only allows or blocks. If you need mutation (auto-add securityContext fields), use Kyverno. |

---

## Self-test (one question — the killer one)

Out loud:

> **"PSS replaced PodSecurityPolicy. Walk me through how the new model works and how you'd enforce 'restricted' across the cluster."**

**Reference answer (intuitive version):**

"PSS is built into the apiserver — no extra components. It has three levels: privileged (no restrictions), baseline (no hostPID, no privileged containers, sane defaults), and restricted (must run as non-root, drop all capabilities, seccomp required). You apply it per namespace via labels: `pod-security.kubernetes.io/enforce: restricted`. Three modes: enforce blocks non-compliant pods, audit logs violations without blocking, warn shows a warning to the user.

To enforce restricted cluster-wide, I'd start by labeling all namespaces with warn and audit set to restricted, and enforce set to baseline. That gives me visibility without breaking anything. I'd fix the violations I see, then flip enforce to restricted namespace by namespace. For rules that PSS can't express — like blocking specific base images or requiring specific labels — I'd add Kyverno policies on top."

---

## Further reading

- [Pod Security Standards docs](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission docs](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

---

## Next: the deep-dive

When the bouncer analogy feels obvious, jump to [`pod-security-standards.md`](./deep-dive.md). The deep-dive covers:

- The full list of what each level restricts
- The PSP → PSS migration path
- Kyverno as the custom-rules layer on top of PSS
- Exemptions (system namespaces, specific users/ServiceAccounts)
- 4 self-test drills + 4-dimensions framing
