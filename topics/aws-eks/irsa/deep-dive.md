# IRSA — IAM Roles for Service Accounts (the deep-dive)

> **Goal**: by the end you can answer **"Walk me through what happens when a pod with an IRSA-annotated ServiceAccount calls `aws s3 ls`."** — naming every hop: projected token, webhook injection, STS AssumeRoleWithWebIdentity, OIDC public key verification, trust policy condition matching, credential caching, and rotation. You can also name the four realistic failure modes.

> Start with the [simple version](./simple.md) if you haven't — the concert ticket analogy is the mental model.

---

## The senior framing — this is about credential federation, not IAM users

Before IRSA, the options for giving a pod AWS access were:
1. **Instance profile on the node** — all pods on the node share the same IAM role. A compromised pod can access any AWS resource the node can. Blast radius = entire node.
2. **Long-lived access key in a K8s Secret** — worse. Long-lived credentials can be exfiltrated, don't rotate automatically, and show up in `kubectl describe pod`.
3. **kube2iam / kiam** — open-source proxies that intercept the EC2 metadata endpoint and return pod-specific credentials. Clever hack; fragile in practice (race conditions, network path dependency, maintenance burden).

IRSA solves this with **credential federation via OIDC**. The pod never holds a long-lived secret. It holds a short-lived JWT signed by the cluster, exchanges it for temporary STS credentials, and the credentials expire automatically. Compromise a pod → attacker gets credentials valid for at most 1 hour, scoped to exactly one IAM role.

---

## Prerequisites — what must be set up once per cluster

### 1. EKS OIDC provider registration

Every EKS cluster has a unique OIDC issuer URL:

```
https://oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF
```

The cluster's control plane (specifically, the API server) signs projected tokens using a private key. The OIDC well-known endpoint at:

```
https://oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF/.well-known/openid-configuration
```

returns the JWKS (JSON Web Key Set) — the public key AWS STS uses to verify tokens.

You register this URL as an IAM OIDC Identity Provider in your account:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-east-1 \
  --approve
```

This is a one-time operation per cluster. Without it, STS has no way to verify any JWT from this cluster, and every `AssumeRoleWithWebIdentity` call fails with `NotAuthorized`.

### 2. IAM role with trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF:sub": "system:serviceaccount:my-namespace:my-app-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Two conditions:
- **`sub`**: the JWT subject claim. For IRSA, this is always `system:serviceaccount:<namespace>:<serviceaccount-name>`. This scopes the role to *one* ServiceAccount in *one* namespace. A pod in `other-namespace` with the same SA name cannot assume this role.
- **`aud`**: the JWT audience claim. IRSA tokens are issued with `aud: sts.amazonaws.com`. This prevents the token from being used against other OIDC relying parties.

### 3. ServiceAccount annotation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
    # Optional — override token expiry (default 86400s = 24h for SA token; IRSA tokens default to 1h):
    eks.amazonaws.com/token-expiration: "3600"
```

---

## The webhook — how env vars and volumes appear in the pod

EKS runs a mutating admission webhook called `pod-identity-webhook` (deployed as part of the EKS managed control plane). When a pod is created:

1. The API server sends the pod spec to the webhook before persisting it.
2. The webhook checks if the pod's ServiceAccount has the `eks.amazonaws.com/role-arn` annotation.
3. If yes, the webhook mutates the pod spec to add:
   - An `emptyDir` volume + `projected` ServiceAccount token volume mounted at `/var/run/secrets/eks.amazonaws.com/serviceaccount/`
   - Two environment variables: `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN`
   - Optionally: `AWS_ROLE_SESSION_NAME` (defaults to the pod name)

The projected token is NOT the default Kubernetes service account token (the one at `/var/run/secrets/kubernetes.io/serviceaccount/token`). It's a separate, audience-scoped token issued specifically for IRSA with a configurable TTL (default 1 hour).

---

## The full token exchange flow

```
Pod startup:
  kubelet mounts projected token at /var/run/secrets/eks.amazonaws.com/serviceaccount/token
  (Token is a JWT with: iss=OIDC_URL, sub=system:serviceaccount:ns:sa, aud=sts.amazonaws.com, exp=+1h)

App runs `aws s3 ls`:
  AWS SDK v2 credential chain:
  1. Check env AWS_ACCESS_KEY_ID → not set
  2. Check ~/.aws/credentials → not present in container
  3. Check AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN → FOUND

  SDK reads token from file.
  SDK calls:
    POST https://sts.amazonaws.com/
    Action=AssumeRoleWithWebIdentity
    &RoleArn=arn:aws:iam::123456789012:role/my-app-role
    &RoleSessionName=my-pod-name
    &WebIdentityToken=<JWT>
    &DurationSeconds=3600

AWS STS:
  1. Decodes the JWT header to find the key ID (kid).
  2. Fetches the OIDC provider's JWKS endpoint (cached, ~15 min TTL).
  3. Verifies the JWT signature against the matching public key.
  4. Checks JWT claims:
     - iss == registered OIDC provider URL ✓
     - aud == sts.amazonaws.com ✓
     - exp > now ✓ (fails if clock skew > 5 min)
  5. Checks trust policy conditions:
     - sub == system:serviceaccount:my-namespace:my-app-sa ✓
     - aud == sts.amazonaws.com ✓
  6. Returns: AccessKeyId, SecretAccessKey, SessionToken, Expiration (+1h).

SDK caches credentials in memory.
SDK calls s3:ListBuckets with the session token.

Before expiry, SDK re-calls AssumeRoleWithWebIdentity with the refreshed JWT
(kubelet has already written a new token to the projected volume).
```

---

## JWT anatomy — what the token actually contains

A decoded IRSA projected token looks like:

```json
{
  "iss": "https://oidc.eks.us-east-1.amazonaws.com/id/AABBCC112233DDEEFF",
  "sub": "system:serviceaccount:my-namespace:my-app-sa",
  "aud": ["sts.amazonaws.com"],
  "exp": 1748000000,
  "iat": 1747996400,
  "kubernetes.io": {
    "namespace": "my-namespace",
    "pod": { "name": "my-app-pod-xyz", "uid": "abc-123" },
    "serviceaccount": { "name": "my-app-sa", "uid": "def-456" }
  }
}
```

The `kubernetes.io` claims are IRSA-specific extensions — STS ignores them, but they're useful for auditing in CloudTrail.

---

## Token rotation

The kubelet mounts the projected token and handles rotation. By default:
- The token is refreshed when 80% of its TTL has elapsed (so for a 1-hour token: rotation at ~48 minutes).
- The file at the projected path is updated in place.
- The AWS SDK polls the file and picks up the new token transparently.

**This means a running pod always has a valid token**, even across multiple hours of runtime. The pod does not need to restart for credential rotation.

---

## Failure modes (the ones that actually happen in production)

### 1. Clock skew > 5 minutes

JWTs include an `exp` claim. STS validates it against its own clock. If the node's system clock is off by more than ~5 minutes, STS returns `TokenExpiredException`. Symptoms: `NoCredentialProviders: no valid providers in chain` or `ExpiredTokenException` in SDK logs.

Fix: ensure `chronyd` / `systemd-timesyncd` is running on nodes. EKS-optimized AMIs do this by default, but custom AMIs sometimes miss it.

### 2. Missing OIDC provider registration

If you create a cluster but forget to run `associate-iam-oidc-provider`, every `AssumeRoleWithWebIdentity` call fails with `InvalidIdentityToken: No OpenIDConnect provider found`. The error shows up in pod logs, not in the pod's status.

Fix: `eksctl utils associate-iam-oidc-provider` (idempotent). Then verify with `aws iam list-open-id-connect-providers`.

### 3. Wrong audience in trust policy

A common copy-paste mistake: the trust policy condition checks `aud == sts.amazonaws.com` but the projected token was issued with a different audience (or vice versa). STS returns `Access denied`.

Less obvious variant: using EKS Pod Identity (newer model) but still writing an IRSA-style trust policy. The Pod Identity model uses a different mechanism entirely.

### 4. Trust policy `sub` claim mismatch

If the ServiceAccount annotation points to role `my-role`, but the trust policy's `sub` condition says `system:serviceaccount:other-namespace:other-sa`, the role won't be assumable. Common when copy-pasting trust policies between roles and forgetting to update the namespace/SA name.

Debug: check CloudTrail for `AssumeRoleWithWebIdentity` events — the event shows the actual `sub` and `aud` claims presented, which you can compare against the trust policy.

### 5. Webhook not running / pod predates webhook

If the pod-identity-webhook is not running (e.g., during a cluster upgrade disruption), pods created during the outage won't have the env vars injected. The SA annotation is there but nothing acted on it. The SDK falls through to the EC2 instance metadata and uses the node's instance profile — potentially with broader permissions.

Detect: `kubectl describe pod <pod>` and check for the absence of `AWS_WEB_IDENTITY_TOKEN_FILE` env var.

---

## EKS Pod Identity — the newer alternative

AWS launched EKS Pod Identity in late 2023. The key difference from IRSA:

| Aspect | IRSA | EKS Pod Identity |
|---|---|---|
| Trust policy model | Per-role, per-SA `StringEquals` condition | Pod Identity Association CRD (or API call) |
| Setup per role | Edit trust policy + annotate SA | Create `PodIdentityAssociation` object |
| Credential injection | Webhook injects env vars | EKS agent DaemonSet handles it |
| OIDC registration | Required (one-time) | Not required |
| Credential endpoint | STS AssumeRoleWithWebIdentity | EKS credential API (different endpoint) |

Pod Identity is simpler to configure at scale (no per-role trust policy edits, no OIDC provider setup). IRSA is still more widely used and has more tooling support. Both are production-valid. Know both exist; expect interviewers to ask which you've used and why.

---

## The interview answer in 60 seconds

> "IRSA is a credential federation mechanism. The pod holds a short-lived JWT — projected into the pod by a mutating webhook — signed by the EKS cluster's OIDC provider. When the app makes an AWS API call, the SDK detects `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` env vars, reads the JWT, and calls `sts:AssumeRoleWithWebIdentity`.
>
> STS fetches the cluster's OIDC public key (from the registered IAM OIDC provider), verifies the JWT signature, then checks the trust policy conditions: the `sub` claim must match the ServiceAccount's namespace and name, and the `aud` claim must be `sts.amazonaws.com`. If all checks pass, STS returns temporary credentials.
>
> No long-lived secret anywhere. The token rotates every hour via kubelet's projected volume machinery. Blast radius of a compromised pod: 1 hour of credentials, scoped to one IAM role.
>
> The four things that break this in production: clock skew over 5 minutes, missing OIDC provider registration, wrong `sub` in the trust policy, and the webhook not injecting the env vars during an upgrade disruption."

---

## Self-test drills

### 1. Walk me through what happens when a pod with an IRSA-annotated ServiceAccount calls `aws s3 ls`.

**Reference answer:** (See the full token exchange flow above.) Key hops: kubelet mounts projected JWT → pod-identity-webhook injects env vars → SDK detects vars → SDK calls STS AssumeRoleWithWebIdentity → STS fetches OIDC public key → STS verifies JWT + trust policy conditions → STS returns temp creds → SDK calls S3.

### 2. What four things can break IRSA in production?

**Reference answer:**
1. Clock skew > 5 min → `TokenExpiredException` from STS.
2. Missing OIDC provider → `InvalidIdentityToken: No OpenIDConnect provider found`.
3. Wrong `sub` / `aud` in trust policy → `Access denied` from STS.
4. Webhook not running → env vars not injected → SDK falls back to node instance profile.

### 3. How does the pod's JWT get rotated, and why doesn't the app care?

**Reference answer:**
- Kubelet refreshes the projected token file when 80% of TTL has elapsed (e.g., at ~48 min for a 1-hour token).
- The file at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` is updated in place.
- The AWS SDK reads the file on each `AssumeRoleWithWebIdentity` call, so it always gets the latest token.
- SDK credential caching means it only calls STS when the current STS credentials are near expiry — not on every API call.
- Net result: zero restarts, zero application code changes, continuous rotation.

### 4. What's the difference between IRSA and EKS Pod Identity?

**Reference answer:**
- IRSA: pod-identity-webhook injects env vars; trust policy on the IAM role uses `StringEquals sub/aud` conditions; requires OIDC provider registration.
- EKS Pod Identity: EKS agent DaemonSet on each node handles credential injection; association is configured via a `PodIdentityAssociation` API call (no trust policy edits per role); no OIDC setup.
- Pod Identity is simpler to manage at scale; IRSA is more mature and widely supported by tooling (Terraform aws provider, eksctl, etc.).
- Both produce short-lived, workload-scoped AWS credentials. The credential delivery mechanism differs, not the security model.

---

## Further reading / watching

- **AWS docs — IRSA walkthrough**: search "EKS IAM roles for service accounts" on docs.aws.amazon.com. The setup steps are authoritative.
- **EKS Best Practices Guide — IAM**: [aws.github.io/aws-eks-best-practices/security/docs/iam/](https://aws.github.io/aws-eks-best-practices/security/docs/iam/) — covers IRSA, Pod Identity, and least-privilege patterns.
- **AWS blog — "Diving into IAM Roles for Service Accounts"**: search exact title. Explains the webhook internals with diagrams.
- **EKS Pod Identity announcement (re:Invent 2023)**: search "EKS Pod Identity re:Invent 2023" on YouTube for the launch demo.
- **CloudTrail for IRSA debugging**: search "EKS IRSA CloudTrail troubleshooting" — the pattern of reading `AssumeRoleWithWebIdentity` events to debug `sub`/`aud` mismatches is documented in several AWS blogs.

---

## The 4 dimensions (senior framing)

- **Tech**: no static credentials. JWT projected into pod. STS token exchange via OIDC federation. 1-hour credential TTL. Trust policy `sub` condition = one SA in one namespace = least-privilege by default. Failure modes are all clock, trust policy, or webhook configuration — not application code.
- **People**: developers don't need AWS access to test IRSA locally. They need the SA annotation and the IAM role ARN — both can be templated in Helm values. On-call runbook: "pod can't call AWS" → check `kubectl describe pod` for `AWS_WEB_IDENTITY_TOKEN_FILE` → check CloudTrail for `AssumeRoleWithWebIdentity` errors → check clock skew on node.
- **CI/CD**: IAM roles and trust policies should be in Terraform. The ServiceAccount annotation goes in Helm values or Kustomize overlays. Never hardcode `AWS_ACCESS_KEY_ID` in a Helm chart — if you see one in a PR, block it. Automated tooling (eksctl, Terraform `aws_eks_pod_identity_association`) manages Pod Identity associations.
- **Operations**: monitor `aws_sts_assume_role_with_web_identity_errors` via CloudWatch or a custom metric. Alert on repeated `TokenExpiredException` — usually signals clock skew. Rotate the OIDC private key annually (EKS handles this automatically for managed clusters). Audit CloudTrail for unexpected `AssumeRoleWithWebIdentity` calls — any call from an unknown `sub` claim is a signal to investigate.
