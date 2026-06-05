# ServiceAccounts + token projection — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through how a ServiceAccount token gets into a pod, and what changed with bound-token projection in K8s 1.21+."** — naming the TokenRequest API, the projected volume, the audience binding, and the relationship to IRSA.

> Start with the [simple version](./simple.md) if you haven't read it. The hotel key card analogy is the spine of this whole topic.

---

## The senior framing — credentials in the pod are a supply-chain risk

Every pod token is a credential. If the pod is compromised, so is the token. The question is: what can an attacker do with it, and for how long?

- **Legacy long-lived token**: stolen once, valid forever. Scope = whatever RBAC says. An attacker who exfiltrated it in 2022 can still use it in 2025.
- **Bound token (1.21+)**: stolen once, valid for at most 1 hour. Scope = same RBAC, but also audience-bound (only valid for the K8s API server, not for arbitrary services). An attacker who exfiltrated it has a 1-hour window.

This is the same trade-off as session cookies vs short-lived JWTs in web auth. The short-lived version is not impenetrable — it just dramatically shrinks the blast radius of a credential leak.

---

## Legacy tokens — what you're migrating away from

Before K8s 1.24, creating a ServiceAccount automatically created a corresponding Secret with a long-lived, audience-unbound token:

```yaml
# Auto-created Secret (K8s < 1.24 behavior — do not create these manually anymore)
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: my-sa-token-xxxxx
  annotations:
    kubernetes.io/service-account.name: my-sa
data:
  token: <base64 JWT>
  ca.crt: <base64>
  namespace: <base64>
```

Properties of the legacy token:
- **Never expires** — no `exp` claim. Valid until the Secret is deleted.
- **Audience-unbound** — any service can verify it; it's not restricted to the K8s API.
- **Static** — stored in etcd. If etcd is compromised, all tokens are compromised.
- **Survives pod deletion** — the Secret outlives the pod. If you forget to delete the Secret, the token remains valid.

As of K8s 1.24, auto-creation of token Secrets is disabled. As of 1.29, the projected volume mechanism is the only default.

**When you still create token Secrets manually**: for long-running non-pod consumers (CI/CD pipelines, external monitoring agents) that can't use the TokenRequest API. Even then, prefer IRSA/Workload Identity where available.

---

## Bound token projection — the modern default

Since K8s 1.20 (GA in 1.21), the default mount mechanism is the **projected ServiceAccount token**:

```yaml
# What kubelet automatically injects (you don't write this yourself)
spec:
  volumes:
  - name: kube-api-access
    projected:
      defaultMode: 0644
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607    # ~1 hour; kubelet refreshes at 80% of expiry
          path: token
      - configMap:
          name: kube-root-ca.crt    # cluster CA cert
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
```

What makes it "bound":

1. **Audience-bound** — the JWT's `aud` claim is set to the K8s API server URL. The API server rejects tokens with a different audience. An attacker can't reuse this token to impersonate the pod to, say, a Vault server.

2. **Pod-bound** — the JWT contains the pod's UID in its claims. If the pod is deleted, the API server's token reviewer knows the binding is gone and rejects the token even before expiry.

3. **Time-limited** — the `exp` claim defaults to 1 hour. Kubelet refreshes it automatically when 80% of the expiry has elapsed (at ~48 minutes). The pod just reads the same file; kubelet updates it in place.

The JWT payload looks like:

```json
{
  "iss": "https://kubernetes.default.svc",
  "sub": "system:serviceaccount:payments:payments-api",
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1700000000,
  "iat": 1699996393,
  "kubernetes.io": {
    "namespace": "payments",
    "pod": {
      "name": "payments-api-6d8f5-xxxxx",
      "uid": "abc123-..."
    },
    "serviceaccount": {
      "name": "payments-api",
      "uid": "def456-..."
    }
  }
}
```

---

## The TokenRequest API — what kubelet actually calls

Kubelet doesn't generate the token itself. It calls the API server's **TokenRequest API**:

```
POST /api/v1/namespaces/{namespace}/serviceaccounts/{name}/token
```

Request body:

```json
{
  "spec": {
    "audiences": ["https://kubernetes.default.svc"],
    "expirationSeconds": 3600,
    "boundObjectRef": {
      "kind": "Pod",
      "apiVersion": "v1",
      "name": "payments-api-6d8f5-xxxxx",
      "uid": "abc123-..."
    }
  }
}
```

The `boundObjectRef` is what makes the token pod-bound. The API server signs the token with the cluster's service account signing key. When the pod later uses the token, the API server's token reviewer:
1. Verifies the signature
2. Checks the audience matches
3. Checks the expiry
4. Checks the bound pod still exists

You can also call TokenRequest yourself — useful for minting custom-audience tokens for IRSA or Vault:

```bash
kubectl create token my-sa \
  --audience=vault \
  --duration=1h \
  --namespace=payments
```

---

## automountServiceAccountToken — the defense-in-depth pattern

Two places to disable token auto-mount:

```yaml
# Option 1: Per-ServiceAccount (applies to all pods using this SA)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: static-webserver
  namespace: frontend
automountServiceAccountToken: false    # all pods using this SA get no token by default
```

```yaml
# Option 2: Per-pod (overrides the SA's setting)
spec:
  serviceAccountName: static-webserver
  automountServiceAccountToken: false  # this pod specifically gets no token
  containers:
  - name: nginx
    image: nginx:1.25
```

Pod-level overrides SA-level. So you can have an SA with `automountServiceAccountToken: false` but a specific pod that sets it to `true` — the pod wins.

**When to use**: any pod that doesn't call the K8s API. Most web servers, workers, and batch jobs have no reason to call `kubectl` or the K8s REST API. Mounting a credential they don't need is unnecessary attack surface. This is a Kyverno-enforceable policy pattern — flag pods that don't set it and have a SA with broad permissions.

---

## Explicit projected volumes — custom audiences

When your pod needs to authenticate to something *other* than the K8s API (Vault, a custom OIDC provider), you can request a token with a custom audience:

```yaml
spec:
  serviceAccountName: payments-api
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          audience: "https://vault.example.com"    # Vault checks this
          expirationSeconds: 3600
          path: token
  containers:
  - name: app
    volumeMounts:
    - name: vault-token
      mountPath: /var/run/secrets/vault
      readOnly: true
```

Vault is configured to trust tokens from your K8s OIDC issuer with `aud=https://vault.example.com`. The pod presents this token to Vault via Vault's K8s auth method. No static Vault token ever lives in the pod.

---

## IRSA (EKS) and Workload Identity (GKE) — cloud IAM via bound tokens

The killer application of bound token projection. Instead of storing AWS/GCP credentials in the pod, you exchange the K8s token for a cloud IAM credential.

### IRSA (EKS — IAM Roles for Service Accounts)

```yaml
# ServiceAccount annotated with an IAM role ARN
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: payments
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/payments-s3-reader"
```

What happens at runtime:
1. Kubelet projects a token with `audience=sts.amazonaws.com` into the pod.
2. The AWS SDK in the pod calls STS's `AssumeRoleWithWebIdentity` with this token.
3. STS verifies the token against the EKS OIDC issuer URL.
4. STS returns short-lived AWS credentials (15min default).
5. The pod uses those credentials to call S3, DynamoDB, etc.

No AWS access key or secret key ever enters the pod. The K8s token is the credential bootstrap mechanism.

```bash
# Verify IRSA is working
kubectl exec -n payments <pod> -- \
  aws sts get-caller-identity
# Should return the ARN of the assumed role, not the node's instance role
```

**Common IRSA gotcha**: the OIDC provider URL must be registered in your AWS account under IAM → Identity Providers. If you create a new EKS cluster and forget to register it, IRSA silently falls back to the node instance role — and your pods get much broader permissions than intended.

### GKE Workload Identity

Same pattern, GCP-flavored:

```yaml
# K8s SA annotated with GCP service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bigquery-reader
  namespace: analytics
  annotations:
    iam.gke.io/gcp-service-account: "bigquery-reader@my-project.iam.gserviceaccount.com"
```

The GKE metadata server exchanges the projected K8s token for a GCP access token. Same no-static-credentials security model.

---

## The interview answer in 60 seconds

> "When kubelet starts a pod, it calls the TokenRequest API to get a short-lived JWT — bound to the pod's UID, audience-bound to the K8s API server, and expiring in one hour. Kubelet mounts this into the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token` via a projected volume. Before K8s 1.21, that file contained a long-lived token from an auto-created Secret — it never expired, and stealing it gave you that ServiceAccount's permissions forever. The modern bound token expires and is automatically refreshed by kubelet at 80% of its lifetime, so the pod code doesn't change but the credential is much safer.
>
> In EKS, IRSA extends this further: the ServiceAccount is annotated with an IAM role ARN, and the projected token's audience is set to `sts.amazonaws.com`. The AWS SDK exchanges it for short-lived IAM credentials automatically. No static AWS keys ever enter the pod.
>
> The operational pattern I enforce: `automountServiceAccountToken: false` on any ServiceAccount whose pods don't call the K8s API, and a named ServiceAccount with a narrow RBAC Role on any pod that does. Never run a production pod as `default`."

---

## Self-test drills

### 1. Walk me through how a ServiceAccount token gets into a pod and what changed in K8s 1.21+.

**Reference answer**: kubelet calls TokenRequest API → gets a bound JWT (audience-bound, pod-bound, 1h expiry) → projects it into the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Before 1.21: long-lived token from a Secret, never expired. After 1.21: auto-rotated by kubelet. Pod code unchanged. See full flow above.

### 2. What does "audience-bound" mean for a ServiceAccount token, and why does it matter?

**Reference answer**: the JWT's `aud` claim is set to the K8s API server URL. The API server rejects tokens with a different audience. An attacker can't take a stolen K8s token and use it to authenticate to Vault or another OIDC-trusting service (unless that service is configured to accept K8s tokens, which it shouldn't be by default). It scopes the credential to its intended consumer.

### 3. When would you set `automountServiceAccountToken: false` and where?

**Reference answer**: on any SA (or pod) whose workloads don't call the K8s API — which is most application pods. Set it at the SA level to make it the default for all pods using that SA; pods that do need a token can override it. Also useful as a Kyverno policy: require all SAs used by non-control-plane workloads to have `automountServiceAccountToken: false` and mount tokens explicitly when needed.

### 4. Explain IRSA in one minute.

**Reference answer**: ServiceAccount is annotated with an IAM role ARN. EKS kubelet projects a token with audience `sts.amazonaws.com`. AWS SDK in the pod calls STS `AssumeRoleWithWebIdentity` with this token. STS verifies the token against the EKS OIDC issuer registered in IAM. Returns short-lived AWS credentials. Pod calls AWS services with those credentials. No static AWS keys. Common gotcha: the OIDC issuer URL must be registered in IAM or IRSA silently falls back to the node instance role.

---

## Further reading

- [K8s ServiceAccount docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Bound ServiceAccount Token Volume KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1205-bound-service-account-tokens)
- [TokenRequest API reference](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
- [IRSA EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [GKE Workload Identity docs](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

---

## The 4 dimensions (senior framing)

- **Tech**: bound tokens (audience-bound, pod-bound, 1h expiry, auto-rotated); TokenRequest API; projected volume mechanism; `automountServiceAccountToken: false` pattern; IRSA and Workload Identity as the "no static credentials" pattern; `kubectl create token` for custom-audience tokens.
- **People**: the "default ServiceAccount with broad RBAC" pattern comes from teams that copied a tutorial and never cleaned up. Make the secure default the easy default: Helm chart scaffolding should create a named SA with `automountServiceAccountToken: false` by default; pods that need API access explicitly opt in with a narrow Role.
- **CI/CD**: Kyverno policies: require named (non-default) ServiceAccounts on all production Deployments; flag `automountServiceAccountToken: true` without an explicit narrow RoleBinding; require IRSA annotation on SAs that call AWS APIs (no static credentials). Run periodic audits to find pods still using legacy long-lived token Secrets.
- **Operations**: monitor TokenRequest API errors — a spike means kubelet can't refresh tokens, which means pods will lose API access at expiry. Alert on IRSA STS call failures (the common symptom of a misconfigured OIDC provider). Audit: list all manually-created `kubernetes.io/service-account-token` Secrets in the cluster and sunset them.
