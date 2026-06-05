# Cardinality discipline — the simple version (the filing cabinet analogy)

> Read this first. Once the "one drawer per unique combination" analogy clicks, the [deep-dive](./cardinality-discipline.md) becomes easy.

This doc explains **one idea**:

> **Cardinality is the number of drawers in your filing cabinet. Every unique combination of metric name + label values needs its own drawer. Add a label with 100,000 possible values, and you need 100,000 drawers — Prometheus runs out of memory.**

---

## Your metrics are a filing cabinet

Imagine Prometheus is a filing cabinet. Every time series is one drawer. Each drawer is labeled with a metric name + all its label values.

```
Drawer 1: http_requests_total  job=checkout  status=200  method=GET
Drawer 2: http_requests_total  job=checkout  status=500  method=GET
Drawer 3: http_requests_total  job=checkout  status=200  method=POST
...
```

3 labels with 3 values each = 3 × 3 × 3 = **27 drawers**. Manageable.

Now add a `user_id` label with 1 million users:

```
http_requests_total  job=checkout  status=200  method=GET  user_id=00000001
http_requests_total  job=checkout  status=200  method=GET  user_id=00000002
http_requests_total  job=checkout  status=200  method=GET  user_id=00000003
... × 1,000,000 users × 6 status/method combos
```

That's **6 million drawers** — and Prometheus stores every one of them in memory. At ~1KB per time series in memory overhead, that's **6 GB just for that one metric**. Prometheus OOMs.

This is called a **cardinality explosion**.

---

## The golden rule

> **Labels are for what you'd group or filter by. Not for high-resolution identifiers.**

Good labels (low cardinality, stable set of values):
- `job` — handful of services
- `status` — 5xx, 4xx, 2xx (~5 values)
- `method` — GET, POST, PUT (~6 values)
- `region` — us-east-1, eu-west-1 (~10 values)
- `env` — production, staging (~3 values)

Bad labels (high cardinality, unbounded):
- `user_id` — millions of users
- `request_id` — every request has a unique one
- `session_id` — unbounded
- `email` — unbounded
- `ip_address` — unbounded

The test: "Will the number of distinct values for this label grow over time without bound?" If yes, it's not a label — it's a log field.

---

## What Prometheus looks like when it's dying from cardinality

1. Memory usage trends up indefinitely (not leveling off)
2. Query latency increases (more series to scan)
3. Eventually: OOMKilled
4. After restart: restores from WAL, OOMs again immediately

At this point you need to find the culprit and drop the offending label.

---

## How to find the culprit (the one-liner)

```promql
# Top 10 metric families by series count
topk(10,
  count by (__name__) ({__name__=~".+"})
)
```

This returns the 10 metric names with the most time series. If `http_requests_total` shows 2 million series and everything else shows 100, you found your problem.

Then drill in:

```promql
# What label values does this metric have?
count by (user_id) (http_requests_total)
```

If this returns 1 million rows, `user_id` is your cardinality killer.

---

## The fix: drop the label via relabeling

Once identified, drop the bad label in the scrape config so it never enters the TSDB:

```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: "user_id"
```

The data isn't lost — move user-level analysis to your **log system** (Loki, CloudWatch Logs) where each line doesn't need its own indexed series.

---

## Self-test (out loud)

> **"Your Prometheus is OOMing. What does 'cardinality explosion' mean and how would you find the culprit?"**

**Reference answer:**

"Cardinality is the number of time series — each unique combination of metric name plus label values is one series stored in memory. A cardinality explosion happens when a label with a huge number of distinct values gets added — like user_id on a request metric — creating millions of series where there were thousands before.

To find it:
```promql
topk(10, count by (__name__) ({__name__=~".+"}))
```

This shows the 10 metric families with the most series. Then I'd drill into the top offender and count by each label to find which label value has unbounded cardinality.

The fix is to drop that label via `metric_relabel_configs` with `action: labeldrop` before it enters the TSDB. For user-level data, I'd use structured logs in Loki instead — where you can filter by user_id at query time without indexing it."

---

## Further reading

- [Prometheus cardinality docs](https://prometheus.io/docs/practices/naming/#labels)
- [Grafana cardinality blog post](https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/)
- [Robust Perception — cardinality is a feature](https://www.robustperception.io/cardinality-is-key/)

---

## Next: the deep-dive

When the filing-cabinet analogy is obvious, jump to [`cardinality-discipline.md`](./cardinality-discipline.md). The deep-dive covers:

- Exact numbers: what explodes at 100K series, 1M series, and 10M series
- The TSDB admin API for live cardinality investigation
- Grafana Mimirtool and the cardinality API
- The Loki angle (same lesson for log labels)
- Proactive cardinality governance (PR checks, CI enforcement)
- 4 drills, the 60-second interview answer, and the 4-dimensions framing
