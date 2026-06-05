# Cert rotation + monitoring expiry — the deep-dive

> **Goal**: by the end you can answer the killer question — **"How would you detect and prevent cert expiry outages across hundreds of services?"** — naming the dual-layer monitoring strategy (blackbox exporter + cert-manager metrics), cert-manager renewal mechanics, manual rotation patterns, client-side pinning gotchas, and the mTLS client cert rotation choreography.

> Start with [cert-rotation-simple.md](./cert-rotation-simple.md) if you haven't read it. The passport analogy is the spine of everything here.

---

## The senior framing — cert expiry is always a "known unknown"

The date is on the cert. You knew it was going to expire. The failure isn't technical — it's operational: no one built an alert that fired early enough, no one was responsible for the renewal, or the renewal automation silently failed without anyone noticing.

The canonical example is the **LinkedIn outage pattern**: a cert nobody was actively watching expired, causing TLS handshake failures, which caused cascading service failures that looked (initially) like a database problem. Root cause analysis: cert expiry. Fix time: hours, because figuring out "which cert" took most of the time.

The senior answer isn't just "use cert-manager." It's knowing that cert-manager is necessary but not sufficient, understanding the monitoring that complements it, and designing the rotation choreography for the tricky cases (mTLS clients, certificate pinning).

---

## cert-manager renewal mechanics — how it actually works

### The renewal loop

cert-manager runs a reconcile loop on `Certificate` objects. The loop evaluates:

```
if cert is missing or expired:
    trigger immediate issuance
elif cert.notAfter - now <= renewBefore:
    trigger renewal
else:
    schedule next check at (cert.notAfter - renewBefore)
```

The default reconcile frequency is driven by Kubernetes controller-manager re-queue logic — effectively every few minutes for active objects, and the renewal is triggered when the `renewBefore` threshold is crossed.

**Default `renewBefore`**: if not set, cert-manager uses 1/3 of the cert's `duration`. For a 90-day Let's Encrypt cert: 30 days. Renewal starts at day 60 of the 90-day window.

**Why 30 days matters**: this is your recovery window. If renewal fails on day 60, you have 30 days to:
1. Notice (your alert at < 30 days remaining fires).
2. Diagnose (Let's Encrypt down? DNS credentials wrong? Rate limit hit?).
3. Fix and re-trigger.

Don't set `renewBefore: 24h`. That gives you 24 hours to notice and fix. You might not even see the PagerDuty alert for 4 hours.

### What `cmctl renew` does

Forces an immediate renewal regardless of where you are in the cert's lifecycle:

```bash
cmctl renew api-tls -n production
```

This creates a new `CertificateRequest`. The old cert stays in the Secret until the new one is ready, then the Secret is updated atomically. Use this after:
- Suspected private key compromise
- CA root rotation
- Emergency rotation before any alerting threshold

**What it does NOT do**: it doesn't revoke the old cert. The old cert remains valid until its `notAfter`. If you need to revoke (compromised private key), you also need to explicitly revoke via the CA.

### The Secret update — how pods pick it up

When cert-manager renews a cert and updates the Secret, pods that mount the Secret as a volume get updated files automatically (kubelet syncs volume contents within ~1 minute). BUT: the application must re-read the files.

Most TLS libraries don't watch files — they load the cert at startup and hold it in memory. This means after renewal, the pod is still serving the old cert until it restarts.

**Solutions**:
1. **Stakater Reloader** — watches Secrets, triggers `Deployment` rollout when content changes. Most common pattern.
2. **Custom SIGHUP handler** — your application listens for SIGHUP and reloads TLS config without restarting.
3. **Short TTL + scheduled restart** — for short-lived certs (1h Vault certs), the pod lifecycle is shorter than the cert TTL if you redeploy frequently enough.
4. **Envoy sidecar / service mesh** — Envoy's xDS dynamic cert config picks up cert changes via the SDS (Secret Discovery Service) API — no pod restart needed. This is what Istio does.

---

## Monitoring layer 1: Prometheus blackbox exporter

The blackbox exporter makes real TLS connections to your HTTPS endpoints and reports the cert's expiry time as a Prometheus metric. It's the most important monitoring tool for cert expiry because:

- It's end-to-end: it catches cert problems whether cert-manager manages the cert or not.
- It simulates what your users experience — if the cert check fails, your users also get TLS errors.
- It works for any TLS endpoint: CDN edge, external load balancer, database, internal service mesh.

### Configuration

```yaml
# blackbox-exporter config
modules:
  https_cert_check:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      tls_config:
        insecure_skip_verify: false    # must be false to check the cert chain
      preferred_ip_protocol: "ip4"
```

### Prometheus scrape config

```yaml
- job_name: 'blackbox-tls-expiry'
  metrics_path: /probe
  params:
    module: [https_cert_check]
  static_configs:
    - targets:
        - https://api.example.com
        - https://app.example.com
        - https://internal-service.payments.svc.cluster.local:8443
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter.monitoring.svc.cluster.local:9115
```

### Key metrics and alert rules

```yaml
# The core metric: seconds until cert expiry
probe_ssl_earliest_cert_expiry

# Alert rule: < 30 days remaining (ticket-level)
- alert: CertExpiryWarning
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "TLS cert for {{ $labels.instance }} expires in < 30 days"
    description: "{{ $labels.instance }} cert expires in {{ $value | humanizeDuration }}"

# Alert rule: < 7 days remaining (urgent)
- alert: CertExpiryUrgent
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "TLS cert for {{ $labels.instance }} expires in < 7 days"

# Alert rule: < 24 hours remaining (P1 — wake someone up)
- alert: CertExpiryImminent
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 1
  for: 1m
  labels:
    severity: page
  annotations:
    summary: "TLS cert for {{ $labels.instance }} expires in < 24 hours"
```

**The 30/7/1 alert ladder** gives you:
- 30 days: "team, this cert needs attention; renewal automation might be broken"
- 7 days: "escalate to senior SRE; renewal is definitely broken and needs manual intervention"
- 1 day: "wake someone up"

---

## Monitoring layer 2: cert-manager controller metrics

cert-manager exposes Prometheus metrics about its own view of `Certificate` objects:

| Metric | What it measures |
|---|---|
| `certmanager_certificate_expiration_timestamp_seconds` | Unix timestamp when the cert in the Secret expires (cert-manager's view) |
| `certmanager_certificate_ready_status` | 1 if Certificate is Ready, 0 if not |
| `certmanager_certificate_renewal_timestamp_seconds` | Unix timestamp when cert-manager plans to renew |
| `certmanager_http_acme_client_request_count` | ACME API requests made (useful for rate limit debugging) |
| `certmanager_clock_time_seconds` | cert-manager's clock (useful for debugging time drift issues) |

```yaml
# Alert: Certificate not Ready
- alert: CertManagerCertNotReady
  expr: certmanager_certificate_ready_status{condition="False"} == 1
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "cert-manager Certificate {{ $labels.name }} in {{ $labels.namespace }} is not Ready"

# Alert: cert-manager's renewal failed and expiry is approaching
- alert: CertManagerRenewalFailedWithExpiry
  expr: |
    (certmanager_certificate_expiration_timestamp_seconds - time()) < 7 * 86400
    AND certmanager_certificate_ready_status{condition="False"} == 1
  for: 5m
  labels:
    severity: critical
```

**Why both layers?**
- cert-manager metrics only cover `Certificate` objects cert-manager manages. CDN certs, external LBs, database certs — cert-manager doesn't know about them.
- Blackbox exporter catches everything (any TLS endpoint you probe), but only tells you the cert is expiring — not why cert-manager isn't renewing it.
- Together: blackbox is the early warning for all certs; cert-manager metrics are the diagnostic tool when you're investigating a specific failure.

---

## Manual rotation patterns

Sometimes automation isn't available:
- A cert issued outside cert-manager (CDN, external service, legacy DB).
- cert-manager is broken and needs emergency manual intervention.
- A CA root rotation that requires coordination.

### Manual rotation checklist

```
1. Generate new private key and CSR (or use the CA's web UI).
2. Submit CSR to CA, receive signed cert.
3. Verify the new cert:
   - openssl verify -CAfile ca-chain.pem newcert.pem
   - openssl x509 -in newcert.pem -noout -dates
   - openssl x509 -in newcert.pem -noout -subject -ext subjectAltName
4. Update the Secret/config with the new cert + key.
5. Trigger rolling restart of affected Deployments.
6. Verify with blackbox exporter / openssl s_client:
   - echo | openssl s_client -connect api.example.com:443 2>/dev/null | openssl x509 -noout -dates
7. Monitor error rates for 15 minutes post-rotation.
8. Update any out-of-band tracking (spreadsheet, Vault, notes).
```

### Force cmctl renew when automation is stuck

```bash
# Force renewal now, regardless of renewBefore
cmctl renew <cert-name> -n <namespace>

# Check status after
cmctl status certificate <cert-name> -n <namespace>

# If cert-manager itself is the issue, restart it
kubectl rollout restart deployment cert-manager -n cert-manager
```

---

## Client-side pinning gotchas

Certificate pinning is when a client stores the exact cert (or its fingerprint / public key hash) and rejects connections that don't match — even if the TLS chain is otherwise valid.

**Where you see pinning**:
- Mobile apps (HPKP is deprecated, but many apps implement custom pinning via OkHttp / AFNetworking).
- Internal services using strict mTLS where the expected peer cert is hardcoded.
- curl / wget with `--pinnedpubkey` in old scripts.
- Some IoT devices with hardcoded cert fingerprints.

**The problem**: you rotate the cert (because it's expiring). The cert is brand new and valid. But the pinned client rejects it because the fingerprint changed.

**Detection**: TLS handshake failures immediately after cert rotation, from specific clients only. `"SSL certificate problem: certificate subject name mismatch"` or `"handshake failure"` in client logs.

**Solutions**:
1. **Pin the CA, not the leaf cert.** Clients should pin the CA's public key, not the leaf cert's. Leaf certs rotate; the CA doesn't (on the scale of years). If clients pin the CA, any cert signed by that CA works. This is the correct engineering approach.
2. **Dual-cert rotation window.** Serve both the old and new cert simultaneously (SNI-based or via interim dual-response). Give clients time to update. Remove the old cert after all clients have transitioned. Requires infra support (some load balancers can serve multiple certs).
3. **Force client update.** For mobile apps: publish an update that includes the new pin before rotating the cert. Requires lead time.
4. **Emergency: roll back the cert.** If pinned clients are business-critical and can't be updated, restore the old cert temporarily. Buy time for proper coordination.

---

## mTLS client cert rotation

When services do mutual TLS, both sides present a cert. Rotating the server cert is the easy part. Rotating the **client cert** is where teams get burned.

### The problem

Service A (client) presents client cert to Service B (server). Service B verifies A's cert against the trusted CA bundle in its config.

You rotate Service A's client cert. But Service B's trusted CA bundle still expects the old CA (or the old cert, if B is pinning). Result: Service A's new cert is rejected by Service B.

### The rotation choreography

For cert-manager with short-lived Vault certs (the good case):
- Both services trust the same intermediate CA.
- When A's cert rotates, B validates it against the same intermediate CA.
- No choreography needed — as long as the CA doesn't change.

For CA root rotation (the hard case):
1. **Add the new CA to both trust bundles first.** Before issuing any certs from the new CA, update every service's trusted CA bundle to include *both* old and new CA. This is the dual-trust window.
2. **Issue new certs from the new CA.** Services with the dual-trust bundle will accept certs from either CA.
3. **Wait for old certs to expire or force-rotate them.**
4. **Remove the old CA from trust bundles.** Only after all old-CA certs have been replaced.

This is why CA root rotation is operationally complex. Steps 1 and 4 require touching every service's trust config. In cert-manager, the CA bundle is typically in a ConfigMap or the `ca.crt` field in Secrets. Automating steps 1 and 4 across hundreds of services is non-trivial.

**Tools that help**: cert-manager's `trust-manager` (formerly trust-manager) is a Kubernetes controller that distributes CA bundles across namespaces, managed as a `Bundle` CRD. It synchronizes CA trust updates cluster-wide.

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: my-org-ca-bundle
spec:
  sources:
    - useDefaultCAs: true
    - secret:
        name: vault-intermediate-ca
        key: tls.crt
  target:
    configMap:
      key: "ca-bundle.crt"
    additionalFormats:
      jks:
        key: "ca-bundle.jks"
```

trust-manager distributes this `ConfigMap` to every namespace, keeping it up to date when the source Secret changes.

---

## The interview answer in 60 seconds

> "Two layers of detection and one layer of prevention.
>
> Prevention: cert-manager with a `renewBefore` of 30 days on 90-day certs. Renewal starts a month before expiry, giving 30 days to catch and fix failures. For Vault short-lived certs, the window is proportionally shorter but auto-rotation is built in.
>
> Detection layer 1: Prometheus blackbox exporter probing every HTTPS endpoint. Alerts at 30 days (ticket), 7 days (urgent), 24 hours (P1 page). This catches everything — CDN certs, database TLS, anything not managed by cert-manager.
>
> Detection layer 2: cert-manager Prometheus metrics — alert on any `Certificate` that's NotReady for more than 30 minutes, and on certs where renewal has clearly failed (expiry < 7 days AND NotReady).
>
> The senior gotcha I'd always name: client-side pinning. If any client pins the leaf cert's fingerprint, rotating the cert breaks them silently. The fix is to pin the CA instead of the leaf cert, and to test cert rotation in staging with real clients before production. For mTLS, CA root rotation requires a dual-trust window — add the new CA to all trust bundles before issuing from it; remove the old CA only after all old-CA certs have rotated out."

---

## Self-test drills

### 1. How would you detect and prevent cert expiry outages across hundreds of services?

**Reference answer:**
- **Prevention**: cert-manager with appropriate `renewBefore` (30 days for 90-day certs). Vault PKI for internal short-lived certs with auto-rotation via cert-manager or Vault Agent.
- **Detection layer 1**: blackbox exporter probing all HTTPS endpoints. Alert ladder: 30/7/1 day thresholds for warning/urgent/page.
- **Detection layer 2**: cert-manager Prometheus metrics — `certmanager_certificate_ready_status` alert on NotReady > 30 minutes; `certmanager_certificate_expiration_timestamp_seconds` for expiry cross-check.
- **Coverage gap**: certs not managed by cert-manager (CDN, external LB, databases) — only blackbox catches these.
- **Post-rotation check**: `openssl s_client` or blackbox probe after any cert rotation to confirm the new cert is being served.

### 2. A cert expired in production. Walk me through the emergency response.

**Reference answer:**
1. Immediately: `echo | openssl s_client -connect <host>:443 2>/dev/null | openssl x509 -noout -dates` — confirm which cert expired and what domain.
2. Check cert-manager: `cmctl status certificate <name>` — is it trying to renew? Why is it stuck?
3. If cert-manager is stuck: check `Challenge` events (HTTP-01 endpoint down? DNS credentials wrong?). Fix the underlying issue.
4. Force renewal: `cmctl renew <name>` after the fix.
5. If cert-manager can't renew in time: manually generate and deploy a cert (from Let's Encrypt staging or Vault) to restore service while the automation fix is in progress.
6. Post-mortem: why didn't the 30-day alert fire? Add it if missing. Was the cert outside cert-manager management? Add it to blackbox exporter targets.

### 3. What is client-side certificate pinning, and how do you handle cert rotation with pinned clients?

**Reference answer:**
- Pinning: client stores the cert fingerprint or public key hash and rejects any cert that doesn't match, even if TLS chain is otherwise valid.
- Symptom after rotation: TLS handshake failures from specific clients immediately after cert rotation.
- Fix option 1 (correct): switch clients to CA pinning instead of leaf cert pinning. Leaf certs rotate; CAs don't.
- Fix option 2 (operational): dual-cert rotation window — serve old + new cert simultaneously. Let clients transition. Remove old cert after all clients updated.
- Fix option 3 (emergency): restore old cert, coordinate client update, then rotate properly.
- Advice: test cert rotation in staging with representative clients before doing it in production.

### 4. How would you handle mTLS client cert rotation across 50 services?

**Reference answer:**
- If using cert-manager + Vault PKI: certs are issued with the same intermediate CA. Rotation is automatic (cert-manager renews before expiry); no choreography needed as long as the CA doesn't change.
- If rotating the CA root: dual-trust window required. (1) Add new CA to all services' trusted CA bundles using trust-manager `Bundle` CRD. (2) Begin issuing certs from new CA. (3) Wait for all old-CA certs to expire or force-rotate them. (4) Remove old CA from bundles.
- Use trust-manager (cert-manager project) to distribute CA bundles cluster-wide — avoids manually updating 50 ConfigMaps.
- Monitor for TLS handshake failures during rotation: any service with an old-CA cert connecting to a service that's dropped the old CA trust will fail.

---

## Further reading / watching

- **Prometheus blackbox exporter** — [github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter). The `ssl` module docs show the TLS cert expiry probe configuration.
- **cert-manager monitoring** — [cert-manager.io/docs/observability/metrics](https://cert-manager.io/docs/observability/metrics/). Every Prometheus metric cert-manager exposes, with descriptions.
- **trust-manager** — [cert-manager.io/docs/trust/trust-manager](https://cert-manager.io/docs/trust/trust-manager/). The `Bundle` CRD for cluster-wide CA bundle distribution.
- **Stakater Reloader** — [github.com/stakater/Reloader](https://github.com/stakater/Reloader). Triggers Deployment rollouts when watched Secrets change — the standard pattern for picking up renewed certs.
- **"The cert expiry timeline" — Let's Encrypt blog** — [letsencrypt.org/docs/expiration-emails](https://letsencrypt.org/docs/expiration-emails/). Let's Encrypt sends expiry notification emails — useful fallback, but don't rely on it as your primary alert.
- **HPKP deprecation** — search "HPKP deprecated Chrome" for why HTTP Public Key Pinning was removed from browsers. The reasons are directly applicable to why you should pin CAs, not leaf certs.

---

## The 4 dimensions (senior framing)

- **Tech**: cert-manager renewal at `renewBefore` (default 1/3 of duration; 30 days for 90-day certs); blackbox exporter for end-to-end TLS cert expiry detection; cert-manager Prometheus metrics for cert-manager-managed certs; trust-manager for CA bundle distribution; Stakater Reloader for pod restart on cert rotation; dual-trust window for CA root rotation; CA pinning vs leaf pinning.
- **People**: cert expiry outages are almost always operational failures, not technical ones. "Someone should have noticed" is the answer. Building the 30/7/1 alert ladder makes expiry a team responsibility with enough lead time to act. Designate an owner for non-cert-manager certs (CDN, external LBs). Document the manual rotation procedure in the runbook so any on-call can execute it — not just the cert specialist.
- **CI/CD**: cert rotation should be tested in staging before production. CI environments should use short-lived certs with automated rotation — this exercises the rotation machinery regularly and gives you early warning when something breaks. If your test environment has certs that auto-rotate every hour, you'll know immediately when cert-manager is broken. Include cert validity checks in integration test suites: `openssl s_client` verification against staging endpoints as part of deployment gates.
- **Operations**: the blackbox exporter target list needs to be complete and maintained. If a new service gets added without being added to the probe list, it's invisible. Use service discovery where possible (Kubernetes service discovery in Prometheus to automatically probe all services with a specific annotation). Post-rotation SOP: always verify the new cert is being served before closing the incident. Monitor cert-manager health (not just cert health): if the cert-manager deployment is unhealthy, all certs will eventually fail to renew — alert on cert-manager pod readiness too.
