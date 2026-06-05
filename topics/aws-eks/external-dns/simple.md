# ExternalDNS — the simple version (the hotel concierge analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) will be obvious.

This doc only explains **one idea**:

> **ExternalDNS watches your Kubernetes Services and Ingresses, and automatically keeps your DNS provider in sync — so you don't manually manage Route 53 records when pods move, load balancers rotate, or clusters scale.**

That's it. Everything else (TXT ownership records, `--policy` modes, IRSA, multi-cluster gotchas) is precision on top.

---

## The hotel concierge analogy

You're checking into a hotel. Your room number is 412. The concierge updates the front-desk board: "Guest Max → Room 412."

Then you move to Room 507. The concierge updates the board: "Guest Max → Room 507."

You never have to tell arriving guests your room number. They ask the concierge. The concierge always has the current answer.

| In the hotel world | In the Kubernetes world |
|---|---|
| Guest's room number | The load balancer IP / hostname assigned to a Service |
| The front-desk board | Route 53 (or Cloudflare, etc.) |
| The concierge updating the board | ExternalDNS controller |
| Guests asking for your room | Users resolving `api.example.com` |
| Room 412 → Room 507 move | New ALB provisioned (different hostname) after deploy |

Without ExternalDNS, when your ALB hostname changes, you'd have to go into Route 53 and update the CNAME by hand. ExternalDNS is the concierge that does that automatically.

---

## The one-sentence mental model

> **ExternalDNS reads a `hostname` annotation off your Service or Ingress, and upserts a matching record in Route 53.**

That's the whole loop:

1. You create an Ingress with `external-dns.alpha.kubernetes.io/hostname: api.example.com`.
2. ExternalDNS sees it.
3. ExternalDNS calls Route 53 API: create (or update) a CNAME record for `api.example.com` pointing at the ALB hostname.
4. ExternalDNS also creates a TXT record as an ownership marker (so it knows it manages this record, not some other tool or human).
5. Done. DNS works.

When you delete the Ingress, ExternalDNS (in `sync` mode) deletes the record too.

---

## The 3 things you need to understand

### 1. What ExternalDNS watches

It watches Kubernetes resources for hostname annotations or spec fields:

- **Services** (`type: LoadBalancer`) — looks at the annotation `external-dns.alpha.kubernetes.io/hostname`
- **Ingresses** — reads the `spec.rules[].host` field automatically
- **HTTPRoutes** (Gateway API) — newer, requires the right source arg

### 2. IRSA — how it talks to AWS

ExternalDNS needs permission to change Route 53 records. In EKS, you give it a service account with an IAM role attached (IRSA). The IAM policy needs:

```json
{
  "Effect": "Allow",
  "Action": ["route53:ChangeResourceRecordSets", "route53:ListHostedZones",
             "route53:ListResourceRecordSets"],
  "Resource": ["arn:aws:route53:::hostedzone/<ZONE_ID>"]
}
```

No long-lived credentials — just a short-lived token exchanged by the OIDC provider.

### 3. The TXT ownership record — why it exists

ExternalDNS creates a TXT record alongside every DNS record it manages. Something like:

```
"heritage=external-dns,external-dns/owner=my-cluster,external-dns/resource=ingress/default/my-ingress"
```

This is the ownership marker. ExternalDNS won't touch records that don't have its TXT marker — so it won't accidentally delete a record a human created by hand. And in a multi-cluster setup, each cluster uses a different `--owner-id` so they don't fight over the same records.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does ExternalDNS actually do? | Watches K8s resources, calls DNS provider API to create/update/delete records |
| What triggers it? | A hostname annotation on a Service, or `spec.rules[].host` on an Ingress |
| How does it authenticate to Route 53? | IRSA (IAM Role for Service Account via OIDC token) |
| What's the TXT record for? | Ownership marker — prevents ExternalDNS from touching records it didn't create |
| What's `--policy sync`? | It will delete DNS records when the K8s resource disappears |
| What's `--policy upsert-only`? | It creates/updates but never deletes — safer for shared zones |
| Multi-cluster risk? | Two clusters with the same `--owner-id` fight over records. Use unique owner IDs per cluster. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through how ExternalDNS gets a DNS record into Route 53 when I create a Service or Ingress."**

**Reference answer (intuitive version):**

"You create an Ingress with `spec.rules[].host: api.example.com` and annotate it. ExternalDNS, running as a Deployment in the cluster, watches the K8s API for changes to Services and Ingresses. When it sees the new Ingress, it reads the hostname, then calls the Route 53 API to upsert a CNAME (or ALIAS) record pointing to the ALB hostname. It also writes a TXT ownership record so it knows it manages this entry. ExternalDNS has an IRSA service account with a scoped Route 53 IAM policy — no static credentials. On next reconcile (default every minute), if anything changes — the ALB hostname rotates, or you delete the Ingress — ExternalDNS updates or removes the Route 53 record. In `sync` mode it will delete; in `upsert-only` mode it won't delete, just update."

If that came out clearly, jump to the [deep-dive](./deep-dive.md) for TXT record internals, `--domain-filter`, the multi-cluster owner ID pattern, cert-manager integration, and `--policy create-only`.

---

## Further reading / watching

- **ExternalDNS docs**: [kubernetes-sigs.github.io/external-dns](https://kubernetes-sigs.github.io/external-dns/latest/) — the Route 53 tutorial is the fastest hands-on start
- **ExternalDNS GitHub** — the Tutorials/ folder has per-provider setup guides
- **AWS blog — "ExternalDNS for Amazon EKS"** — search the title; the AWS blog post explains IRSA setup end-to-end

---

## Next: the deep-dive

When the concierge analogy and the annotation-to-Route53 loop feel obvious, jump to [`external-dns.md`](./deep-dive.md). The deep-dive covers:

- The full reconciliation loop and polling interval
- TXT record format — exactly what goes in the string and why
- `--policy` modes in depth (sync vs upsert-only vs create-only)
- `--domain-filter` and `--zone-id-filter` for scoping to specific hosted zones
- Multi-cluster ownership: the `--owner-id` mechanism and why you need unique IDs
- IRSA policy — the exact IAM actions needed
- cert-manager integration pattern
- The Gateway API HTTPRoute source
- Self-test drills and the 4-dimensions framing
