# Long-term Prometheus storage — the simple version (the filing room analogy)

> Read this first. Once the hot-desk vs archive-room analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **Prometheus is great at keeping the last 15 days of data in a fast local filing cabinet. But it was never designed to be your multi-year archive. Thanos, Mimir, and VictoriaMetrics are the archive rooms.**

---

## Prometheus is a fast hot-desk, not an archive room

Prometheus keeps data locally on disk, defaulting to **15 days retention**. That local storage is fast — queries are sub-second. But:

- If a node dies, data is lost.
- If you have 5 Prometheus replicas for HA, you have 5 copies of the same data with no deduplication.
- You can't query data older than 15 days without digging through backups.

The solutions all solve the same problem: **persist Prometheus data to object storage (S3/GCS) so it survives forever and is queryable across all replicas.**

---

## The three players — in one sentence each

| Tool | The one-sentence pitch |
|---|---|
| **Thanos** | Sidecars ship data from each Prometheus to S3; a central Querier merges results and deduplicates across HA replicas. Most flexible, most moving parts. |
| **Mimir** | A drop-in Prometheus-compatible remote-write target that shards, stores in S3, and scales horizontally. Cloud-native microservices, fully managed by Grafana Labs. |
| **VictoriaMetrics** | A single binary that replaces Prometheus's storage engine. Simplest ops, fastest ingest, excellent compression, good enough for most teams. |

---

## The pick-one cheat sheet

| Situation | Recommended choice |
|---|---|
| Already running Prometheus, want to add long-term retention without re-architecting | **Thanos sidecar** |
| Need global query view across many clusters, or query federation | **Thanos** or **Mimir** |
| Small-to-medium team, simplicity is the priority | **VictoriaMetrics** |
| Running on-prem or air-gapped, need minimal dependencies | **VictoriaMetrics** |
| Large scale (10M+ series), multi-tenant, fully managed, cloud budget | **Mimir** |

---

## What deduplication means and why it matters

If you run 2 Prometheus replicas for HA (so one can die without losing scraping), both replicas scrape the same targets and produce the same metrics — with slightly different timestamps.

When you query, you'd see duplicated data points. Thanos and Mimir deduplicate this by merging series with the same labels and aligning timestamps during query time. Without deduplication, your graphs show doubled counters.

---

## Downsampling — why you need it for long-term data

Raw samples every 15 or 30 seconds is great for the last week. But querying 2 years of data at 15-second resolution across 500K series is brutally slow.

Thanos (and Mimir) automatically downsample old data:
- After 40 hours: aggregate to 5-minute resolution
- After 10 days: aggregate to 1-hour resolution

A 2-year latency graph uses 1-hour samples — still useful for trend analysis, and queries finish in under a second.

---

## Self-test (out loud)

> **"When would you pick Mimir vs Thanos vs VictoriaMetrics for long-term Prometheus storage?"**

**Reference answer:**

"If the team already runs Prometheus and just needs long-term retention, I'd add Thanos as the simplest path — a sidecar per Prometheus, an S3 bucket, a Querier. No re-architecting.

If we need a simple, low-ops solution and aren't already deeply invested in the Thanos ecosystem, VictoriaMetrics is my first choice — single binary, very fast, handles 5M+ series comfortably, and its remote-write API is Prometheus-compatible.

If we're at scale (10M+ series), multi-tenant, and the team can operate Kubernetes microservices well, Mimir is the cloud-native answer. Grafana Labs offers a managed version, which removes the ops burden.

The senior nuance: don't jump to Mimir or Thanos from a small Prometheus setup for bragging rights. VictoriaMetrics is boring and exactly right for most teams."

---

## Further reading

- [Thanos docs](https://thanos.io/tip/thanos/getting-started.md/)
- [Mimir docs](https://grafana.com/docs/mimir/latest/)
- [VictoriaMetrics docs](https://docs.victoriametrics.com/)
- [Grafana blog — Thanos vs Mimir vs VictoriaMetrics](https://grafana.com/blog/2021/05/12/the-future-of-prometheus-long-term-storage/)

---

## Next: the deep-dive

When the hot-desk vs archive-room analogy clicks, jump to [`long-term-storage.md`](./deep-dive.md). The deep-dive covers:

- Thanos's full architecture (sidecar, store gateway, compactor, querier, ruler)
- Mimir's microservices and the trade-offs vs Thanos
- VictoriaMetrics's monolith design and why it's often faster
- Deduplication mechanics in detail
- Downsampling configuration
- The query federation pattern
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
