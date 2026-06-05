# Endpoints vs EndpointSlices — the simple version (the office directory analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **EndpointSlices fix a scalability problem: the old system put every worker's address in a single shared document. Changing one address meant rewriting and re-sending the entire document to every office.**

---

## The office directory that doesn't scale

Imagine your company has 5,000 employees. Every time someone joins or leaves, HR updates a single **company-wide directory** — one massive spreadsheet — and emails it to every manager in every office building so they can update their local copy.

Your company hires and fires a lot (pods come and go frequently). Every hire or fire means:

1. HR edits the single giant spreadsheet.
2. HR emails the full spreadsheet to all 500 managers (nodes).
3. Every manager downloads the entire thing and replaces their local copy.

Even if just **one person** changed desks, every manager gets the whole 5,000-row spreadsheet again. That's O(N²) network traffic.

This is exactly what the old Kubernetes **Endpoints object** did.

---

## The fix: split the directory into pages

EndpointSlices are like splitting that directory into **small booklets of 100 people each**. Now:

- When someone joins desk 47, only the booklet containing desk 47 gets updated and re-sent.
- The 500 managers only download the one changed booklet.
- 50 booklets total → each update touches at most 1 booklet.

That's bounded, O(1)-ish update cost regardless of how many people (pods) you have.

| Old way (Endpoints) | New way (EndpointSlices) |
|---|---|
| 1 giant directory per service | Many small slices (100 pods per slice, by default) |
| 1 pod IP changes → full object sent to all nodes | 1 pod IP changes → 1 slice sent to affected nodes |
| Fine at 10 pods per service | Falls apart at 100+ pods |
| O(N²) update cost at scale | O(1)-ish per slice change |

---

## How they look in practice

The old Endpoints object (one object, all IPs together):

```yaml
# kubectl get endpoints my-service
apiVersion: v1
kind: Endpoints
subsets:
- addresses:
  - ip: 10.0.1.1
  - ip: 10.0.1.2
  # ... 998 more IPs for a 1000-pod service
  ports:
  - port: 8080
```

An EndpointSlice (one of many, each capped at 100 endpoints):

```yaml
# kubectl get endpointslices -l kubernetes.io/service-name=my-service
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
addressType: IPv4
endpoints:
- addresses: ["10.0.1.1"]
  conditions:
    ready: true
- addresses: ["10.0.1.2"]
  conditions:
    ready: true
# ... up to 100 per slice
ports:
- port: 8080
```

---

## Who uses EndpointSlices?

kube-proxy, ingress controllers (NGINX, Envoy), and service meshes (Istio, Linkerd) all consume EndpointSlices directly. The old Endpoints objects are still generated for backward compatibility (via the `EndpointSliceMirroring` controller), but modern consumers prefer slices.

You don't normally write EndpointSlices by hand — the controller creates and manages them automatically.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| Why were EndpointSlices introduced? | The single Endpoints object for a Service had to be fully rewritten and re-synced to all nodes on every pod change. At scale (1000+ pods), this caused massive API-server and kube-proxy CPU spikes. |
| How many endpoints per slice? | 100 by default (configurable). Each change touches at most 1 slice. |
| Do I have to do anything to use EndpointSlices? | No. Modern K8s clusters (1.21+) use them automatically. kube-proxy consumes them by default. |
| What happened to the old Endpoints object? | It still exists. The `EndpointSliceMirroring` controller keeps it in sync for backward compatibility. But it's the legacy path. |
| Can I still use `kubectl get endpoints`? | Yes, and it still works. But `kubectl get endpointslices` gives you the ground truth. |

---

## Self-test (one question — the killer one)

Out loud:

> **"Why did Kubernetes introduce EndpointSlices, and what scale problem do they solve?"**

**Reference answer (intuitive version):**

"The original Endpoints object stored every pod IP for a Service in a single API object. Every time any pod changed — new pod, dying pod, failed readiness probe — the entire object had to be rewritten and then synced to kube-proxy on every node in the cluster. With a large Service (hundreds of pods) and frequent pod churn, this created O(N²) update traffic: N pod changes each triggering a full N-endpoint object update to all nodes. The API server and kube-proxy both got hammered.

EndpointSlices shard each Service's endpoints into small objects, capped at 100 endpoints each by default. A single pod IP change now only updates the one slice containing that pod. The rest of the slices are untouched. The update is bounded regardless of how many pods the Service has."

---

## Further reading

- [K8s docs — EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Deep-dive: endpoints-endpointslices.md](./deep-dive.md) — covers topology-aware routing, the mirroring controller, how kube-proxy consumes slices, and what happens to in-flight connections on slice updates
