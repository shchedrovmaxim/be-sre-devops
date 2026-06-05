# Grafana dashboards as code — the simple version (the blueprint analogy)

> Read this first. Once the blueprint analogy clicks, the [deep-dive](./grafana-as-code.md) becomes easy.

This doc explains **one idea**:

> **A Grafana dashboard saved by clicking "Save" in the UI is like building a house without blueprints. You can't reproduce it, you can't review it, and when the house burns down, it's gone. Dashboards as code are the blueprints.**

---

## Why "I'll just click and build it in Grafana" doesn't scale

The UI is great for exploration. It's not a production practice.

| Click-ops dashboards | Dashboards as code |
|---|---|
| Lives in a database | Lives in Git |
| No review process | PR review — anyone can spot a broken query |
| Changes are invisible | Every change is a diff |
| "Who changed this panel?" → mystery | `git blame` gives you the answer |
| Server crashes → dashboard gone | Rebuilt from code in minutes |
| 10 similar dashboards → 10 manual copies | One template → 10 generated dashboards |

The senior framing: **a dashboard is code**. It has bugs, it has versions, it has reviewers. Treat it like code.

---

## The two approaches you'll see in the wild

### 1. Grafonnet / jsonnet (the canonical)

Grafana's dashboard format is JSON. Grafonnet is a jsonnet library that generates that JSON programmatically:

```jsonnet
// dashboards/checkout.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local graphPanel = grafana.graphPanel;

dashboard.new('Checkout API', uid='checkout-api')
.addPanel(
  graphPanel.new('Request Rate')
  .addTarget({
    expr: 'sum by (job)(rate(http_requests_total{job="checkout-api"}[5m]))',
    legendFormat: '{{job}}'
  })
)
```

jsonnet compiles to JSON, which you push to Grafana via API or provision via file.

### 2. Grafana Operator + Dashboard CRD (the Kubernetes-native way)

In a Kubernetes environment, create a `GrafanaDashboard` custom resource:

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: checkout-api
  namespace: monitoring
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana
  json: |
    {
      "title": "Checkout API",
      "uid": "checkout-api",
      "panels": [...]
    }
```

The Grafana Operator watches `GrafanaDashboard` resources and pushes them to Grafana automatically. No manual API calls. GitOps-native.

---

## The provisioning loop (file-based)

The simplest approach — no operator needed:

```yaml
# grafana.ini / Helm values
dashboards:
  default:
    my-dashboards:
      folder: /etc/grafana/dashboards   # a directory of JSON files
      disableDeletion: true             # don't allow UI deletion of provisioned dashboards
      updateIntervalSeconds: 60
```

Grafana watches the directory and reloads dashboards automatically. Mount the JSON files via ConfigMap or a git-sync sidecar. Provisioned dashboards show a lock icon in the UI — you can view but not accidentally overwrite them from the UI.

---

## The workflow in production

1. Engineer writes jsonnet (or edits JSON) in a feature branch.
2. PR review — teammates verify the PromQL, the panel layout, the alert thresholds.
3. Merge → CI compiles jsonnet to JSON → commits to a `dashboards/` directory.
4. Grafana picks up the new JSON within 60 seconds (file provisioning) or immediately (Operator).
5. The dashboard is in production, reviewable in Git, reproducible forever.

---

## Self-test (out loud)

> **"Why dashboards as code over the UI, and what's the actual workflow?"**

**Reference answer:**

"The UI is for exploration. Production dashboards belong in Git for the same reason production code belongs in Git: you need review, history, reproducibility, and the ability to rebuild after an incident.

The workflow: write or generate dashboard JSON (via Grafonnet/jsonnet for programmatic generation, or the Grafana Foundation SDK for typed languages), review in a PR, merge to main, deploy via file provisioning or the Grafana Operator in Kubernetes. The dashboard shows up in Grafana automatically. Provisioned dashboards are read-only in the UI — you can't accidentally overwrite them.

The senior test: if your Grafana server died right now, could you rebuild every dashboard from Git in under 30 minutes? If yes, you're doing dashboards as code. If no, you have click-ops debt."

---

## Further reading

- [Grafonnet](https://grafana.github.io/grafonnet/API/)
- [Grafana Foundation SDK](https://github.com/grafana/grafana-foundation-sdk) (Go/TypeScript/Python)
- [Grafana Operator](https://github.com/grafana/grafana-operator)
- [Grafana provisioning docs](https://grafana.com/docs/grafana/latest/administration/provisioning/)

---

## Next: the deep-dive

When the blueprint analogy is obvious, jump to [`grafana-as-code.md`](./grafana-as-code.md). The deep-dive covers:

- Grafonnet / jsonnet workflow with a full working example
- Grafana Foundation SDK (the newer Go/TypeScript/Python approach)
- Grafana Operator and Dashboard CRD for Kubernetes-native GitOps
- The provisioning loop (file vs API vs Operator)
- Testing dashboards (mimirtool analyze, dashboard-linter)
- Folder and access-control management as code
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
