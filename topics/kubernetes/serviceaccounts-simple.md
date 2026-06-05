# ServiceAccounts + token projection — the simple version (the hotel key card analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./serviceaccounts.md) becomes easy.

This doc only explains **one idea**:

> **A ServiceAccount is a pod's identity. The token is the key card. Before K8s 1.21, the key card never expired. After 1.21, it auto-rotates every hour — which is much safer.**

That's the whole concept. Everything else is just precision on top.

---

## Hotel key cards

You check into a hotel. They give you a plastic key card that opens your room and the gym. When you check out, the card stops working.

| Hotel world | K8s world |
|---|---|
| Your name on the booking | The **ServiceAccount** — an identity for the pod |
| Your key card | The **token** — a JWT the pod presents to the API server |
| "Opens Room 301 and the gym" | The **RBAC RoleBinding** — what the token authorizes |
| Key card auto-expires at checkout | **Bound token projection** — token expires in 1 hour, auto-renewed |
| Old hotels: card works forever | **Legacy tokens** — stored in a Secret, never expired |

The big change in K8s 1.21: new hotels started issuing **time-limited key cards** that auto-renew while you're in the room. Old hotels left cards under your door that worked until someone physically deactivated them.

---

## What actually lands inside the pod

Every pod gets a directory mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`. Inside:

```
token       ← the JWT the pod uses to call the K8s API
ca.crt      ← the cluster's CA cert (so the pod can verify the API server)
namespace   ← the pod's own namespace, as a file
```

Before K8s 1.21, `token` was a long-lived JWT from a Secret — it never changed, never expired. After K8s 1.21 with bound token projection, `token` is a short-lived JWT that:
- expires in 1 hour by default
- is automatically refreshed by kubelet before it expires
- is **audience-bound** (only valid for the K8s API server, not any other service)

If you steal the old token, you have it forever. If you steal the new token, you have at most an hour.

---

## The simplest YAML patterns

```yaml
# Per-pod: don't mount a token at all (for pods that never call the K8s API)
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-api:1.0
```

```yaml
# Per-ServiceAccount: disable auto-mount for all pods that use this SA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: worker
  namespace: jobs
automountServiceAccountToken: false
```

```yaml
# Explicit projection with custom audience and expiry
spec:
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          audience: "https://kubernetes.default.svc"
          expirationSeconds: 3600
          path: token
  containers:
  - name: app
    volumeMounts:
    - name: sa-token
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
```

---

## The two rules every pod should follow

1. **If the pod doesn't talk to the K8s API, set `automountServiceAccountToken: false`.** Most application pods (web servers, workers) never call `kubectl` or the API. Mounting a token they don't use is unnecessary attack surface.

2. **If the pod does talk to the K8s API, give it a named ServiceAccount with a narrow RBAC Role.** Never let it run as `default` with over-broad permissions.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What's a ServiceAccount? | A K8s identity for a pod. Every pod has one — either `default` or explicitly named. |
| What's the token? | A JWT the pod uses to authenticate to the K8s API server. |
| What changed in K8s 1.21? | Tokens are now short-lived (1h default) and auto-rotated by kubelet. The old tokens were long-lived Secrets. |
| What's `automountServiceAccountToken: false`? | Tells K8s not to mount the token into the pod — safe default for pods that don't need API access. |
| What's IRSA? | In EKS: a ServiceAccount annotated with an IAM role ARN. The bound token is exchanged for an AWS IAM credential. Pods get AWS permissions without storing AWS keys. |
| What happens to the token if the pod is deleted? | The bound token is invalidated immediately — another advantage over the old long-lived tokens. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through how a ServiceAccount token gets into a pod, and what changed with bound-token projection in K8s 1.21+."**

**Reference answer (intuitive version):**

"The pod specifies a ServiceAccount name (or uses `default`). When kubelet starts the pod, it requests a token from the API server via the TokenRequest API — a short-lived JWT bound to the pod's UID and expiring in 1 hour. Kubelet mounts this into the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token` as a projected volume. Before 1.21, the token came from a long-lived Secret and never rotated — an attacker who got it owned that ServiceAccount forever. Now the token expires and is rotated automatically by kubelet. The pod code doesn't change — it reads the same file path — but the underlying credential is much safer."

---

## Further reading

- [ServiceAccount token projection docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [BoundServiceAccountTokenVolume KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1205-bound-service-account-tokens)

---

## Next: the deep-dive

When the hotel analogy feels obvious, jump to [`serviceaccounts.md`](./serviceaccounts.md). The deep-dive covers:

- The legacy token → projected token migration in detail
- The TokenRequest API and what kubelet does under the hood
- IRSA (EKS) and Workload Identity (GKE) — how bound tokens enable cloud IAM
- The `automountServiceAccountToken` pattern per-pod and per-SA
- 4 self-test drills + 4-dimensions framing
