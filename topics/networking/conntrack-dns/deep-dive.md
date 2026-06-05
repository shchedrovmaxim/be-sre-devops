# conntrack + UDP DNS — the deep-dive

> **Goal**: by the end you can answer the killer question — **"5% of pods get intermittent DNS NXDOMAIN under load. Walk through how you'd diagnose this."** — naming the conntrack insert_failed race condition, the kernel counters to check, the table-full failure mode, and NodeLocal DNS Cache as the canonical fix.

> Start with the [simple version](./simple.md) if you haven't. The coat-check analogy is the spine.

---

## The senior framing — conntrack is the hidden load-bearing primitive

netfilter/conntrack is the kernel module that makes kube-proxy work. It tracks every "connection" (including UDP flows) so that DNAT/SNAT can maintain state across packets. Every DNS query, every Service call, every pod-to-pod packet that crosses a NAT boundary goes through it.

When conntrack fails — either because the table is full or because the insert race condition fires — packets are silently dropped. No error in the app. No error in the kubelet. No warning in kube-proxy. The only evidence is in kernel counters that most monitoring setups don't expose.

The conntrack + DNS race condition was documented in the famous 2018 Weave Works blog post "Racy conntrack and DNS lookup timeouts." It's become one of the canonical K8s "senior SRE" problems because:
1. It's a kernel-level issue, not an app-level issue
2. It's invisible without knowing the right counters to check
3. The fix (NodeLocal DNS Cache) is non-obvious until you understand the root cause

---

## What conntrack actually is

### The data structure

conntrack is a hash table in kernel memory. Each entry represents a tracked "connection":

```
Key: {src_ip, src_port, dst_ip, dst_port, protocol}  → 5-tuple
Value: {state, timeout, DNAT/SNAT info, counters}
```

The table is bounded by two kernel parameters:
- `nf_conntrack_max`: maximum number of entries (default 65536 or 131072 on most distros; EKS default is often 131072)
- `nf_conntrack_buckets`: hash table bucket count (default `nf_conntrack_max / 4`)

The relationship: with 65536 max entries and 16384 buckets, each bucket holds ~4 entries on average. A full bucket requires chaining (linked list). Chaining degrades lookup performance.

### Checking current state

```bash
# On a node (via kubectl debug or node SSH):

# Current entries vs max
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Table statistics (one line per CPU; 8th column = insert_failed)
cat /proc/net/stat/nf_conntrack
# fields: entries  searched  found  new  invalid  ignore  delete  delete_list  insert  insert_failed  drop  early_drop  error  search_restart

# Or with the conntrack tool:
conntrack -S  # per-CPU stats, cleaner output
```

---

## The UDP state machine

TCP connections have clear state (SYN, ESTABLISHED, FIN_WAIT, etc.). UDP is stateless — but conntrack still tracks UDP flows using a simplified state machine:

```
[NEW] → packet sent, entry created, awaiting response
[UNREPLIED] → timeout starts; no response seen yet
[ASSURED] → response received, connection is "established"
[TIMEOUT] → no activity for nf_conntrack_udp_timeout seconds → entry removed
```

Default timeouts:
- `nf_conntrack_udp_timeout`: **30 seconds** (while unreplied or established)
- `nf_conntrack_udp_timeout_stream`: **180 seconds** (after seeing traffic in both directions — "assured")

For DNS queries:
- Query goes out → entry created in [NEW] state
- Response comes back in < 1 second → entry moves to [ASSURED], then removed after 180s
- If no response: stays [UNREPLIED] → removed after 30s

The 30-second unreplied timeout means failed DNS queries "hold" a conntrack entry for 30s. Under high DNS load with failures, you can accumulate thousands of UNREPLIED entries eating table capacity.

---

## The DNAT + UDP race condition — root cause

This is the 2018 Weave Works finding. Here's the exact scenario:

### The setup

kube-proxy uses DNAT to redirect Service ClusterIP traffic to pod IPs. For a DNS query:
```
Pod sends UDP to 10.96.0.10:53 (CoreDNS ClusterIP)
→ iptables PREROUTING: DNAT to 10.0.0.5:53 (a CoreDNS pod)
→ conntrack entry created: {src: pod-IP:1234, dst: 10.96.0.10:53, proto: UDP}
→ packet forwarded to 10.0.0.5:53
```

### The race condition

The problem occurs when two pods on the same node (or two goroutines in the same pod) send DNS queries simultaneously that hash to the same conntrack bucket:

```
Thread 1: DNS query A → hash bucket 42 → bucket empty → begin insert
Thread 2: DNS query B → hash bucket 42 → bucket empty → begin insert

Thread 1: insert entry A into bucket 42 → success
Thread 2: insert entry B into bucket 42 → bucket occupied → insert_failed
           → packet B silently dropped
```

Thread 2's DNS packet is dropped before it ever reaches iptables DNAT. The pod waits for a response that will never come, hits the DNS timeout (5 seconds), and retries. The retry might succeed (different hash value, or bucket is now available).

From the application's perspective: 5-second pause, then the lookup succeeds. Or it fails again and the app sees a DNS resolution error. This is the "intermittent NXDOMAIN under load."

### Why it's load-dependent

Under light load, the probability of two simultaneous queries landing on the same bucket is low. Under heavy load (many pods, high DNS query rate), the probability grows. The 2018 Weave post showed that on a busy node, the rate of `insert_failed` events correlated directly with DNS failure rate.

### The kernel counter

The smoking gun is `insert_failed` in `/proc/net/stat/nf_conntrack`:

```bash
# Sum insert_failed across all CPUs
awk '{sum += $9} END {print sum}' /proc/net/stat/nf_conntrack
# Also: column 8 in 0-indexed (9th field) — depends on kernel version

# Using conntrack tool (cleaner):
conntrack -S | grep insert_failed
```

If this counter is growing during a DNS failure incident, you've confirmed the race condition.

---

## The table-full failure mode

Separate from the race condition: if `nf_conntrack_count` reaches `nf_conntrack_max`, new packets that require conntrack entry creation are **dropped**.

```bash
# Check if you're near the limit:
echo "Current: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"
echo "Max: $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
# If current/max > 80%, you have a table pressure problem
```

Common causes in K8s:
- **Short-lived connections at high rate**: many pods making many short TCP/UDP connections; each connection holds an entry for its timeout period
- **UNREPLIED UDP entries accumulating**: failed DNS queries holding entries for 30s each
- **Default `nf_conntrack_max` too low**: some distros default to 65536; on a node with 100 pods each making 100 connections/sec, this fills in seconds

### Tuning

```bash
# Raise the max (on every node, requires privileged access):
sysctl -w net.netfilter.nf_conntrack_max=524288

# Also raise buckets to maintain hash distribution (~max/4):
sysctl -w net.netfilter.nf_conntrack_buckets=131072

# Reduce UDP timeout to free entries faster:
sysctl -w net.netfilter.nf_conntrack_udp_timeout=10     # from 30s
sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=60  # from 180s

# Persist in /etc/sysctl.d/ or via DaemonSet
```

**EKS note**: Amazon Linux 2 nodes have `nf_conntrack_max` set to `2 × nf_conntrack_buckets`. The default bucket count is 65536, so max starts at 131072. Under high pod density (100+ pods/node), this can fill. Use a DaemonSet with `privileged: true` to tune sysctl values.

---

## Canonical mitigations

### Fix 1 (best): NodeLocal DNS Cache

See [`coredns-nodelocal.md`](../coredns-nodelocal/deep-dive.md) for full details. Summary:
- Cache hits: served locally, no conntrack needed at all
- Cache misses: sent to CoreDNS over TCP — TCP avoids the UDP race condition
- Effect: eliminates `insert_failed` for DNS traffic; dramatically reduces conntrack table churn from DNS

This is the standard answer for any interview question about DNS reliability under load.

### Fix 2: `single-request` resolver option

A workaround added to glibc: send A and AAAA queries serially instead of in parallel. The race condition is partly triggered by parallel A+AAAA queries from the same socket hitting the same bucket.

```
# In /etc/resolv.conf:
options ndots:5 single-request
```

Or configure via CoreDNS rewrite rules, or per-application via `resolv.conf` in the pod spec.

**Limitation**: serial A+AAAA increases DNS lookup latency for external FQDNs. Not a complete fix — still race-prone with multi-pod load. It reduces frequency but doesn't eliminate the root cause.

### Fix 3: `single-request-reopen` (glibc option)

A variant that closes and reopens the UDP socket between A and AAAA queries — uses different source ports, different 5-tuples, avoids some collision patterns. Same limitation as `single-request`.

### Fix 4: raise `nf_conntrack_max`

Buy time. Appropriate as an emergency measure. Does not fix the race condition — only the table-full failure mode. Required if `nf_conntrack_count` is near `nf_conntrack_max`.

### Fix 5: host-network for critical workloads

Pods with `hostNetwork: true` use the node's network namespace. UDP queries from host-network pods don't go through kube-proxy DNAT — they query CoreDNS directly (or the VPC DNS resolver if not cluster DNS). No conntrack for the DNS path.

This is a drastic option with security implications (pod can see all node-level network traffic). Use only for critical system components (monitoring agents, security tools) where DNS reliability is paramount and security isolation is managed separately.

---

## Diagnosis flow — the full step-by-step

For the interview question "5% of pods get intermittent DNS NXDOMAIN under load":

### Step 1: Confirm it's DNS

```bash
# From an affected pod:
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod> -- dig +short kubernetes.default.svc.cluster.local
# If NXDOMAIN: confirmed DNS issue
# If timeout: could be DNS or network; check with -timeout flag
```

### Step 2: Check which nodes have the problem

```bash
# Schedule test pods on each node and measure failure rate
for node in $(kubectl get nodes -o name); do
  kubectl run dns-test-${node##*/} \
    --image=busybox \
    --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"'${node##*/}'"}}}' \
    --restart=Never \
    --rm -it -- sh -c "for i in $(seq 100); do nslookup kubernetes.default > /dev/null 2>&1 || echo FAIL; done"
done
```

### Step 3: Check conntrack on the affected node

```bash
# Via kubectl debug (K8s 1.23+):
kubectl debug node/<node-name> -it --image=ubuntu -- bash

# Inside the debug pod:
cat /proc/net/stat/nf_conntrack
# Column 9 (0-indexed: insert_failed) growing? → race condition confirmed

cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# count > 80% of max? → table pressure
```

### Step 4: Check CoreDNS metrics

```bash
# SERVFAIL rate
kubectl exec -n kube-system <coredns-pod> -- \
  curl -s localhost:9153/metrics | grep 'coredns_dns_responses_total{.*SERVFAIL'
```

### Step 5: Check NodeLocal is installed (or not)

```bash
kubectl get pods -n kube-system -l k8s-app=node-local-dns
# Not present → deploy NodeLocal
# Present but not on all nodes → DaemonSet scheduling issue
```

### Step 6: Deploy NodeLocal

Standard solution. See coredns-nodelocal.md for deployment details.

### Step 7: Emergency stop-gap

If NodeLocal can't be deployed immediately:
```bash
# Raise conntrack max via DaemonSet (applies on all nodes without SSH):
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: conntrack-tuner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: conntrack-tuner
  template:
    metadata:
      labels:
        app: conntrack-tuner
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: tuner
        image: busybox
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          sysctl -w net.netfilter.nf_conntrack_max=524288
          sysctl -w net.netfilter.nf_conntrack_udp_timeout=15
          sleep infinity
EOF
```

---

## Prometheus monitoring

```promql
# Conntrack table fullness (%)
node_nf_conntrack_entries / node_nf_conntrack_entries_limit * 100
# Alert if > 80%

# insert_failed — requires node exporter textfile collector or custom metric
# Check your node exporter for nf_conntrack_stat metrics

# DNS NXDOMAIN rate from CoreDNS
rate(coredns_dns_responses_total{rcode="NXDOMAIN"}[1m])

# DNS error rate (SERVFAIL = upstream problem; NXDOMAIN = name not found)
rate(coredns_dns_responses_total{rcode=~"SERVFAIL|REFUSED"}[1m])
```

---

## The interview answer in 60 seconds

> "I'd diagnose this in layers. First, confirm it's truly NXDOMAIN and not connection refused — run `dig` from the pod. Then check if it's load-correlated and node-specific: that pattern is classic conntrack.
>
> On the affected node, I'd check two kernel counters. First, `insert_failed` in `/proc/net/stat/nf_conntrack`. If that's non-zero and growing, I've found the race condition: two goroutines simultaneously inserting a UDP conntrack entry for the same 5-tuple — the second insert fails and the packet is dropped before it reaches iptables DNAT. Second, `nf_conntrack_count` vs `nf_conntrack_max` — if count is near max, new DNS packets are being dropped because the table is full.
>
> The canonical fix is NodeLocal DNS Cache. It puts a caching DNS daemon on each node listening on `169.254.20.10`. Cache hits never touch conntrack. Cache misses go to CoreDNS over TCP — TCP avoids the UDP race condition. Deploying NodeLocal typically eliminates the intermittent failures entirely.
>
> Stop-gap while deploying NodeLocal: raise `nf_conntrack_max` to 524288 and lower `nf_conntrack_udp_timeout` to 15 seconds to free entries faster. That buys time but doesn't fix the race condition."

---

## Self-test drills

### 1. 5% of pods get intermittent DNS NXDOMAIN under load. Walk through how you'd diagnose this.

**Reference answer:** Confirm DNS NXDOMAIN (not timeout/refused). Check load-correlation and node-specificity. On affected node: check `insert_failed` in `/proc/net/stat/nf_conntrack` (race condition) and `nf_conntrack_count` vs `nf_conntrack_max` (table full). Fix: NodeLocal DNS Cache — cache hits bypass conntrack, cache misses use TCP. Stop-gap: raise `nf_conntrack_max`. See the interview answer section above for full formulation.

### 2. What is the conntrack insert_failed race condition?

**Reference answer:** When two UDP packets from different goroutines/pods hash to the same conntrack bucket simultaneously, both threads see the bucket as empty and begin inserting. Thread 1 inserts first, Thread 2's insert fails (`insert_failed` counter increments), Thread 2's packet is silently dropped before reaching iptables. No error is surfaced to the application; the pod just doesn't get a DNS response. The glibc `single-request` option reduces frequency (makes A/AAAA queries serial) but doesn't eliminate the race. NodeLocal's TCP upstream path eliminates it for DNS traffic.

### 3. How do you tune the conntrack table and what are the relevant kernel parameters?

**Reference answer:** `nf_conntrack_max` — maximum entries (raise to 524288 for busy nodes). `nf_conntrack_buckets` — hash buckets (set to `max/4` for good distribution). `nf_conntrack_udp_timeout` — how long UNREPLIED UDP entries live (default 30s; lower to 10-15s to reclaim entries faster). `nf_conntrack_udp_timeout_stream` — how long ASSURED UDP entries live (default 180s; lower if you have many short UDP flows). Apply via `sysctl` in a privileged DaemonSet for cluster-wide effect.

### 4. What is `single-request` in `/etc/resolv.conf` and when does it help?

**Reference answer:** `single-request` makes glibc's resolver send A and AAAA DNS queries serially instead of in parallel from the same UDP socket. The conntrack race is partly triggered by parallel A+AAAA queries sharing a socket (same source port) colliding in the conntrack hash. Serial queries reduce collision probability. However: it increases DNS lookup latency for external names (2 sequential queries instead of 1 parallel pair), and it doesn't eliminate the race for multi-pod load. It's a partial mitigation, not a fix. NodeLocal DNS Cache is the fix.

---

## Further reading / watching

- **Weave Works — "Racy conntrack and DNS lookup timeouts"** (2018): the original post; search "weave works racy conntrack dns" — mandatory reading for any SRE dealing with K8s DNS
- **Linux netfilter project — conntrack documentation**: `netfilter.org` — the authoritative kernel docs on conntrack; understand the `nf_conntrack_stat` fields
- **Brendan Gregg — "Linux Performance Tools, Brendan Gregg" talks** on YouTube: not conntrack-specific, but his methodology for kernel performance investigation applies directly to conntrack diagnosis
- **Cilium blog — "BPF and XDP reference guide"**: explains how Cilium's eBPF path bypasses conntrack for certain traffic paths — the "why eBPF avoids this" angle
- **Kubernetes GitHub issue #56903** — "DNS lookup timeouts in 5.0": one of the canonical issues documenting the problem and the community's diagnosis process; search "kubernetes github 56903 DNS conntrack"

---

## The 4 dimensions (senior framing)

- **Tech**: `nf_conntrack_max` tuning; `insert_failed` as the diagnostic counter; UDP state machine and timeout tuning; NodeLocal TCP upstream as the race condition fix; `single-request` as a partial mitigation; host-network for critical system pods.
- **People**: this failure mode is invisible to application developers — their DNS calls just occasionally take 5 seconds. Document the diagnosis steps in a shared runbook so any SRE on-call can hit the right kernel counters within 5 minutes of a DNS incident. "Check conntrack before blaming the app" should be a first-instinct reflex on the team.
- **CI/CD**: add conntrack monitoring to cluster health checks. Alert on `node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8` before it becomes an incident. As part of cluster provisioning automation, configure the recommended `nf_conntrack_max` for your expected pod density. Include NodeLocal DNS Cache in the baseline cluster configuration — it should be on from day one, not added after the first incident.
- **Operations**: `nf_conntrack_max` tuning is node-kernel state — not visible in kubectl, not in CloudWatch, requires node-level access. Make it a standard part of node bootstrap automation. Alert on `insert_failed` rate > 0 per minute (should be near zero on a healthy cluster with NodeLocal). Runbook: "intermittent DNS NXDOMAIN" → step 1: check conntrack counters → step 2: check NodeLocal pods on all nodes → step 3: raise `nf_conntrack_max` if needed → step 4: deploy NodeLocal.
