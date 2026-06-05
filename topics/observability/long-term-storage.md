# Long-term Prometheus storage — the deep-dive

> **Goal**: by the end you can answer — **"When would you pick Mimir vs Thanos vs VictoriaMetrics for long-term Prometheus storage?"** — covering the architectural trade-offs, deduplication, downsampling, and federation patterns.

> Start with [long-term-storage-simple.md](./long-term-storage-simple.md) first. The hot-desk vs archive-room framing is the spine.

---

## The senior framing — why Prometheus alone isn't enough

Prometheus was designed for local, fast-access, short-retention data. The original design intent: keep 15–30 days of raw data on local SSD, run fast queries, restart cleanly. It was never meant to be a long-term archive.

In production, that leaves four problems:

1. **Data loss on node failure.** A Prometheus StatefulSet on a node that dies loses its data if the PVC is gone.
2. **HA duplication.** Two Prometheus replicas scraping the same targets produce the same data twice — queries without deduplication show doubled values.
3. **No global view.** With separate Prometheus per cluster, you can't query "all clusters" in one query.
4. **Storage cost grows linearly with series count × retention.** At 500K series × 180 days, local disk becomes expensive.

All three long-term storage solutions address these. They differ in operational complexity, scalability ceiling, and feature surface.

---

## Thanos — the most flexible, most complex

### Architecture

Thanos is a sidecar architecture layered on top of existing Prometheus:

```
Cluster A:
  [Prometheus] ←── scrapes targets
  [Thanos Sidecar] ─── ships 2h blocks to S3 every 2h
                   ─── exposes StoreAPI for Querier

Cluster B:
  [Prometheus] + [Thanos Sidecar] → S3

S3 bucket:
  [Thanos Store Gateway] ─── serves historical blocks from S3
  [Thanos Compactor]     ─── merges, deduplicates, downsamples blocks
  [Thanos Ruler]         ─── evaluates recording/alerting rules globally

Global:
  [Thanos Querier] ─── fans out queries to all Sidecars + Store Gateway
                   ─── deduplicates results
  [Thanos Frontend] (optional) ─── query caching, query splitting
```

### Component roles

| Component | What it does |
|---|---|
| **Sidecar** | Runs next to each Prometheus. Ships completed 2h TSDB blocks to S3. Exposes a gRPC StoreAPI so the Querier can reach real-time data from that Prometheus. |
| **Store Gateway** | Reads historical blocks from S3 and serves them via StoreAPI. Indexes block metadata in memory. Supports caching (Memcached, Redis). |
| **Compactor** | Single-writer process that compacts small blocks into larger ones, runs deduplication on overlapping blocks from HA replicas, and creates downsampled blocks (5m, 1h resolution). |
| **Querier** | The query entry point. Fans out PromQL to all StoreAPI backends (sidecars, store gateways), merges results, deduplicates overlapping series. Prometheus-API compatible — Grafana points here. |
| **Ruler** | Evaluates Prometheus recording/alerting rules against the global view (across all clusters). Required if you want cross-cluster alerting. |

### Deduplication in Thanos

Each Prometheus replica gets a unique `external_labels` block:

```yaml
# Prometheus A config
global:
  external_labels:
    replica: a
    cluster: prod-us-east

# Prometheus B config
global:
  external_labels:
    replica: b
    cluster: prod-us-east
```

The Compactor and Querier use the `replica` label for deduplication: series that differ only in `replica` label value are merged into a single series. The `--deduplication.replica-label=replica` flag in the Querier enables this.

### Downsampling

The Compactor automatically creates downsampled resolution blocks:

```
Raw (15s or 30s samples)    → retained for configured raw retention
5-minute resolution blocks  → created after raw blocks age past 40h
1-hour resolution blocks    → created after 5m blocks age past 10d
```

Configure retention per resolution in the Compactor:

```yaml
# thanos compactor flags
--retention.resolution-raw=30d   # keep raw for 30 days
--retention.resolution-5m=90d    # keep 5m resolution for 90 days
--retention.resolution-1h=365d   # keep 1h resolution for 1 year
```

Grafana's Thanos datasource automatically selects the right resolution based on the query time range.

---

## Mimir — cloud-native, horizontally scalable

### Architecture

Mimir is a horizontally scalable, multi-tenant Prometheus backend originally forked from Cortex. It accepts **remote-write** from Prometheus (push, not sidecar):

```
Prometheus ─── remote_write ──→ Mimir (Distributor)
                                     │
                                ┌────┴─────┐
                           Ingester    Ingester   (write path, in-memory)
                                │         │
                              Compactor + Store Gateway (read path, S3)
                                     │
                                   Querier ←── Grafana
```

### Key components

| Component | What it does |
|---|---|
| **Distributor** | Receives remote-write, validates, shards series across Ingesters by consistent hashing. Replicates to 3 Ingesters by default. |
| **Ingester** | Buffers series in memory (WAL-backed). Flushes blocks to S3 every 2h. |
| **Querier** | Reads from Ingesters (recent data) and Store Gateway (historical). Merges, deduplicates. |
| **Store Gateway** | Serves blocks from S3. Same role as Thanos Store Gateway. |
| **Compactor** | Merges, deduplicates, and downsamples blocks in S3. |
| **Ruler** | Evaluates recording and alerting rules per-tenant. |

### Why Mimir over Thanos

- **No sidecar** — remote-write is simpler to configure and doesn't require sidecar containers alongside each Prometheus.
- **Multi-tenancy built-in** — each tenant (team, cluster) is isolated. Cardinality limits per tenant. Billing per tenant.
- **Horizontal scaling** — each component scales independently. The ingester scales to handle 10M+ series per node.
- **Grafana Labs managed** — Grafana Cloud runs Mimir. You can hand off ops entirely.

### Mimir remote-write config in Prometheus

```yaml
remote_write:
  - url: http://mimir:8080/api/v1/push
    headers:
      X-Scope-OrgID: my-team    # multi-tenancy header
    queue_config:
      capacity: 10000
      max_shards: 30
      max_samples_per_send: 2000
    write_relabel_configs:
      # Optional: drop metrics before sending
      - source_labels: [__name__]
        action: drop
        regex: "go_gc_.*"
```

---

## VictoriaMetrics — the simple, fast alternative

### Architecture

VictoriaMetrics is a monolithic time-series database that's Prometheus-compatible at the API level. The single-node binary handles ingest, storage, and query:

```
Prometheus ─── remote_write ──→ [VictoriaMetrics single binary]
                                  (or vmagent → vminsert → vmstorage → vmselect for cluster mode)

Grafana ──── PromQL / MetricsQL ──→ VictoriaMetrics
```

### Why VictoriaMetrics is often the right choice

- **Compression**: VictoriaMetrics uses a custom compression algorithm. At the same series count, it uses ~4–10× less disk than Prometheus TSDB.
- **Ingest speed**: Benchmarks consistently show 2–5× higher ingest throughput than Prometheus or Mimir per core.
- **Simplicity**: Single binary to operate. No sidecar. No distributed system to debug.
- **Retention**: set `--retentionPeriod=24` for 2 years retention. That's all.
- **MetricsQL**: a superset of PromQL with additional functions. All standard PromQL works as-is.

### VictoriaMetrics limits

- Deduplication in cluster mode exists but is simpler than Thanos/Mimir (exact timestamp matching required).
- Native multi-tenancy is only in the cluster version.
- Less ecosystem tooling and commercial support than Thanos/Mimir.
- Global query federation across multiple clusters requires running multiple VictoriaMetrics instances and using `vmcluster` or `Grafana data source mixing`.

### VictoriaMetrics Prometheus config

```yaml
# prometheus.yml
remote_write:
  - url: http://victoriametrics:8428/api/v1/write
```

Or replace the Prometheus scrape layer entirely with `vmagent` (VictoriaMetrics's lightweight scrape agent):

```yaml
# vmagent can replace the Prometheus binary for scraping
# Uses the same scrape_config format, less RAM, supports sharding
```

---

## Trade-off matrix

| | Thanos | Mimir | VictoriaMetrics |
|---|---|---|---|
| **Architecture** | Sidecar + components | Remote-write + microservices | Single binary (or cluster) |
| **Operational complexity** | High (8+ components) | High (7+ components) | Low (1 binary) |
| **Multi-tenancy** | Limited (label-based) | Native, per-tenant limits | Cluster version only |
| **Global view** | Yes (Querier fans out) | Yes (native) | With vmcluster / multi-source Grafana |
| **Deduplication** | Yes (replica label) | Yes (native) | Yes (in cluster mode) |
| **Downsampling** | Yes (Compactor) | Yes (Compactor) | No native downsampling |
| **Scalability ceiling** | ~10M series per Prometheus | 100M+ series (Mimir scales horizontally) | ~10M series (single node), more in cluster |
| **Disk efficiency** | Good (TSDB compression) | Good | Excellent (~4–10× better than Prometheus) |
| **Managed offering** | No official managed | Grafana Cloud Mimir | Managed cloud.victoriametrics.com |
| **When to choose** | Already running Prometheus, need federation/HA | Large scale, multi-tenant, cloud budget | Small-to-medium, simplicity, on-prem |

---

## Query federation patterns

### Thanos — single Querier federates all clusters

```yaml
# thanos querier flags
--store=sidecar-cluster-a:10901
--store=sidecar-cluster-b:10901
--store=store-gateway:10901
```

A single PromQL query fans out to all stores. Transparent to Grafana — one datasource points at the Thanos Querier.

### Mimir — per-tenant federation

Each cluster writes to a different `X-Scope-OrgID` tenant. Grafana uses a Mimir datasource with tenant ID switching for per-cluster views, or a "meta-tenant" for cross-cluster queries (Mimir's `multi-tenant-query` feature).

### VictoriaMetrics — Grafana data source mixing

Run one VictoriaMetrics per cluster. In Grafana, use the `Mixed` datasource option or the `datasource` variable to switch clusters within a dashboard. Less elegant but operationally simple.

---

## The 60-second interview answer

> "Prometheus is designed for local fast-access data, not long-term storage. It defaults to 15-day retention, loses data on node failure, and has no deduplication for HA replicas.
>
> The three solutions: Thanos adds a sidecar to each Prometheus that ships 2h blocks to S3; a central Querier fans out queries to all clusters and deduplicates HA replica data. Mimir is a horizontally scalable remote-write backend — Prometheus pushes to it, no sidecar needed, native multi-tenancy, built for scale. VictoriaMetrics is a single binary that replaces Prometheus's storage engine; most efficient compression, simplest ops, fast ingest.
>
> My pick depends on context: if the team already runs Prometheus and needs federation, I start with Thanos — it's additive, no re-architecture. If I'm starting fresh and want the simplest possible long-term storage, VictoriaMetrics wins — one binary, excellent compression, PromQL-compatible. If we're at scale (10M+ series), multi-tenant, and have a cloud budget, Mimir. The senior nuance: don't reach for Mimir from a small Prometheus setup. VictoriaMetrics is boring and exactly right for most teams."

---

## Self-test drills

### Drill 1 — architecture

A company runs 5 Prometheus instances across 3 Kubernetes clusters. They want a single Grafana dashboard showing data from all clusters with deduplication for their HA pair. What's the simplest architecture?

**Answer:** Thanos sidecar on each Prometheus + Thanos Store Gateway + Thanos Querier. Each Prometheus gets unique `external_labels` with `replica` and `cluster`. The Thanos Querier fans out queries to all sidecars and the store gateway, deduplicating on `replica` label. Grafana points to the Querier as its datasource.

### Drill 2 — deduplication

Two Prometheus replicas have external_labels `replica=a` and `replica=b`. Both scrape the same target and produce `http_requests_total{job="api", replica="a"}` and `http_requests_total{job="api", replica="b"}`. How does Thanos deduplicate them?

**Answer:** The Thanos Querier is started with `--deduplication.replica-label=replica`. When a query returns series that differ only in the `replica` label value, it merges them into a single series by taking the max value at each timestamp (configurable). The merged series has neither `replica=a` nor `replica=b` — the label is stripped.

### Drill 3 — choose your tool

A 5-person SRE team runs a single Kubernetes cluster with 300K series and wants 1-year retention. They're on-prem (no cloud managed service). What do you recommend?

**Answer:** VictoriaMetrics single binary. At 300K series, it's well within single-node capacity. On-prem means no managed Mimir. Thanos's operational complexity (8+ components) isn't justified for one cluster with a 5-person team. VictoriaMetrics takes an afternoon to set up, runs on a single VM, compresses well, and handles PromQL queries natively. Add `--retentionPeriod=12` for 1 year. Done.

---

## Further reading

- [Thanos architecture](https://thanos.io/tip/thanos/getting-started.md/)
- [Mimir architecture docs](https://grafana.com/docs/mimir/latest/references/architecture/)
- [VictoriaMetrics single-node docs](https://docs.victoriametrics.com/#single-node-version)
- [Cortex / Mimir vs Thanos — Banzai Cloud comparison (older but still relevant)](https://banzaicloud.com/blog/thanos-cortex/)
- [vmagent — VictoriaMetrics scraping agent](https://docs.victoriametrics.com/vmagent/)

---

## The 4 dimensions

- **Tech**: Thanos sidecar + S3 for additive federation; Mimir remote-write for multi-tenant cloud-native scale; VictoriaMetrics monolith for simplicity and compression. Deduplication via replica labels. Downsampling (5m, 1h) for multi-year query performance. Query federation via Thanos Querier fan-out or Grafana multi-datasource.
- **People**: VictoriaMetrics is the simplest to hand off — one binary, PromQL-compatible, one config file. Thanos requires understanding 8 components and their failure modes. Mimir requires Kubernetes expertise to operate. Match complexity to team size and maturity. New on-call engineers shouldn't need to understand distributed consensus on night 1.
- **CI/CD**: infrastructure for the storage tier is Terraform-managed (S3 bucket, IAM, StatefulSets). Prometheus remote-write config is part of the kube-prometheus-stack Helm values — version-controlled and reviewed. Retention config changes go through PR review; accidentally shortening retention is a data-loss event.
- **Operations**: monitor the storage tier itself: `thanos_store_api_*` metrics, VictoriaMetrics's `/metrics` page, Mimir ingester WAL replay duration on restart. Alert on object storage write failures (the S3 bucket is the long-term source of truth — write failures mean data loss). Retention policies set from day one; retrofitting shorter retention is politically hard after teams start relying on the data.
