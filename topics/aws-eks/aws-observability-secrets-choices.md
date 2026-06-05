# CloudWatch vs Prometheus + Secrets Manager vs Parameter Store — the deep-dive

> **Goal**: by the end you can answer the killer question — **"When would you pick CloudWatch over Prometheus, and Secrets Manager over Parameter Store?"** — with concrete trade-offs, cost math, the complementary-not-competing framing, and the ESO integration story.

> Start with the [simple version](./aws-observability-secrets-choices-simple.md) if you haven't — the toolbox analogy is the mental model.

---

## The senior framing — these are not either/or choices

The most common mistake: framing this as a religious debate. "We're a Prometheus shop." "We use CloudWatch everywhere." Neither is the senior answer.

The senior answer is:
- **CloudWatch + Prometheus are complementary.** CloudWatch owns AWS-native metrics (RDS, Lambda, ALB, EKS control plane). Prometheus owns app-level metrics (pod-emitted, high-cardinality, custom business metrics). Both feed into one Grafana.
- **Secrets Manager + Parameter Store are complementary.** Secrets Manager owns secrets that need automatic rotation (database passwords, API keys with expiry). Parameter Store owns static config values and non-rotating secrets at scale. ESO bridges both into Kubernetes transparently.

Knowing when each is the right choice — and why — is what the interview is testing.

---

## CloudWatch — the AWS-native metrics layer

### What CloudWatch does

CloudWatch is a managed metrics, logs, and alarms service. Every AWS service writes to it automatically:
- RDS: `CPUUtilization`, `DatabaseConnections`, `FreeStorageSpace`, `ReadLatency`
- Lambda: `Invocations`, `Errors`, `Throttles`, `Duration`, `ConcurrentExecutions`
- ALB: `RequestCount`, `HTTPCode_ELB_5XX_Count`, `TargetResponseTime`, `HealthyHostCount`
- EKS control plane: `cluster_failed_node_count`, API server metrics (via Container Insights)

Zero configuration needed for these. They appear in CloudWatch the moment the service exists.

### Custom metrics — where cost matters

Publishing custom metrics from your app to CloudWatch costs **$0.30 per metric per month** (first 10,000 metrics; decreases with scale). If you have 1,000 custom metrics, that's $300/month — before API call costs.

**CloudWatch EMF (Embedded Metrics Format)**: avoids per-metric charges by encoding metrics inside structured log lines. The logs agent parses them and extracts metrics. Costs shift from per-metric to per-GB-ingested (logs pricing). Clever workaround for high-volume custom metrics.

### CloudWatch Metrics Insights (CMI)

CMI is SQL-like query language over CloudWatch metrics:

```sql
SELECT AVG(CPUUtilization)
FROM SCHEMA("AWS/RDS", DBInstanceIdentifier)
GROUP BY DBInstanceIdentifier
ORDER BY AVG() DESC
LIMIT 10
```

Good for dashboards and simple aggregations. Not comparable to PromQL for:
- Rate calculations over arbitrary windows (`rate(http_requests_total[5m])`)
- Histogram percentile calculations (`histogram_quantile(0.99, ...)`)
- Multi-label aggregation and recording rules

If you need those, Prometheus is necessary.

### Container Insights — CloudWatch for EKS

Container Insights is the EKS-specific CloudWatch add-on. It deploys a DaemonSet that collects:
- Per-pod CPU/memory usage
- Per-namespace aggregates
- Node-level disk I/O

```bash
# Enable Container Insights via eksctl
eksctl utils enable-addons --cluster my-cluster --addon aws-cloudwatch-observability
```

The CloudWatch agent DaemonSet sends data to CloudWatch. You pay per ingested metric and per GB of logs.

**The gotcha**: Container Insights generates a lot of metrics (one per pod per metric type). At 100 pods, that's hundreds of metrics per minute. Cost can surprise teams used to "CloudWatch is free."

---

## Prometheus — the app-level metrics layer

### How Prometheus works

Prometheus uses a **pull model**: it scrapes HTTP endpoints every `--scrape-interval` (default 1m). Each app exposes `/metrics` in the Prometheus text exposition format.

```yaml
# PodMonitor — ServiceMonitor is the standard way in kube-prometheus-stack
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

The Prometheus Operator watches `ServiceMonitor` and `PodMonitor` CRDs and automatically updates Prometheus's scrape config.

### Why PromQL is powerful

PromQL operates on time series with labels. What you can't do in CloudWatch:

```promql
# Request rate, per-route, filtered to 5xx, over 5-minute window
rate(http_requests_total{status=~"5.."}[5m])

# p99 latency from histogram buckets
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Error ratio — what fraction of requests are failing
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
```

This power is load-bearing for SLO work. Without histogram_quantile, you can't compute accurate latency percentiles from aggregated data.

### Cost model — Prometheus is low marginal cost

Once the Prometheus stack is running (CPU/memory cost, storage cost, HA cost), each additional metric is near-zero marginal cost. Compare:

| Metric volume | CloudWatch custom metrics/month | Prometheus (self-hosted) |
|---|---|---|
| 100 metrics | $30 | $0 marginal (stack already running) |
| 1,000 metrics | $300 | $0 marginal |
| 100,000 metrics | $30,000 | Prometheus horizontal scaling cost (TSDB storage + compute) |

The break-even is roughly **100-500 custom metrics**. Below that, CloudWatch custom metrics are fine. Above that, Prometheus saves money.

### Managed options — Amazon Managed Prometheus (AMP)

AMP is AWS's managed Prometheus. You don't run the Prometheus server — AWS does. You write remote_write config in your Prometheus (or use the ADOT collector) to send metrics to AMP.

```yaml
# remote_write to AMP (in Prometheus config)
remote_write:
  - url: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write
    sigv4:
      region: us-east-1
    queue_config:
      max_samples_per_send: 1000
```

AMP pricing: $0.10/metric/month for active series (much cheaper than CloudWatch custom metrics). PromQL works natively. The trade-off: you still run a scraping agent in the cluster (Prometheus or ADOT) — AMP only takes over storage and query.

### The typical production pattern

```
AWS services (RDS, Lambda, ALB)
    → CloudWatch
    → Grafana (CloudWatch datasource)

App pods (/metrics endpoint)
    → Prometheus (scrape)
    → Grafana (Prometheus datasource) or → AMP → Grafana

Unified Grafana dashboard: one pane, two data sources
```

This is the senior answer to "CloudWatch vs Prometheus." Not a choice — a composition.

---

## Secrets Manager — secrets that rotate

### What Secrets Manager does

Secrets Manager stores secrets as JSON key-value pairs, encrypted with KMS. Its defining feature: **automatic rotation**.

### Rotation mechanics

Rotation uses a Lambda function that Secrets Manager invokes on a schedule. The Lambda goes through four phases:

1. **createSecret**: generate a new secret value, store it as `AWSPENDING` version
2. **setSecret**: update the target service (e.g., rotate the RDS password)
3. **testSecret**: verify the new secret works (connect to RDS with the new password)
4. **finishSecret**: mark the new version as `AWSCURRENT`, old version as `AWSPREVIOUS`

AWS provides built-in rotation Lambda functions for RDS, Redshift, and DocumentDB. For custom services, you write the Lambda yourself (or use a community template).

**The two-phase rotation trick**: during rotation, both `AWSCURRENT` and `AWSPENDING` are active briefly. This allows rolling deployments to continue reading the old secret while the new one is being tested. Zero-downtime rotation.

```bash
# Enable rotation — RDS password, rotate every 30 days
aws secretsmanager rotate-secret \
  --secret-id "prod/db/password" \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123:function:SecretsManagerRDSRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

### Pricing

- **Storage**: $0.40/secret/month
- **API calls**: $0.05 per 10,000 calls
- **KMS**: $0.03 per 10,000 API calls (for customer-managed KMS key — optional)

For 10 secrets with 100k API calls/month: ~$4.50/month. For 1,000 secrets: ~$400/month in storage alone.

### Audit log

Every `GetSecretValue`, `PutSecretValue`, `RotateSecret` call is logged in CloudTrail automatically. This is the audit story for compliance: "who accessed the prod DB password and when."

---

## Parameter Store — config + non-rotating secrets at scale

### Tiers — Standard vs Advanced

| | Standard | Advanced |
|---|---|---|
| Free storage | Up to 10,000 parameters | No free tier |
| Parameter storage cost | Free | $0.05/param/month |
| API throughput | 40 TPS per account | 1,000 TPS |
| Max value size | 4 KB | 8 KB |
| Parameter policies (TTL, notification) | No | Yes |

For most teams: Standard is fine. Advanced makes sense only at high throughput or when you need parameter expiry policies.

### Parameter types — String vs StringList vs SecureString

- **String**: plaintext. Use for non-sensitive config (feature flag values, region names, log levels).
- **StringList**: comma-separated plaintext. Rarely used.
- **SecureString**: encrypted with KMS. Required for any actual secret (passwords, tokens). Can use AWS-managed key (free) or customer-managed KMS key (pay per API call).

```bash
# Write a SecureString parameter
aws ssm put-parameter \
  --name "/prod/db/password" \
  --value "s3cr3t" \
  --type SecureString \
  --key-id alias/aws/ssm   # AWS-managed key — free

# Read it (value is decrypted transparently with --with-decryption)
aws ssm get-parameter \
  --name "/prod/db/password" \
  --with-decryption
```

### Hierarchy — path-based organization

Parameter Store supports slash-separated paths, making it easy to group by environment and service:

```
/prod/api/db_host
/prod/api/db_password
/prod/api/jwt_secret
/staging/api/db_host
/staging/api/db_password
```

Get all params for a service in one call:

```bash
aws ssm get-parameters-by-path --path "/prod/api/" --with-decryption --recursive
```

---

## External Secrets Operator (ESO) — the K8s bridge

ESO is the standard pattern for syncing AWS secrets into Kubernetes. It introduces two CRDs:

- **SecretStore** (or **ClusterSecretStore** for cluster-wide): defines the backend (which AWS account, which region, which IRSA role to use)
- **ExternalSecret**: maps an AWS secret to a Kubernetes Secret

```yaml
# ClusterSecretStore — backed by Secrets Manager via IRSA
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

```yaml
# ExternalSecret — sync a Secrets Manager secret to a K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials     # the K8s Secret name to create
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD  # key in the K8s Secret
      remoteRef:
        key: prod/db/password   # Secrets Manager secret name
        property: password      # JSON key within the secret value
```

ESO creates and updates the K8s Secret on the `refreshInterval`. Pods consume the K8s Secret normally — they don't call AWS APIs directly.

**ESO with Parameter Store** — same pattern, different `service`:

```yaml
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-east-1
```

The pod doesn't know or care whether the secret came from Secrets Manager or Parameter Store. That's the clean separation.

---

## Decision framework — the 4 questions

When choosing between these tools, walk through:

**CloudWatch vs Prometheus:**
1. Who generates the metric? AWS service → CloudWatch. App pod → Prometheus.
2. Do I need PromQL? Yes → Prometheus (or AMP). No → CloudWatch is fine.
3. What's the metric volume? < 500 custom metrics → CloudWatch cost is manageable. > 500 → Prometheus or AMP saves money.
4. Do I want to operate a stack? No → CloudWatch (or AMP). Yes, for maximum control → self-hosted Prometheus.

**Secrets Manager vs Parameter Store:**
1. Does the secret need automatic rotation? Yes → Secrets Manager. No → either.
2. How many secrets? < 50 → Secrets Manager cost is fine. > 200 → Parameter Store at scale.
3. Do I need fine-grained per-secret audit? Yes → Secrets Manager (access audited in CloudTrail at the secret level). Parameter Store also logs to CloudTrail but at the API level.
4. Is this config or a secret? Config (feature flags, log levels) → Parameter Store String. Secret (password, token) → Parameter Store SecureString or Secrets Manager.

---

## The interview answer in 60 seconds

> "CloudWatch and Prometheus are complementary, not competing. CloudWatch for AWS-native metrics — RDS, Lambda, ALB — because those land there automatically with zero configuration. Prometheus for app-level metrics from pods, because PromQL and histogram_quantile are essential for SLO work and high-cardinality app metrics. In practice I'd run both, with Grafana as the unified frontend using two data sources. For teams that don't want to operate Prometheus, AMP is the managed middle ground — keeps PromQL, removes the operational burden, at $0.10/metric/month vs CloudWatch's $0.30.
>
> Secrets Manager vs Parameter Store: Secrets Manager when the secret needs automatic rotation — database passwords especially. The built-in rotation Lambda rotates the RDS password, tests the new one, then promotes it. Zero-downtime rotation. Cost is $0.40/secret/month — fine for tens of secrets, expensive for hundreds.
>
> Parameter Store for non-rotating config and secrets at scale. SecureString for actual secrets (KMS-encrypted), String for plaintext config. Much cheaper at scale — Standard tier is free for storage.
>
> In Kubernetes, ESO (External Secrets Operator) bridges either backend into K8s Secrets. Pods don't know if the secret came from Secrets Manager or Parameter Store — ESO syncs it on a refresh interval. The IRSA role for ESO gets read-only access to the specific secret paths. That's the clean pattern."

---

## Self-test drills

### 1. When would you pick CloudWatch over Prometheus, and Secrets Manager over Parameter Store?

**Reference answer:** (the 60-second answer above) — CloudWatch for AWS-native metrics; Prometheus for app metrics and PromQL. Secrets Manager for rotation; Parameter Store for scale or non-rotating secrets. Both pairs are complementary; ESO bridges either secrets backend into K8s.

### 2. Walk through Secrets Manager rotation mechanics. How does zero-downtime rotation work?

**Reference answer:**
- Rotation calls a Lambda through four phases: createSecret (generate new value as AWSPENDING) → setSecret (update target, e.g., RDS password) → testSecret (verify new value works) → finishSecret (promote AWSPENDING to AWSCURRENT, demote old to AWSPREVIOUS).
- During rotation, both AWSCURRENT and AWSPREVIOUS are retrievable. Apps reading the secret during rotation get the current value. If a rolling deploy is in progress, old pods still get AWSCURRENT until rotation completes.
- Zero-downtime: the new password is set on the DB before the old one is invalidated. The testSecret phase verifies connectivity. Only after verification does AWSCURRENT flip to the new value.
- The gotcha: if your app caches the secret value (e.g., hardcodes it at startup), rotation won't help — the pod needs to re-read Secrets Manager on every connection, or ESO needs to resync and you need to restart pods.

### 3. What is ESO and why does it exist?

**Reference answer:**
- ESO (External Secrets Operator) syncs secrets from external backends (Secrets Manager, Parameter Store, Vault, GCP Secret Manager) into Kubernetes Secret objects.
- Exists because: (a) pods should use K8s Secrets natively (env vars, volume mounts); (b) but storing secrets in etcd via Kubernetes manifests is unsafe; (c) ESO keeps the source of truth in a secure backend (Secrets Manager/Parameter Store) while giving pods the standard K8s Secret experience.
- Two CRDs: `SecretStore` (or `ClusterSecretStore`) defines the backend + auth; `ExternalSecret` maps a remote key to a K8s Secret. ESO syncs on `refreshInterval`.
- Auth is via IRSA — the ESO ServiceAccount has an IAM role with read access to specific secret paths. No static credentials.
- Bonus: ESO supports templating (compose multiple secrets into one K8s Secret), pushSecret (write K8s Secrets to external backends), and secret version tracking.

### 4. When does it make sense to use Amazon Managed Prometheus (AMP) instead of self-hosted Prometheus?

**Reference answer:**
- AMP is justified when: (a) the team doesn't want to operate Prometheus HA, TSDB storage scaling, and retention management; (b) you still need PromQL (unlike CloudWatch); (c) $0.10/metric/month is cheaper than the engineering time to operate self-hosted Prometheus.
- Self-hosted Prometheus wins when: (a) full control over scrape configs and retention; (b) very large metric volumes where even AMP pricing is expensive; (c) you already have the expertise and operational runbooks.
- Practical pattern: start with kube-prometheus-stack (self-hosted). When the operational burden becomes real (TSDB WAL issues, OOM on Prometheus pod, storage scaling), evaluate AMP. Many teams never need to migrate.
- AMP doesn't replace the scraping agent — you still run Prometheus or ADOT in the cluster for scraping. AMP replaces the storage and query layer only.

---

## Further reading / watching

- **AWS docs — CloudWatch Container Insights for EKS**: the setup guide explains the DaemonSet, cost model, and available metrics
- **AWS docs — Amazon Managed Prometheus**: the pricing page and the remote_write configuration guide
- **Prometheus docs — Storage**: understanding TSDB and retention is the key to operating self-hosted Prometheus
- **External Secrets Operator docs** ([external-secrets.io](https://external-secrets.io/)): the SecretStore + ExternalSecret tutorial is the fastest hands-on start
- **AWS docs — Secrets Manager rotation**: "Rotating secrets" in the Secrets Manager User Guide — the four-phase Lambda lifecycle is documented in detail
- **AWS blog — "Secrets Manager vs Parameter Store"**: search the title for the most current cost comparison

---

## The 4 dimensions (senior framing)

- **Tech**: CloudWatch for AWS-native; Prometheus for app metrics + PromQL; AMP as the managed middle. Secrets Manager for rotation ($0.40/secret/month); Parameter Store SecureString for non-rotating secrets at scale. ESO (IRSA + ExternalSecret CRDs) for K8s secret injection. Key cost number: CloudWatch custom metrics $0.30/metric/month vs AMP $0.10/metric/month vs self-hosted Prometheus near-zero marginal.

- **People**: The "CloudWatch vs Prometheus" debate is a false dichotomy that causes friction between infra and app teams. Frame it as "each tool for the right layer" and have one Grafana that both teams use. For secrets, define a clear policy: "anything that rotates goes in Secrets Manager; everything else goes in Parameter Store." Document the path prefix schema (`/prod/service/key`) and enforce it in IRSA policies (restrict to specific paths).

- **CI/CD**: Grafana dashboards and alerting rules as code (Grafonnet or JSON in Git). Prometheus recording rules and alert rules as PrometheusRule CRDs in Git (managed by the Prometheus Operator). Secrets Manager rotation tests in CI — deploy a Lambda rotation function change and verify it doesn't break the rotation test step. ESO ExternalSecret manifests are part of the app's Helm chart — the secret reference is declared alongside the app that uses it.

- **Operations**: Alert on ESO sync failures (`externalsecret_sync_calls_error` metric from ESO). Alert on Secrets Manager rotation failures (CloudTrail event `RotateSecret` with errorCode). Monitor Prometheus scrape success (`up == 0` means a pod's `/metrics` endpoint is unreachable). For CloudWatch, set composite alarms on correlated metrics (e.g., "high CPU + low IOPS" on RDS) rather than individual metric alarms that produce noise. Runbook for "pod can't read secret": check ESO pod logs, verify IRSA role binding, verify the ExternalSecret status (`kubectl describe externalsecret`).
