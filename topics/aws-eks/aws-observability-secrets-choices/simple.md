# CloudWatch vs Prometheus, Secrets Manager vs Parameter Store — the simple version (the toolbox analogy)

> Read this first. Once the "right tool for the right job" frame clicks, the [deep-dive doc](./deep-dive.md) will be straightforward.

This doc only explains **one idea**:

> **CloudWatch and Prometheus are not competing tools — they're complementary. Same with Secrets Manager and Parameter Store. The senior skill is knowing which one to reach for and why, not picking a side.**

That's it. Everything else (cost math, PromQL vs CloudWatch Metrics Insights, rotation, IRSA + ESO integration) is precision on top.

---

## The toolbox analogy

You have two toolboxes. One came with the house (free, integrated, always there). One you bought separately (more powerful, more options, costs more to maintain).

| Situation | Which toolbox? |
|---|---|
| Quick check on a leaky pipe | The house toolbox — it's right there |
| Renovating the whole kitchen | The specialist toolbox — worth the setup |

That's CloudWatch vs Prometheus. CloudWatch is the house toolbox: it's already there, AWS services write to it automatically, no setup required. Prometheus is the specialist toolbox: more powerful, but you own it.

Same logic applies to secrets:
- **Parameter Store** = the house toolbox (cheap, simple, built in)
- **Secrets Manager** = the specialist toolbox (auto-rotation, audit, costs more)

---

## CloudWatch vs Prometheus — one-line each

**CloudWatch**: managed, pay-per-metric ingested, native AWS service integration, no PromQL. Your RDS instance, ECS task, and Lambda function write to it automatically with zero configuration.

**Prometheus**: open-source, runs in your cluster, PromQL for powerful queries, low marginal cost per metric once running, but **you operate it**. Scrapes metrics from your app pods via HTTP.

---

## The decision in two questions

**Question 1: Who generates the metric?**

- AWS service (RDS CPU, Lambda errors, ALB request count) → **CloudWatch**. It's already there. You'd have to do extra work to get these into Prometheus.
- Your app pods (request latency histogram, queue depth, custom business metrics) → **Prometheus**. Apps expose `/metrics`, Prometheus scrapes them. CloudWatch needs an agent or EMF to ingest these.

**Question 2: Do you need PromQL?**

- Yes → Prometheus. CloudWatch Metrics Insights is SQL-ish but not nearly as expressive as PromQL for rate(), histogram_quantile(), label-based aggregation.
- No → CloudWatch is fine. Most alarm and dashboard use cases don't need PromQL.

---

## The typical pattern (the "both" answer)

Senior engineers don't pick one. They use both, with one Grafana frontend:

```
AWS services → CloudWatch → Grafana (CloudWatch datasource)
App pods     → Prometheus → Grafana (Prometheus datasource)
```

One Grafana dashboard, two data sources, unified view. No one has to choose between "AWS infra visibility" and "app-level observability."

---

## Secrets Manager vs Parameter Store — one-line each

**Secrets Manager**: automatic rotation (built-in for RDS, Redshift; custom Lambda for others), fine-grained audit log, $0.40/secret/month + $0.05/10k API calls. Built for secrets that need to rotate.

**Parameter Store**: cheaper ($0.05/10k API calls for Standard; SecureString adds small KMS cost), no rotation, two tiers (Standard = free storage; Advanced = $0.05/parameter/month). Built for config and non-rotating secrets.

---

## The decision in two questions

**Question 1: Does this secret need to rotate?**

- Yes (database password, API key you want to rotate on a schedule) → **Secrets Manager**. It has the rotation machinery built in.
- No (a static config value, a fixed API token that rotates manually) → **Parameter Store**. Simpler, cheaper.

**Question 2: How many secrets?**

- 10 secrets → either is fine. Secrets Manager costs $4/month for 10 secrets.
- 1,000 secrets → Secrets Manager is $400/month in storage fees alone. Parameter Store SecureString is much cheaper at scale.

---

## The cost math at a glance

| | Secrets Manager | Parameter Store (SecureString) |
|---|---|---|
| Storage | $0.40/secret/month | Standard: free; Advanced: $0.05/param/month |
| API calls | $0.05/10k | $0.05/10k |
| Rotation | Built-in | Not available |
| Audit log | CloudTrail (free) | CloudTrail (free) |
| 10 secrets, 100k calls | ~$4.50/mo | ~$0.50/mo |
| 1,000 secrets, 1M calls | ~$405/mo | ~$5/mo |

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| CloudWatch or Prometheus? | Both. CloudWatch for AWS-native metrics; Prometheus for app metrics. One Grafana for both. |
| When is CloudWatch enough alone? | Small teams, mostly AWS-managed services, no PromQL need, no custom app metrics. |
| When do you need Prometheus? | PromQL, histogram_quantile, high-cardinality app metrics, custom business metrics from pods. |
| Secrets Manager or Parameter Store? | Rotation needed → Secrets Manager. Static/config → Parameter Store. Many secrets at scale → Parameter Store. |
| SecureString vs Standard? | SecureString: KMS-encrypted (required for actual secrets). Standard: plaintext (fine for non-sensitive config). |
| How does K8s read AWS secrets? | External Secrets Operator (ESO) — reads from either Secrets Manager or Parameter Store and syncs to K8s Secrets. |

---

## Self-test (one question — the killer one)

Out loud:

> **"When would you pick CloudWatch over Prometheus, and Secrets Manager over Parameter Store?"**

**Reference answer (intuitive version):**

"CloudWatch over Prometheus when you're monitoring AWS-managed services — RDS CPU, Lambda throttles, ALB request rates — because those metrics land in CloudWatch automatically. No agent, no scrape config. Also when the team is small and doesn't want to operate a Prometheus stack. The trade-off is no PromQL and higher per-metric cost at scale.

Prometheus over CloudWatch when you have app pods emitting custom metrics — request latency histograms, queue depths, business counters. PromQL is essential for histogram_quantile and multi-label aggregation. Low marginal cost once running. The trade-off is you own the Prometheus stack.

The senior answer is usually both, with Grafana as the unified frontend.

Secrets Manager over Parameter Store when the secret needs automatic rotation — database passwords especially, where you want the password to rotate without a human touching it. Secrets Manager has built-in rotation for RDS. The cost is $0.40/secret/month, which is fine for tens of secrets but expensive for hundreds.

Parameter Store when it's a non-rotating config value or a static API key, or when you have many secrets and the Secrets Manager cost adds up. SecureString encrypts with KMS, so it's secure — just no rotation. In Kubernetes, ESO (External Secrets Operator) syncs either source into a K8s Secret object, so the consuming pod doesn't care which backend is used."

If that came out clearly, jump to the [deep-dive](./deep-dive.md).

---

## Further reading / watching

- **AWS docs — CloudWatch vs Container Insights**: the Container Insights docs explain the agent-based approach for EKS
- **AWS docs — Amazon Managed Prometheus (AMP)**: AWS-managed Prometheus — eliminates the "you operate it" burden while keeping PromQL
- **External Secrets Operator**: [external-secrets.io](https://external-secrets.io/) — the standard K8s pattern for syncing AWS secrets into K8s
- **AWS blog — "Secrets Manager vs Parameter Store"**: search the title — the cost comparison is current

---

## Next: the deep-dive

When the "both, with Grafana" and "rotation = Secrets Manager" frames feel obvious, jump to [`aws-observability-secrets-choices.md`](./deep-dive.md). The deep-dive covers:

- CloudWatch costs at scale (custom metrics are expensive — $0.30/metric/month)
- Prometheus: the scrape model, recording rules, alerting, managed options (AMP)
- The typical production pattern: CloudWatch + Prometheus → Grafana
- Secrets Manager rotation mechanics: built-in Lambda functions, two-phase rotation
- Parameter Store tiers: Standard vs Advanced, and the SecureString vs Standard distinction
- ESO integration story: ExternalSecret CRD, SecretStore, ClusterSecretStore
- Self-test drills and 4-dimensions framing
