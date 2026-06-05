# Loki + label cardinality — the simple version (the library analogy)

> Read this first. Once the library catalogue analogy clicks, the [deep-dive](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **Loki's promise is "shove logs in cheaply" — but only if you give it a small, stable label set. Add the wrong labels and the catalogue explodes, queries slow to a crawl, and costs skyrocket.**

---

## Loki is a library with a tiny card catalogue

A traditional library (Elasticsearch, Splunk) reads every book and indexes every word inside it. You can search for any word instantly, but building and maintaining that index is expensive — you need a big server room.

Loki is a library that **only indexes the labels on the book spine**, not the contents. To find text inside a book, you pull all books matching the spine label and read them yourself. This is fast enough for most use cases and the catalogue stays tiny.

| Library analogy | Loki world |
|---|---|
| Book spine labels | **Stream labels** — `{app="checkout", namespace="production"}` |
| Library catalogue (indexed) | Loki's **index** — only labels, not log content |
| Book contents | The actual log lines (stored compressed in object storage) |
| Pulling books matching a spine label | Label-based query: `{app="checkout"}` |
| Reading inside a book for a keyword | `| grep "timeout"` — filtering at query time |

The index is **tiny** because it only holds labels and which files contain those streams. That's why Loki is cheap.

---

## The cost of one bad label

If you put `request_id` as a Loki stream label:

```
{app="checkout", namespace="production", request_id="a3f9b2c1-00fa..."}
{app="checkout", namespace="production", request_id="b7d42e3f-11ab..."}
{app="checkout", namespace="production", request_id="c1e55a4b-22bc..."}
... × 1 million requests per day
```

One million streams per day. Each stream is a row in the index. Loki ingests billions of tiny log streams instead of a few large ones — the index explodes, ingest slows, queries become unusably slow.

The correct approach: `request_id` is not a label. It's a field in the log line.

```
{app="checkout", namespace="production"}
2026-06-05 12:34:56 {"level":"info","request_id":"a3f9b2c1","msg":"order created"}
```

To query by `request_id`:

```logql
{app="checkout"} | json | request_id="a3f9b2c1"
```

This pulls the `checkout` stream (one stream, not a million) and filters at query time. Slower than an index lookup, but the index stays small and the cost is manageable.

---

## The golden rule for Loki labels

> **Labels are for what you'd GROUP or FILTER streams by. Not for what you'd search for inside a log line.**

Good stream labels (low cardinality, stable, used to filter whole streams):
- `app` — service name
- `namespace` — Kubernetes namespace
- `env` — production, staging
- `container` — container name within a pod
- `region` — cloud region

Bad stream labels (high cardinality, per-request values):
- `request_id`, `trace_id`, `session_id`
- `user_id`, `email`
- `status_code` (if you have thousands of routes, status code adds cardinality)
- `pod` or `pod_name` (potentially hundreds of pods → hundreds of streams per service)

The `pod` label is a common gotcha — in a service with 50 pod replicas, adding `pod` as a stream label creates 50 streams per service instead of 1. When pods roll over, old streams are left behind. Consider whether you actually need pod-level stream splitting or can achieve the same with `| json | pod="..."` at query time.

---

## Structured logging — the practical solution

If your log lines are JSON, LogQL can extract any field at query time without indexing it:

```bash
# Application logs JSON
{"level":"error","service":"checkout","user_id":"12345","msg":"payment failed","trace_id":"abc"}

# LogQL to find errors for a specific user
{app="checkout", namespace="production"} | json | user_id="12345" | level="error"
```

No `user_id` label on the stream. The field is extracted at query time from the JSON body. This is the pattern that makes Loki's cost model work.

---

## Self-test (out loud)

> **"Why is Loki's promise of 'just shove logs in cheaply' contingent on label discipline, and what happens if you screw it up?"**

**Reference answer:**

"Loki indexes stream labels, not log content. That's the cost win — a tiny index, logs stored cheaply in object storage. But if you put high-cardinality values like `request_id` or `user_id` as stream labels, you create one stream per unique value — millions of streams per day. The index explodes, ingest queues back up, queries become unusably slow, and storage costs jump.

The fix is to keep stream labels coarse and stable — `app`, `namespace`, `env` — and put everything else in the log line body as structured JSON fields. LogQL's `| json` or `| logfmt` extracts those fields at query time. Slower than a label lookup, but the index stays small and cheap. The rule: if you'd use it to find a specific log line (user_id, trace_id), it's a log field. If you'd use it to select a whole stream (all logs from the checkout service in production), it's a label."

---

## Further reading

- [Loki label best practices](https://grafana.com/docs/loki/latest/get-started/labels/best-practices/)
- [LogQL reference](https://grafana.com/docs/loki/latest/query/)
- [Loki cardinality blog](https://grafana.com/blog/2021/08/09/new-in-loki-2.3-log-queries-just-got-easier-with-logql-v2/)

---

## Next: the deep-dive

When the library analogy is obvious, jump to [`loki-cardinality.md`](./deep-dive.md). The deep-dive covers:

- Loki's storage model (streams, chunks, index) in detail
- The `chunks_per_stream` metric as a cardinality canary
- LogQL pattern extraction, `| logfmt`, `| json`, `| regexp`
- The `limits_config` ingestion rate limits and cardinality limits
- Detecting cardinality issues via the Loki API
- Structured logging patterns (Go, Node.js, Python examples)
- 3 drills, the 60-second interview answer, and the 4-dimensions framing
