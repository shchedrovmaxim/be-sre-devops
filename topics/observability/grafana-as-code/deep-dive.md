# Grafana dashboards as code — the deep-dive

> **Goal**: by the end you can answer — **"Why dashboards as code over the UI, and what's the actual workflow?"** — covering Grafonnet/jsonnet, the Foundation SDK, the Grafana Operator, provisioning, testing, and folder/RBAC management as code.

> Start with [grafana-as-code-simple.md](./simple.md) first. The blueprint analogy is the spine.

---

## The senior framing — "I can rebuild every dashboard from Git"

The test is simple: if Grafana's database was wiped right now, how long would it take to restore all dashboards?

- **Click-ops answer**: hours of manual recreation, and some dashboards are gone forever (no one remembers what they looked like).
- **Dashboards as code answer**: `helm upgrade grafana . --values production.yaml` → 5 minutes to provisioning, dashboards restored from Git automatically.

That's the senior signal. Not "we use jsonnet" — but "our dashboards are reproducible from source, reviewed in PRs, and version-controlled with the same discipline as application code."

---

## Grafonnet / jsonnet — the canonical approach

Grafana dashboards are JSON blobs. jsonnet is a data templating language that generates JSON with variables, loops, imports, and functions. Grafonnet is the jsonnet library for Grafana's dashboard schema.

### Install

```bash
# Install jsonnet
brew install jsonnet   # or go install github.com/google/go-jsonnet/cmd/jsonnet@latest

# Install grafonnet via jsonnet bundler
go install github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb@latest
jb init
jb install github.com/grafana/grafonnet/gen/grafonnet-latest@main
```

### A working example

```jsonnet
// dashboards/checkout-api.jsonnet
local grafonnet = import 'github.com/grafana/grafonnet/gen/grafonnet-latest/main.libsonnet';
local dashboard = grafonnet.dashboard;
local panel = grafonnet.panel;
local query = grafonnet.query;

dashboard.new('Checkout API')
+ dashboard.withUid('checkout-api')
+ dashboard.withDescription('Checkout service RED metrics')
+ dashboard.withTags(['checkout', 'slo'])
+ dashboard.withTimezone('browser')
+ dashboard.withRefresh('30s')
+ dashboard.panels.withPanels([
    // Row: Request Rate (R)
    panel.timeSeries.new('Request Rate (req/s)')
    + panel.timeSeries.gridPos.withH(8)
    + panel.timeSeries.gridPos.withW(12)
    + panel.timeSeries.gridPos.withX(0)
    + panel.timeSeries.gridPos.withY(0)
    + panel.timeSeries.queryOptions.withTargets([
        query.prometheus.new(
          'Prometheus',
          'sum by (job)(rate(http_requests_total{job="checkout-api"}[5m]))'
        )
        + query.prometheus.withLegendFormat('{{job}}'),
    ]),

    // Row: Error Rate (E)
    panel.timeSeries.new('Error Rate (%)')
    + panel.timeSeries.gridPos.withH(8)
    + panel.timeSeries.gridPos.withW(12)
    + panel.timeSeries.gridPos.withX(12)
    + panel.timeSeries.gridPos.withY(0)
    + panel.timeSeries.queryOptions.withTargets([
        query.prometheus.new(
          'Prometheus',
          |||
            100 * sum by (job)(rate(http_requests_total{job="checkout-api",status=~"5.."}[5m]))
            / sum by (job)(rate(http_requests_total{job="checkout-api"}[5m]))
          |||
        )
        + query.prometheus.withLegendFormat('error %'),
    ]),
])
```

### Compile and deploy

```bash
# Compile to JSON
jsonnet -J vendor dashboards/checkout-api.jsonnet > dist/checkout-api.json

# Push to Grafana API
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
  -d "{\"dashboard\": $(cat dist/checkout-api.json), \"overwrite\": true}" \
  https://grafana.internal/api/dashboards/db

# Or: commit the JSON to the dashboards/ directory
# and let file provisioning or the Grafana Operator pick it up
```

---

## Grafana Foundation SDK — the newer approach

The Foundation SDK is the Grafana Labs-maintained alternative to Grafonnet, available in Go, TypeScript, and Python. It generates the same JSON but using typed APIs in those languages.

```go
// dashboards/checkout.go
package dashboards

import (
    "github.com/grafana/grafana-foundation-sdk/go/cog"
    "github.com/grafana/grafana-foundation-sdk/go/common"
    "github.com/grafana/grafana-foundation-sdk/go/dashboard"
    "github.com/grafana/grafana-foundation-sdk/go/timeseries"
)

func CheckoutDashboard() *cog.Builder[dashboard.Dashboard] {
    return dashboard.NewDashboardBuilder("Checkout API").
        Uid("checkout-api").
        Tags([]string{"checkout", "slo"}).
        Refresh("30s").
        WithPanel(
            timeseries.NewPanelBuilder().
                Title("Request Rate (req/s)").
                WithTarget(
                    prometheus.NewDataqueryBuilder().
                        Expr(`sum by (job)(rate(http_requests_total{job="checkout-api"}[5m]))`).
                        LegendFormat("{{job}}"),
                ),
        )
}
```

**When to use Foundation SDK vs Grafonnet:**
- Foundation SDK: team is primarily Go/TypeScript, wants type safety and IDE autocompletion for dashboard authoring.
- Grafonnet: existing jsonnet tooling, or you prefer a JSON-first approach.

Both produce identical output (a Grafana dashboard JSON). The choice is tooling preference.

---

## Grafana Operator — Kubernetes-native GitOps

The [Grafana Operator](https://github.com/grafana/grafana-operator) watches `GrafanaDashboard`, `GrafanaFolder`, `GrafanaAlertRuleGroup`, and `GrafanaDatasource` CRDs and reconciles them against the Grafana API.

### Dashboard CRD

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: checkout-api
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana  # which Grafana instance to push to
  folder: "Checkout Team"
  json: |
    {
      "title": "Checkout API",
      "uid": "checkout-api",
      "tags": ["checkout", "slo"],
      "panels": [
        {
          "type": "timeseries",
          "title": "Request Rate",
          "targets": [
            {
              "expr": "sum by (job)(rate(http_requests_total{job=\"checkout-api\"}[5m]))",
              "legendFormat": "{{job}}"
            }
          ]
        }
      ]
    }
```

### The GitOps workflow with the Operator

1. Dashboard JSON lives in an app team's Helm chart as a `GrafanaDashboard` resource.
2. ArgoCD syncs the Helm chart to the cluster.
3. Grafana Operator sees the new `GrafanaDashboard` resource and pushes it to Grafana.
4. Dashboard appears in Grafana in the specified folder.

No API token management, no manual `curl`. The Operator handles authentication to Grafana.

### Folder management as code

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaFolder
metadata:
  name: checkout-team
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  title: "Checkout Team"
```

### Alert rule groups as code

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaAlertRuleGroup
metadata:
  name: checkout-slo-alerts
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  folderRef: checkout-team
  interval: 1m
  rules:
    - title: CheckoutHighErrorRate
      condition: C
      data:
        - refId: A
          relativeTimeRange:
            from: 600
            to: 0
          model:
            expr: |
              sum(rate(http_requests_total{job="checkout-api",status=~"5.."}[5m]))
              / sum(rate(http_requests_total{job="checkout-api"}[5m]))
      noDataState: NoData
      execErrState: Alerting
      for: 5m
      labels:
        severity: page
```

---

## File-based provisioning — the simplest approach

No operator needed — Grafana watches a directory:

```yaml
# grafana.ini
[paths]
provisioning = /etc/grafana/provisioning

# /etc/grafana/provisioning/dashboards/default.yaml
apiVersion: 1
providers:
  - name: default
    type: file
    disableDeletion: true      # UI cannot delete provisioned dashboards
    updateIntervalSeconds: 60  # poll interval
    options:
      path: /etc/grafana/dashboards/
      foldersFromFilesStructure: true  # directory structure → folder names
```

In Kubernetes, mount dashboard JSON files via ConfigMap:

```yaml
# In your Grafana Helm values
dashboards:
  default:
    checkout-api:
      json: |
        { ... dashboard json ... }
```

Or use `dashboardsConfigMaps`:

```yaml
dashboardsConfigMaps:
  default: grafana-dashboards
```

Where `grafana-dashboards` is a ConfigMap containing one key per dashboard JSON file. A git-sync sidecar or a CI pipeline that commits JSON files can populate this ConfigMap.

---

## Testing dashboards

### `dashboard-linter`

```bash
# Install
go install github.com/grafana/dashboard-linter@latest

# Lint a dashboard JSON
dashboard-linter lint --strict dist/checkout-api.json
```

Checks for: missing UIDs, duplicate panel IDs, invalid PromQL syntax (limited), deprecated panel types.

### `mimirtool analyze`

```bash
# Find all PromQL expressions used in dashboards and validate them
mimirtool analyze grafana \
  --address=http://grafana.internal \
  --key=${GRAFANA_TOKEN}

# Check which dashboards use specific metrics
mimirtool analyze grafana --metrics http_requests_total
```

This shows you which dashboards would break if a metric is renamed or dropped — invaluable for planned metrics migrations.

### Testing PromQL queries

```bash
# Validate PromQL syntax without a running Prometheus
promtool query instant http://prometheus:9090 \
  'sum by (job)(rate(http_requests_total[5m]))'
```

---

## The provisioning loop — choosing an approach

| Approach | How it works | Best for |
|---|---|---|
| File provisioning | Grafana watches a directory; mount via ConfigMap | Simple setups, no Kubernetes CRD preference |
| API push (CI/CD) | CI pipeline pushes JSON to Grafana API after merge | Teams without Kubernetes operator |
| Grafana Operator CRD | Operator reconciles `GrafanaDashboard` CRDs | Kubernetes-native GitOps teams |
| Terraform | `grafana_dashboard` resource in Terraform | Teams managing Grafana config in Terraform |

The Terraform approach:

```hcl
resource "grafana_dashboard" "checkout_api" {
  config_json = file("${path.module}/dashboards/checkout-api.json")
  folder      = grafana_folder.checkout.id
  overwrite   = true
}

resource "grafana_folder" "checkout" {
  title = "Checkout Team"
  uid   = "checkout"
}
```

---

## Access control as code

### Grafana RBAC via the Operator

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaFolder
metadata:
  name: checkout-team
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  title: "Checkout Team"
  permissions:
    items:
      - role: Editor
        permission: Edit
      - teamId: 5   # checkout team ID in Grafana
        permission: Admin
```

### Via Terraform

```hcl
resource "grafana_folder_permission" "checkout" {
  folder_uid = grafana_folder.checkout.uid
  permissions {
    team_id    = grafana_team.checkout.id
    permission = "Edit"
  }
  permissions {
    role       = "Viewer"
    permission = "View"
  }
}
```

---

## The 60-second interview answer

> "Dashboards as code means every dashboard lives in Git, gets reviewed in a PR, and is reproducible from source. The UI is for exploration; production dashboards need version history, review, and the ability to rebuild.
>
> The three implementation paths: Grafonnet/jsonnet — a jsonnet library that compiles to Grafana's JSON format, great if you want programmatic generation and templating. The Grafana Foundation SDK — a newer, typed approach in Go/TypeScript/Python if you prefer those languages. The Grafana Operator — a Kubernetes CRD-based approach where `GrafanaDashboard` resources live in your Helm charts and the operator pushes them to Grafana automatically, which is the cleanest GitOps path.
>
> The provisioning loop: dashboards can be pushed via Grafana's file provisioning (mount a directory of JSON files), via API in CI after merge, or via the Operator reconciliation loop. Provisioned dashboards are read-only in the UI — you can't accidentally overwrite them.
>
> The senior bar: `dashboard-linter` in CI for syntax, `mimirtool analyze` to verify the PromQL against actual metrics, and the ability to answer 'if Grafana's database was wiped, could we rebuild everything from Git in 30 minutes?' If yes, you're doing it right."

---

## Self-test drills

### Drill 1 — why code over UI

A colleague argues: "We have 3 dashboards, they're simple, why bother with jsonnet?" How do you respond?

**Answer:** Three dashboards now. But someone will clone them for another service (now you have 6, all slightly different, all inconsistent). Someone will change a panel in the UI (now you have undocumented changes, no review, no history). Grafana's DB will get corrupted in a restore or migration (now three dashboards are partially recovered). The cost of doing dashboards as code with 3 dashboards is 2 hours. The cost of click-ops debt with 30 dashboards is days of incident recovery and weeks of inconsistency. The right time to start is now.

### Drill 2 — provisioning

A new `GrafanaDashboard` CRD was merged to the monitoring namespace. Grafana shows the old version of the dashboard. What do you check?

**Answer:**
1. `kubectl get grafanadashboard checkout-api -n monitoring -o yaml` — check status conditions for errors from the operator.
2. `kubectl logs -n monitoring deployment/grafana-operator` — look for API errors (auth failure, JSON validation error).
3. Check the `instanceSelector` in the CRD matches the `dashboards: grafana` label on the Grafana instance.
4. Verify the dashboard `uid` — if a dashboard with the same UID was created manually in the UI, there may be a conflict.

### Drill 3 — testing

How would you verify that a PromQL expression in a dashboard is still valid after a metrics rename?

**Answer:** `mimirtool analyze grafana --address=http://grafana.internal --key=${GRAFANA_TOKEN}` — this scrapes all dashboards from Grafana, extracts all PromQL expressions, and checks which metrics they reference. If the old metric name is gone from Prometheus, mimirtool will flag those panels. Add this to CI on the metrics rename PR to catch dashboard breakage before it ships.

---

## Further reading

- [Grafonnet docs](https://grafana.github.io/grafonnet/API/)
- [Grafana Foundation SDK GitHub](https://github.com/grafana/grafana-foundation-sdk)
- [Grafana Operator](https://github.com/grafana/grafana-operator)
- [Grafana provisioning docs](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [mimirtool analyze](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/)
- [dashboard-linter](https://github.com/grafana/dashboard-linter)

---

## The 4 dimensions

- **Tech**: Grafonnet/jsonnet for programmatic JSON generation; Foundation SDK for typed Go/TS/Python; Grafana Operator for CRD-based Kubernetes GitOps; file provisioning for simpler setups; Terraform for infra-as-code parity. `dashboard-linter` + `mimirtool analyze` for CI validation. UIDs must be stable and unique — auto-generated UIDs break across redeploys.
- **People**: provisioned dashboards are read-only in the UI — this prevents accidental overwrites but also requires a PR to make any change. Communicate this expectation clearly. Provide a "dev Grafana" environment where teams can iterate in the UI, then export JSON and commit it. Don't make the process so rigid that teams give up and use a shadow Grafana instance.
- **CI/CD**: CI pipeline: `jsonnet -J vendor` to compile → `dashboard-linter lint` → `mimirtool analyze` → commit JSON to dashboards directory → ArgoCD syncs → Operator pushes to Grafana. On every metrics-renaming PR: run mimirtool to catch dashboard breakage. Dashboard UIDs are stable identifiers — never auto-generate them in CI.
- **Operations**: `grafana_dashboard_version` metric shows the current version of each dashboard — alert on unexpected changes (someone edited in the UI and overrode the provisioned version). Backup Grafana's SQLite/PostgreSQL DB nightly — even with dashboards as code, there's other state (annotations, users, org settings) that isn't code. The Grafana Operator's reconciliation loop is the source of truth — if a dashboard drifts, the next reconciliation cycle restores it.
