# Image pull secrets + private registries — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through every way K8s can authenticate to a private registry and the trade-offs."** — naming the three mechanisms, cloud-native auth for ECR/GAR/ACR, rate limiting pitfalls, and the security implications of `Always` vs immutable tags.

> Start with the [simple version](./image-pull-secrets-simple.md) if you haven't read it. The gym membership analogy is the spine.

---

## The senior framing — credentials in etcd are a risk

When you store a `dockerconfigjson` Secret in K8s, that credential lives in etcd in base64-encoded (not encrypted by default) form. Anyone who can read Secrets cluster-wide can exfiltrate registry credentials. In a compromise scenario, the attacker then:
1. Has your registry credentials.
2. Can pull your images (containing your code).
3. Can potentially push malicious images if write permissions are also in the credential.

**Encryption at rest for etcd** (`--encryption-provider-config`) mitigates point 1. But the cleanest solution is not storing credentials in the cluster at all — which is exactly what cloud-native auth and credential provider plugins achieve.

---

## Mechanism 1 — dockerconfigjson Secret

### Creating the Secret

```bash
# Method 1: kubectl create (most common)
kubectl create secret docker-registry ecr-creds \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --namespace=payments

# Method 2: from existing ~/.docker/config.json
kubectl create secret generic ecr-creds \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  --namespace=payments
```

The Secret stores a base64-encoded JSON blob:

```json
{
  "auths": {
    "my-registry.example.com": {
      "username": "robot",
      "password": "hunter2",
      "email": "robot@example.com",
      "auth": "<base64(username:password)>"
    }
  }
}
```

### Using the Secret — per pod

```yaml
spec:
  imagePullSecrets:
  - name: ecr-creds
  containers:
  - name: api
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payments-api:v1.2.3
```

### Using the Secret — via ServiceAccount (better DX)

Attach the pull secret to the ServiceAccount once; all pods using that SA automatically get it:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-sa
  namespace: payments
imagePullSecrets:
- name: ecr-creds
```

Now every pod with `serviceAccountName: payments-sa` can pull from ECR without explicit `imagePullSecrets` in the pod spec.

### The problem with ECR secrets specifically

ECR credentials expire after **12 hours**. If you create the Secret manually, you must refresh it every 12 hours. Many teams run a CronJob that does this:

```yaml
# A CronJob that refreshes the ECR pull secret every 6 hours
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: refresher
            image: amazon/aws-cli:2
            command:
            - /bin/sh
            - -c
            - |
              TOKEN=$(aws ecr get-login-password --region us-east-1)
              kubectl create secret docker-registry ecr-creds \
                --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
                --docker-username=AWS \
                --docker-password="$TOKEN" \
                --dry-run=client -o yaml | kubectl apply -f -
```

This works but is fragile. The better solution for ECR is cloud-native auth.

---

## Mechanism 2 — cloud-native auth (no Secrets in cluster)

### ECR on EKS

The EKS node's instance IAM role (or the managed node group's IAM role) is granted `ecr:GetAuthorizationToken` and `ecr:BatchGetImage` permissions. The kubelet ECR credential provider calls the ECR API to get a 12-hour token, refreshes it automatically, and passes it to containerd for each image pull.

```bash
# Required IAM policy on the node role
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken",
    "ecr:BatchCheckLayerAvailability",
    "ecr:GetDownloadUrlForLayer",
    "ecr:BatchGetImage"
  ],
  "Resource": "*"   # GetAuthorizationToken has no resource scope; others can be scoped to repo ARNs
}
```

No `imagePullSecrets` needed in pod specs. No Secret in etcd. The credential lives only in kubelet's in-memory cache and is auto-rotated.

**For cross-account ECR pulls**: the source node role needs a resource-based policy on the ECR repository in the target account, or the target repo grants `ecr:BatchGetImage` to the source account's role.

### GAR (Google Artifact Registry) on GKE

GKE nodes have a Compute Engine service account. That SA is granted `roles/artifactregistry.reader` on the GAR repository. The same mechanism — node identity → token exchange → image pull. No pull secrets.

```bash
# Grant the node service account access to a repository
gcloud artifacts repositories add-iam-policy-binding my-repo \
  --location=us-central1 \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### ACR (Azure Container Registry) on AKS

AKS with managed identity: the kubelet managed identity is granted `AcrPull` role on the ACR. Same pattern.

```bash
# Attach ACR to an AKS cluster (handles the RBAC assignment)
az aks update --name my-cluster --resource-group my-rg \
  --attach-acr my-registry
```

---

## Mechanism 3 — kubelet credential provider plugins

The general-purpose, pluggable version of cloud-native auth. A binary installed on each node; kubelet calls it when pulling an image from a matching registry.

```yaml
# /etc/kubernetes/kubelet-credential-provider-config.yaml
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
- name: ecr-credential-provider
  matchImages:
  - "*.dkr.ecr.*.amazonaws.com"
  - "*.dkr.ecr-fips.*.amazonaws.com"
  defaultCacheDuration: "12h"
  apiVersion: credentialprovider.kubelet.k8s.io/v1
```

The kubelet flag:

```
--image-credential-provider-config=/etc/kubernetes/kubelet-credential-provider-config.yaml
--image-credential-provider-bin-dir=/usr/local/bin/credential-providers/
```

AWS ships `ecr-credential-provider` as part of the AWS Node Termination Handler or as a standalone binary. EKS uses this internally — it's what makes ECR auth transparent on managed nodes.

**Benefits over dockerconfigjson Secrets**:
- No credentials stored in etcd
- Auto-rotating (the plugin fetches fresh credentials as needed)
- Works for any registry with a plugin (not just cloud registries)

---

## ImagePullPolicy — the security and performance trade-offs

```yaml
containers:
- name: api
  image: my-registry.example.com/api:v1.2.3
  imagePullPolicy: IfNotPresent   # only pull if the exact tag isn't cached
```

| Policy | Behavior | When to use |
|---|---|---|
| `IfNotPresent` | Pull only if not in node cache. Uses cached image for same tag. | Immutable tags (`v1.2.3`). Performance-efficient. |
| `Always` | Always contact the registry to check the digest. Pull if digest changed. | Mutable tags (`:latest`, `:main`). Security-conscious deployments. |
| `Never` | Never pull. Fail if not cached. | Air-gapped environments; pre-seeded node images. |

### The mutable tag trap

```yaml
image: my-api:latest
imagePullPolicy: IfNotPresent   # DANGEROUS with mutable tags
```

Scenario: you push a new `:latest` image. A pod is rescheduled to a node that already has the old `:latest` cached. `IfNotPresent` → node uses the stale image. Your "new" pod is running old code.

Fix: use `imagePullPolicy: Always` with mutable tags, or (better) use immutable tags and bump the tag on every release.

**The performance cost of `Always`**: on pod start, kubelet makes a registry API call to check the manifest digest. If the registry is slow or temporarily down, pod start is delayed or fails. Under high scaling events (HPA spike), this hits the registry hard. Mitigation: pull-through cache.

### The `latest` tag in production = a lie

`:latest` means "the most recent push." Different nodes may have different versions of `:latest` in cache. You lose reproducibility — `kubectl get pod -o yaml` doesn't tell you what code is actually running. Use immutable tags.

---

## Rate limiting — the Docker Hub problem

Docker Hub rate limits anonymous pulls:
- Unauthenticated: **100 pulls per 6 hours per IP**
- Authenticated (free account): **200 pulls per 6 hours per account**
- Docker Pro/Team: higher limits

In a large cluster, all nodes share the same egress IP (or a small pool). 50 pods pulling base images from Docker Hub can hit the limit in minutes. Symptoms: `ImagePullBackOff` with the message `toomanyrequests: You have reached your pull rate limit`.

**Fixes**:
1. **Pull-through cache**: set up an ECR, GCR, or Harbor instance as a mirror. All pulls go to the mirror; it fetches from Docker Hub once and caches.
2. **Authenticated Docker Hub pulls**: create a Docker Hub account with higher limits; store credentials in an `imagePullSecret`.
3. **Copy images to your own registry**: during CI, `docker pull → docker tag → docker push` all base images to your private registry. Never pull from Docker Hub in production.

```yaml
# ECR pull-through cache configuration (AWS console or Terraform)
# Then update image references:
image: 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/library/nginx:1.25
#       ^^^^^^^^ your ECR                          ^^^^^^^^ the mirror prefix
```

---

## Image caching on nodes

Images pulled by kubelet are cached on the node's disk (managed by containerd or Docker). The cache is persistent across pod restarts but ephemeral if the node is replaced.

**Gotcha with node auto-scaling (Karpenter/CA)**: new nodes have empty image caches. The first pod on a new node always pulls the full image. Under a spike, you get: HPA triggers → Karpenter provisions new node → node's first pod pulls image → registry is hit by all new nodes simultaneously. This is the "pull storm" problem.

**Mitigations**:
- **Image pre-warming**: a DaemonSet that pulls common images on new nodes before workloads land.
- **Smaller images**: multi-stage builds, distroless bases — less data to pull.
- **Registry in-region**: ECR in `us-east-1` for nodes in `us-east-1` → no data egress fees, faster pulls.

---

## The interview answer in 60 seconds

> "Three ways K8s can authenticate to a private registry. One: a `dockerconfigjson` Secret referenced via `imagePullSecrets` — credentials stored in etcd, simple to set up, but long-lived and needs rotation for short-expiry registries like ECR (12-hour tokens). Two: cloud-native auth — on EKS, the node's IAM role has `ecr:GetAuthorizationToken`; the ECR credential provider plugin auto-fetches and rotates tokens; no Secrets in the cluster. Same idea on GKE with Workload Identity → GAR and on AKS with managed identity → ACR. Three: kubelet credential provider plugins — the general mechanism behind cloud-native auth; a binary on each node that kubelet calls per image pull to get fresh credentials.
>
> On ImagePullPolicy: `IfNotPresent` is efficient but dangerous with mutable tags — if the tag is already cached, new pushes aren't picked up. `Always` forces a registry check on every pod start, which is safer but adds latency and registry load. Solution: use immutable tags, `IfNotPresent`, and set up a pull-through cache to handle Docker Hub rate limits and absorb new-node pull storms."

---

## Self-test drills

### 1. Walk me through every way K8s can authenticate to a private registry and the trade-offs.

**Reference answer**: (1) dockerconfigjson Secret — simple, but credentials in etcd, needs rotation; (2) cloud-native auth (IRSA/Workload Identity/managed identity) — no credentials stored, auto-rotating, cloud-specific; (3) kubelet credential provider plugins — general-purpose, no Secrets, node-side binary, works for any registry. See full detail above.

### 2. Why is `imagePullPolicy: IfNotPresent` dangerous with mutable tags?

**Reference answer**: if a node already has the tag cached, it won't pull the new image — even if the tag was updated in the registry. The pod runs the old code. Fix: use immutable tags (version + commit hash) with `IfNotPresent`, or use `Always` with mutable tags and accept the performance cost.

### 3. Your cluster is hitting Docker Hub rate limits. What are your options?

**Reference answer**: (1) set up a pull-through cache (ECR, GCR, Harbor) and reroute image references through it; (2) use authenticated Docker Hub pulls via an `imagePullSecret` with a paid account; (3) copy all needed base images to your own private registry during CI and ban direct Docker Hub references in production. Option 3 is the most robust.

### 4. What's a "pull storm" and how do you prevent it?

**Reference answer**: when Karpenter or the cluster autoscaler provisions new nodes simultaneously (e.g., after an HPA spike), all new nodes have empty image caches and pull the full image at once. With a large fleet, this can overwhelm the registry or deplete bandwidth. Mitigations: pre-warming DaemonSet (pulls common images on node init), smaller images (multi-stage builds, distroless), registry in-region (same AZ as nodes), and pull-through cache (only misses hit the origin registry).

---

## Further reading

- [Pull from a private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kubelet credential providers](https://kubernetes.io/docs/concepts/containers/images/#kubelet-credential-provider)
- [ECR credential provider](https://github.com/kubernetes/cloud-provider-aws/tree/master/cmd/ecr-credential-provider)
- [Harbor — self-hosted registry + proxy cache](https://goharbor.io/)

---

## The 4 dimensions (senior framing)

- **Tech**: three mechanisms (dockerconfigjson Secret, cloud-native auth, kubelet credential provider); `ImagePullPolicy: IfNotPresent` with immutable tags is the right default; pull-through cache prevents Docker Hub rate limit issues; IRSA/Workload Identity is the no-credentials-in-cluster gold standard.
- **People**: the "copy to private registry in CI" rule is a team discipline issue. Without a policy enforced in CI, engineers will pull from Docker Hub directly in their Dockerfiles. Add a Kyverno policy that blocks pods pulling images not from your approved registries — this makes it a hard requirement, not a guideline.
- **CI/CD**: your CI pipeline should: build → push to private registry → update image tag in the manifest. Never have prod pull directly from Docker Hub or a public registry. Scan images in CI before push (Trivy, ECR enhanced scanning). Use immutable tags (semver + git SHA). Renovate/Dependabot for base image updates.
- **Operations**: alert on `ImagePullBackOff` events — both as a reliability signal and a security signal (unexpected images being pulled). Monitor registry pull rates (ECR CloudWatch metrics, Docker Hub API limits). Node warm-up time after an auto-scale event is dominated by image pull time — measure it, optimize with smaller images and pre-warming.
