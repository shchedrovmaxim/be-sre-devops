# ExternalDNS for Route 53 — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through how ExternalDNS gets DNS records into Route 53 when I create a Service or Ingress."** — naming the reconciliation loop, TXT ownership records, IRSA policy, `--policy` modes, multi-cluster gotchas, and the cert-manager integration pattern. You can also reason about failure modes and operational knobs.

> Start with the [simple version](./external-dns-simple.md) if you haven't — the hotel concierge analogy is the mental model.

---

## The senior framing — ExternalDNS is a control loop, not a webhook

Mid-level engineers think of ExternalDNS as "a tool that syncs annotations to DNS." Senior engineers understand it as a **Kubernetes controller implementing the standard reconcile loop** against an external API.

That framing matters operationally: it means reconciliation is **eventual**, not immediate. ExternalDNS polls on an interval (`--interval`, default 1 minute). If Route 53 changes are made outside ExternalDNS (human edits, Terraform), ExternalDNS will overwrite them on the next reconcile — or ignore them, depending on the `--policy`.

Understanding ExternalDNS as a controller also clarifies the failure modes: if the controller pod restarts, the DNS state in Route 53 remains intact until the next successful reconcile. The source of truth is **the K8s API**, not the DNS provider.

---

## The reconciliation loop in detail

Every `--interval` seconds (default: 60s), ExternalDNS:

1. **Reads all sources**: iterates over configured source types — `service`, `ingress`, `httproute`, etc. — via the Kubernetes API. For each, extracts the hostname(s) and the target (usually the load balancer's IP or CNAME hostname).
2. **Reads the DNS provider**: calls Route 53 `ListResourceRecordSets` on the configured hosted zones. Builds an in-memory snapshot of current records.
3. **Computes a diff**: compares desired state (from K8s) vs actual state (from Route 53). Groups changes into creates, updates, and deletes.
4. **Applies changes**: calls Route 53 `ChangeResourceRecordSets` in batches. Route 53 change batches are atomic — they either fully apply or fully fail.
5. **Logs results**: every applied change is logged; failures are retried on the next interval.

The ownership filter runs at step 3: ExternalDNS only manages records that carry a matching TXT ownership marker. Records without the marker are ignored entirely.

---

## Source types — what ExternalDNS watches

### Service (type: LoadBalancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.example.com
    external-dns.alpha.kubernetes.io/ttl: "60"          # optional — default 300s
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 443
```

ExternalDNS reads the `hostname` annotation. The target is `status.loadBalancer.ingress[0].hostname` (or `.ip` for NLB with static IPs).

Record type created:
- If the LB has a hostname → `CNAME` record
- If the LB has an IP (NLB with EIP) → `A` record
- AWS ALIAS records require a separate annotation: `external-dns.alpha.kubernetes.io/alias: "true"` → creates an ALIAS record instead of CNAME

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    # No explicit hostname annotation needed — ExternalDNS reads spec.rules[].host
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

ExternalDNS reads `spec.rules[].host` and `spec.tls[].hosts` automatically. No annotation required on the Ingress for the hostname — though you can add the TTL annotation.

### HTTPRoute (Gateway API)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "api.example.com"
  rules: [...]
```

ExternalDNS reads `spec.hostnames`. Requires `--source=httproute` in the ExternalDNS args. The Gateway API source is newer — validate the ExternalDNS version supports it.

---

## IRSA — authentication to Route 53

ExternalDNS needs an IAM role. In EKS, use IRSA:

```yaml
# ServiceAccount for ExternalDNS
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ExternalDNSRole
```

The IAM policy (minimum required):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/Z0123456789ABCDEF"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": ["*"]
    }
  ]
}
```

**Scoping tip**: `ChangeResourceRecordSets` can be scoped to a specific hosted zone. `ListHostedZones` must be `*` — there's no resource-level filter for that action.

**Trust policy**: the role's trust relationship allows `sts:AssumeRoleWithWebIdentity` from the EKS cluster's OIDC provider, constrained to the ExternalDNS service account.

---

## TXT ownership records — the "this is mine" mechanism

When ExternalDNS creates a DNS record, it also creates a TXT record with an ownership string. The TXT record name is `external-dns-<type>-<record-name>.<zone>` and the value is:

```
"heritage=external-dns,external-dns/owner=my-cluster,external-dns/resource=ingress/default/my-app"
```

Three fields:
- `heritage` — always `external-dns` (distinguishes from cert-manager TXT records)
- `owner` — the `--owner-id` argument (defaults to `default` if not set — **always override this**)
- `resource` — the K8s resource that sourced this record (type/namespace/name)

### Why the TXT record is load-bearing

ExternalDNS **only manages records that have its TXT marker.** Anything without the marker is ignored. This means:

- A human-created Route 53 record won't be touched by ExternalDNS.
- A Terraform-managed record won't be touched.
- Two ExternalDNS instances with different `--owner-id` won't interfere with each other.

The corollary: if you delete the TXT record by hand, ExternalDNS loses ownership tracking and will re-create the DNS record fresh on the next reconcile. Not harmful, but creates a brief gap.

### Multi-cluster gotcha

If two clusters share the same hosted zone and both run ExternalDNS with the **same `--owner-id`**, they will conflict:

- Cluster A creates `api.example.com → alb-a.region.elb.amazonaws.com` with TXT owner `my-cluster`
- Cluster B sees the same `api.example.com` target, believes it owns it (same owner ID), and overwrites it with `alb-b.region.elb.amazonaws.com`
- Records flip-flop every minute as each cluster reconciles

**Fix**: every cluster gets a unique `--owner-id`. Common pattern: `--owner-id=cluster-name-env` (e.g., `prod-us-east-1`).

---

## `--policy` modes — sync, upsert-only, create-only

| Mode | Creates | Updates | Deletes | When to use |
|---|---|---|---|---|
| `sync` (default) | Yes | Yes | Yes | When ExternalDNS fully owns the zone or the filtered records |
| `upsert-only` | Yes | Yes | No | Shared zones where you don't want ExternalDNS to delete anything |
| `create-only` | Yes | No | No | Initial migrations; you want to see what it creates before letting it manage updates |

**The `sync` trap**: if you're sharing a hosted zone with records managed by Terraform or other tooling, `sync` will delete any record that ExternalDNS doesn't see a K8s source for — including manually created records that pre-date ExternalDNS. Always use `--domain-filter` and `upsert-only` when first deploying into an existing zone.

---

## Zone filtering — `--domain-filter` and `--zone-id-filter`

In a real account, there are usually many hosted zones. Scoping ExternalDNS prevents it from accidentally touching zones it shouldn't:

```yaml
# In the ExternalDNS Deployment args:
args:
  - --source=service
  - --source=ingress
  - --domain-filter=internal.example.com   # only manage records in this zone
  - --zone-id-filter=Z0123456789ABCDEF     # optional: also filter by zone ID
  - --provider=aws
  - --aws-zone-type=private                # "public" or "private" or "" (both)
  - --policy=upsert-only
  - --owner-id=prod-cluster-use1
  - --txt-owner-id=prod-cluster-use1       # alias for --owner-id in newer versions
  - --interval=1m
```

`--aws-zone-type=private` is critical for internal clusters — without it, ExternalDNS might try to create public DNS records for a private NLB.

---

## The cert-manager integration pattern

ExternalDNS and cert-manager often run together. The typical pattern:

1. cert-manager provisions TLS certificates via Let's Encrypt DNS-01 challenge (needs to write a TXT record to prove domain ownership).
2. ExternalDNS manages the CNAME/A records for the service hostname.

They are **separate concerns** on the same zone. cert-manager uses its own Route 53 credentials (also via IRSA, separate role). ExternalDNS won't interfere with cert-manager's TXT records because cert-manager's TXT records don't have the `heritage=external-dns` marker — ExternalDNS ignores them.

The common confusion: teams think ExternalDNS and cert-manager conflict. They don't — as long as cert-manager has permission to write its `_acme-challenge.` TXT records and ExternalDNS filters by `heritage=external-dns`.

```yaml
# What cert-manager creates (ExternalDNS ignores this):
# _acme-challenge.api.example.com  TXT  "abc123..."

# What ExternalDNS creates:
# api.example.com                  CNAME  alb-xxx.us-east-1.elb.amazonaws.com
# external-dns-cname-api.example.com  TXT  "heritage=external-dns,external-dns/owner=..."
```

---

## Deployment — minimal working example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  replicas: 1      # ExternalDNS is not HA — single replica is the norm
  strategy:
    type: Recreate  # never run two replicas simultaneously; they'll conflict
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      securityContext:
        fsGroup: 65534
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.2
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=example.com
            - --provider=aws
            - --aws-zone-type=public
            - --policy=sync
            - --owner-id=prod-use1
            - --interval=1m
            - --log-level=info
```

**Replica=1 is intentional.** Two ExternalDNS pods race on the same records. They're not leader-elected by default. Use `Recreate` strategy and a single replica.

---

## The interview answer in 60 seconds

> "ExternalDNS is a Kubernetes controller that reconciles K8s resources against a DNS provider on a polling interval — default every minute.
>
> When you create an Ingress with a hostname in `spec.rules[].host`, or a Service with the `external-dns.alpha.kubernetes.io/hostname` annotation, ExternalDNS reads it, determines the target (the ALB or NLB hostname from the Service status), and calls the Route 53 API to create or update the record. It uses IRSA for auth — a service account mapped to an IAM role with `route53:ChangeResourceRecordSets` on the hosted zone.
>
> Alongside every DNS record it manages, ExternalDNS creates a TXT record as an ownership marker. The TXT value includes the `--owner-id` you configure. ExternalDNS only touches records that have its marker — so manually created records, Terraform-managed records, and cert-manager's TXT records are ignored.
>
> The main operational gotcha: in a multi-cluster setup, if two clusters share the same `--owner-id`, they'll overwrite each other's records every minute. Use unique owner IDs per cluster. And in shared zones, use `--policy=upsert-only` instead of `sync` to prevent ExternalDNS from deleting records it doesn't own."

---

## Self-test drills

### 1. Walk me through how ExternalDNS gets a DNS record into Route 53 when I create a Service or Ingress.

**Reference answer:**
- ExternalDNS polls the K8s API every `--interval` (default 1 min) for Services and Ingresses.
- For a Service (`type: LoadBalancer`), it reads the `external-dns.alpha.kubernetes.io/hostname` annotation and `status.loadBalancer.ingress[0].hostname`.
- For an Ingress, it reads `spec.rules[].host` directly.
- It calls `ListResourceRecordSets` on the filtered Route 53 zone, diffs desired vs actual, and calls `ChangeResourceRecordSets` to apply.
- Alongside the DNS record, it creates a TXT ownership record with `heritage=external-dns,external-dns/owner=<owner-id>,...`.
- Auth is via IRSA — ServiceAccount annotated with an IAM role ARN; the token is exchanged via the OIDC provider.
- Deletes: in `sync` mode, when the K8s resource disappears, ExternalDNS deletes the DNS record on the next reconcile. In `upsert-only`, it never deletes.

### 2. What's the multi-cluster owner ID problem, and how do you fix it?

**Reference answer:**
- If two clusters run ExternalDNS with the same `--owner-id` pointing at the same hosted zone, they treat each other's TXT records as "theirs" and overwrite the DNS records on every reconcile cycle. Records flip-flop.
- Fix: unique `--owner-id` per cluster (e.g., `cluster-name-region-env`). Each cluster only manages records where the TXT owner matches its own ID. Records created by another cluster are invisible to it.
- Secondary fix: if clusters serve different subdomains, use `--domain-filter` to scope each ExternalDNS to the subdomain its cluster owns.

### 3. Why does ExternalDNS create TXT records and what happens if you delete them?

**Reference answer:**
- TXT records are the ownership marker. Without them, ExternalDNS can't distinguish a record it created from a record a human or Terraform created. The marker prevents unintended deletion of externally-managed records.
- If a TXT record is deleted, ExternalDNS loses ownership tracking for that record. On the next reconcile, it will see the DNS record exists (no TXT), conclude it didn't create it, and — in `sync` mode — delete it (since no K8s source matches it anymore). The deletion then triggers ExternalDNS to re-create both the DNS record and the TXT on the following reconcile. Brief DNS gap possible.
- In practice: don't delete TXT records manually. Use `--policy=upsert-only` in zones where other tooling co-exists.

### 4. How does ExternalDNS interact with cert-manager in the same cluster?

**Reference answer:**
- cert-manager handles TLS certificate issuance (DNS-01 ACME challenge: writes `_acme-challenge.` TXT records).
- ExternalDNS handles DNS routing (CNAME/A records for service hostnames).
- They don't conflict: cert-manager's TXT records don't carry `heritage=external-dns`, so ExternalDNS ignores them. Both need separate IRSA roles with Route 53 permissions.
- The typical production pattern: ExternalDNS creates `api.example.com → ALB`, cert-manager issues the TLS cert for `api.example.com`, the Ingress uses the cert. ExternalDNS and cert-manager are independent — neither depends on the other's records.

---

## Further reading / watching

- **ExternalDNS docs**: [kubernetes-sigs.github.io/external-dns](https://kubernetes-sigs.github.io/external-dns/latest/) — the Route 53 provider tutorial is the fastest hands-on start
- **ExternalDNS GitHub — Tutorials/**: the `aws/` folder has step-by-step IRSA setup
- **ExternalDNS GitHub — `--policy` docs**: in `docs/proposal/design.md` — explains the ownership model in detail
- **AWS blog — "Setting up ExternalDNS for Amazon EKS"** — covers IRSA trust policy and IAM scope
- **cert-manager + ExternalDNS pattern**: cert-manager docs section on "DNS01 Challenge" describes the two-tool split

---

## The 4 dimensions (senior framing)

- **Tech**: ExternalDNS is a single-replica controller (not HA) running a polling loop. Records are eventually consistent (up to 1 min lag plus Route 53 propagation ~30-60s). For multi-cluster, unique owner IDs are mandatory. Use `--domain-filter` to scope the zone and `--aws-zone-type` to pick public vs private. IRSA with a scoped IAM policy — `ChangeResourceRecordSets` only on the target zone.

- **People**: ExternalDNS is low-maintenance once running, but the TXT ownership mechanism confuses engineers who see unexpected TXT records in Route 53. Document the pattern in your runbook. New engineers need to understand: "to get DNS, add the annotation to your Ingress" — no Route 53 console access needed. Make that the mental model from day one.

- **CI/CD**: ExternalDNS is deployed via Helm (chart: `external-dns/external-dns`). Pin the chart version, use Renovate for upgrades. IRSA role ARN and `--owner-id` should be in values files (per-cluster, per-environment). Test in staging before prod — a misconfigured `--domain-filter` in prod could leave records unmanaged silently. GitOps (ArgoCD) manages the Deployment; IAM role managed by Terraform.

- **Operations**: monitor ExternalDNS with its own Prometheus metrics (`external_dns_registry_endpoints_total`, `external_dns_source_endpoints_total`, `external_dns_controller_last_sync_timestamp_seconds`). Alert on `time() - external_dns_controller_last_sync_timestamp_seconds > 300` (reconcile hasn't run in 5 min). Runbook for "DNS record not created": check pod logs for `WARN` or `ERROR`, verify IRSA role binding with `kubectl describe pod -n kube-system external-dns-*`, confirm the Ingress annotation or host field is populated. Route 53 record creation failure most commonly due to IAM permission gap or zone ID mismatch.
