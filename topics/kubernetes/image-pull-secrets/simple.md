# Image pull secrets + private registries — the simple version (the membership card analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **To pull from a private registry, kubelet needs credentials. There are three ways to provide them: a Secret, a cloud-native token exchange, or a kubelet credential provider plugin. Each has different security trade-offs.**

---

## The gym membership card

Imagine pulling a Docker image is like entering a gym. A private registry is a members-only gym.

| Gym world | K8s world |
|---|---|
| Membership card in your wallet | `imagePullSecrets` — a dockerconfigjson Secret in the cluster |
| The gym checks the card | kubelet presents the credentials to the registry |
| Corporate gym: your work badge opens the door automatically | Cloud-native auth: IRSA/Workload Identity exchanges your identity for registry access |
| The gym kiosk has a master key that auto-refreshes | kubelet credential provider plugin — auto-refreshing credentials, no Secrets stored |

---

## The three approaches

### 1. dockerconfigjson Secret (the straightforward way)

You create a Secret in the cluster holding registry credentials:

```yaml
# Create the secret
kubectl create secret docker-registry my-registry-creds \
  --docker-server=my-registry.example.com \
  --docker-username=robot \
  --docker-password=hunter2 \
  --namespace=payments

# Reference it in the pod
spec:
  imagePullSecrets:
  - name: my-registry-creds
  containers:
  - name: api
    image: my-registry.example.com/payments-api:v1.0
```

Or attach it to the ServiceAccount so every pod using that SA gets it automatically:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-sa
imagePullSecrets:
- name: my-registry-creds
```

**Trade-off**: simple to set up. But the credential is long-lived and stored in etcd. If etcd is compromised, so are your registry credentials.

### 2. Cloud-native token exchange (the modern way for cloud clusters)

On EKS + ECR, you don't store credentials at all. The node's IAM role has permission to pull from ECR. kubelet calls ECR's `GetAuthorizationToken` API automatically:

```
Node instance role → has ecr:GetAuthorizationToken permission
                   → kubelet gets a 12-hour ECR token
                   → token used to pull the image
```

No Secrets in the cluster. Credentials auto-rotate. This is the right answer for AWS/GKE/AKS workloads.

### 3. Kubelet credential provider plugin (the general modern way)

A binary on the node that kubelet calls to get credentials for a given registry. The plugin handles token refresh. No Secrets in etcd.

This is what cloud providers implement under the hood for their managed registries.

---

## ImagePullPolicy — the three modes

```yaml
containers:
- name: api
  image: my-registry.example.com/api:v1.2.3
  imagePullPolicy: IfNotPresent   # default for tagged images; pull only if not cached
# or:
  imagePullPolicy: Always         # always pull (re-check digest even if cached)
# or:
  imagePullPolicy: Never          # never pull; fail if not in node cache
```

| Policy | When to use |
|---|---|
| `IfNotPresent` | Default for versioned tags. Good for performance; bad for mutable tags. |
| `Always` | Use with mutable tags like `:latest`. Forces a re-pull (and digest check) every time. |
| `Never` | Air-gapped nodes or pre-pulled images. |

**The `:latest` trap**: using `image: myapp:latest` with `IfNotPresent` means nodes that already have a `:latest` image cached will never pull a newer one. Fix: use immutable tags (`v1.2.3`) and set `IfNotPresent`, or use `:latest` with `Always` and accept the performance cost.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Where is an imagePullSecret stored? | As a K8s Secret of type `kubernetes.io/dockerconfigjson` in etcd. |
| Do I need pull secrets on EKS + ECR? | No — the node's IAM role handles it via the ECR credential provider. |
| What does `ImagePullPolicy: Always` cost? | A registry API call on every pod start. Can hit registry rate limits under heavy scaling. |
| What's the Docker Hub rate limit gotcha? | Anonymous pulls: 100/6h per IP. Authenticated: 200/6h. Under high scaling, your nodes get throttled. Use a pull-through cache or authenticated pulls. |
| What's a pull-through cache? | A registry (e.g., ECR or a self-hosted Harbor) that mirrors external registries. Pulls hit the cache; cache pulls from Docker Hub once. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through every way K8s can authenticate to a private registry and the trade-offs."**

**Reference answer (intuitive version):**

"Three ways. First, a `dockerconfigjson` Secret referenced via `imagePullSecrets` on the pod or ServiceAccount — simple, works everywhere, but stores credentials long-term in etcd. Second, cloud-native auth: on EKS, the node's IAM role has `ecr:GetAuthorizationToken` and kubelet gets a rotating ECR token automatically — no Secrets in the cluster, credentials auto-rotate, best approach for ECR. Third, kubelet credential provider plugins: a binary on the node that kubelet calls per image pull to get fresh credentials — the general-purpose version of cloud-native auth, works with any registry that has a plugin. The trade-off is operational complexity: you manage the plugin binary and its config on each node."

---

## Further reading

- [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kubelet credential providers](https://kubernetes.io/docs/concepts/containers/images/#kubelet-credential-provider)

---

## Next: the deep-dive

When the membership card analogy feels obvious, jump to [`image-pull-secrets.md`](./deep-dive.md). The deep-dive covers:

- The exact dockerconfigjson format
- Cloud-native auth for ECR, GAR, ACR in detail
- Kubelet credential provider plugin configuration
- Rate limiting, pull-through caches, and image caching
- The security trade-off of `Always` vs immutable tags
- 4 self-test drills + 4-dimensions framing
