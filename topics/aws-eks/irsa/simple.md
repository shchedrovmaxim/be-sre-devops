# IRSA — the simple version (the concert ticket)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

One idea:

> **IRSA is a concert ticket system. The pod holds a signed ticket (a JWT) from a trusted venue (the EKS OIDC provider). AWS is the bouncer — it checks the ticket is genuinely from that venue, then lets the pod in.**

No IAM user. No long-lived credential. The ticket expires. The pod gets a fresh one automatically.

---

## The concert ticket analogy

| Concert world | IRSA world |
|---|---|
| Venue (ticketing authority) | EKS OIDC provider URL (`https://oidc.eks.region.amazonaws.com/id/XXXX`) |
| Signed ticket (barcode) | Projected service account token (JWT at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`) |
| Ticket says "seat B22, valid until 10pm" | JWT says "ServiceAccount `my-sa` in namespace `default`, expires in 1 hour" |
| The bouncer checks the ticket against the venue's list | AWS STS calls the OIDC provider to verify the JWT signature |
| Once verified, the bouncer lets you in | STS returns temporary AWS credentials |
| You can only go to your section (seat B22) | The IAM role's policy limits what AWS resources the pod can access |
| Ticket auto-renews (venue app refreshes it) | kubelet rotates the projected token every 1 hour |

The key shift from old-style IAM: **no static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`** baked into the pod or Kubernetes secret. The credential is ephemeral, rotated automatically, and scoped to one ServiceAccount.

---

## The 2 concepts that confuse people

### 1. What is the OIDC provider, and why does it need to be registered in IAM?

OIDC (OpenID Connect) is just a standard for "here is a signed JWT, and here is the URL where you can verify the signature." Every EKS cluster gets a unique OIDC issuer URL like `https://oidc.eks.us-east-1.amazonaws.com/id/AABBCC1122`.

When you register this URL as an IAM Identity Provider, you're telling AWS: "trust tokens signed by this URL." Without registration, STS refuses to validate any token from the cluster.

```bash
# One-time setup per cluster:
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve
# This creates the IAM OIDC Provider in your account
```

### 2. How does the pod know to use IRSA?

Two pieces must be in place:

**On the ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

**On the IAM role's trust policy:**
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/AABBCC1122"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.us-east-1.amazonaws.com/id/AABBCC1122:sub": "system:serviceaccount:my-app:my-app-sa",
      "oidc.eks.us-east-1.amazonaws.com/id/AABBCC1122:aud": "sts.amazonaws.com"
    }
  }
}
```

The EKS pod identity mutating webhook then injects two env vars into any pod using this ServiceAccount:
- `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
- `AWS_ROLE_ARN=arn:aws:iam::123456789012:role/my-app-role`

The AWS SDK (v2 for Go, boto3, etc.) has a default credential chain that automatically detects these env vars and calls `sts:AssumeRoleWithWebIdentity` for you. Your application code doesn't change.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What replaces the IAM user / access key? | A projected JWT token that the kubelet rotates every hour |
| Where does the SDK find the token? | `$AWS_WEB_IDENTITY_TOKEN_FILE` env var pointing to the projected volume |
| Who issues the token? | The EKS control plane (Kubernetes API server signs it with the cluster's OIDC key) |
| Who verifies the token? | AWS STS, by calling the cluster's OIDC provider URL to get the public key |
| What does the trust policy's `sub` condition do? | Scopes the IAM role to exactly one ServiceAccount in one namespace |
| What if the clock skews >5 minutes? | STS rejects the token (`TokenExpiredException`) — this is a real production failure mode |
| What's the token TTL? | Default 1 hour; kubelet auto-rotates before expiry; configurable per projected volume |
| Do I need to change application code? | No — the AWS SDK default credential chain picks up the env vars automatically |

---

## Self-test (the killer interview question)

Out loud:

> **"Walk me through what happens when a pod with an IRSA-annotated ServiceAccount calls `aws s3 ls`."**

**Reference answer (intuitive version):**

"First, some setup must be in place: the EKS cluster's OIDC provider URL has been registered as an IAM Identity Provider in the account, and the IAM role has a trust policy that allows `sts:AssumeRoleWithWebIdentity` from that OIDC provider — scoped to the specific `sub` claim matching the ServiceAccount's namespace and name.

When the pod starts, the EKS pod identity webhook (a mutating admission webhook) sees that the pod's ServiceAccount has the `eks.amazonaws.com/role-arn` annotation. It injects two env vars: `AWS_WEB_IDENTITY_TOKEN_FILE` pointing to a projected volume, and `AWS_ROLE_ARN` with the IAM role ARN. Kubernetes mounts the projected token at that path — it's a short-lived JWT signed by the cluster's OIDC private key.

When the app calls `aws s3 ls`, the AWS SDK walks its default credential chain, finds both env vars, reads the JWT from disk, and calls `sts:AssumeRoleWithWebIdentity` with the token and the role ARN.

STS receives the call, sees the JWT's issuer matches a registered OIDC provider, fetches the OIDC provider's public key (via the well-known endpoint), verifies the JWT signature, checks that the audience (`aud`) and subject (`sub`) claims match the trust policy's conditions, and — if all checks pass — returns temporary AWS credentials (access key, secret key, session token, expiry).

The SDK caches these credentials and uses them for the `s3:ListBuckets` call. The credentials expire in 1 hour by default. The kubelet has already rotated the JWT before then, so the next SDK call gets a fresh set."

---

## Further reading / watching

- **AWS docs — "IAM roles for service accounts"**: search "EKS IRSA" on docs.aws.amazon.com — the official walkthrough with the `eksctl` commands.
- **AWS blog — "Diving into IAM Roles for Service Accounts"**: search "AWS IRSA diving deep blog" — explains the webhook injection and credential chain in detail.
- **EKS Best Practices Guide — Security**: [aws.github.io/aws-eks-best-practices/security/docs/iam/](https://aws.github.io/aws-eks-best-practices/security/docs/iam/) — covers IRSA alongside other IAM patterns.
- **EKS Pod Identity** (newer alternative to IRSA): AWS 2023 launch; simpler setup (no per-role trust policy edit). Worth knowing both exist. Search "EKS Pod Identity vs IRSA" for the comparison.

---

## Next: the deep-dive

When the concert ticket analogy and the "env vars → SDK → STS" flow feel obvious, jump to [`irsa.md`](./deep-dive.md). The deep-dive covers:

- The full token exchange flow with actual JWT claims
- The OIDC well-known endpoint mechanics
- The trust policy conditions in detail (`sub`, `aud`, `iss`)
- Failure modes: clock skew, missing OIDC provider, wrong audience, missing webhook
- The projected token lifecycle and rotation
- EKS Pod Identity comparison (the newer model)
- 4-dimensions framing for interview
