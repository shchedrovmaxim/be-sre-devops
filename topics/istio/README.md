# Istio

Service mesh. Mostly complete from earlier focused prep.

## Status: ✅ Largely done — reference for review

## Files

| File | What it covers |
|---|---|
| [full-reference.md](full-reference.md) | Consolidated cheat sheet: architecture, CRDs, mTLS, AuthZ, traffic mgmt, observability, AWS integration, troubleshooting, 26 Q&A drills |

## Section map (within `full-reference.md`)

| Section | Topic |
|---|---|
| Part 1, §1-24 | Big-picture: architecture, CRDs, app-to-infra mapping, mTLS, 503 zoo, traffic mgmt, LB, locality, rate limiting, egress, sidecar lifecycle, observability, multi-cluster, ambient, upgrades, ext_authz, EnvoyFilter, Gateway API, kube-proxy+CoreDNS coexistence, AWS NLB/ALB, istioctl |
| Part 2 | 6 high-likelihood deep dives: JWT/claim authz, AuthorizationPolicy semantics, TLS origination, control-plane outage, VS L7 features, Telemetry CR |
| Part 3 | General troubleshooting playbook (8-step) |
| Part 4 | 26 Q&A model answers |
| Part 5 | Mnemonics, war stories, what NOT to say |

## When to review

- **Before any interview that mentions service mesh**: re-read Parts 4 (Q&A) and 5 (mnemonics + war stories).
- **For an Istio-heavy role**: re-read Part 2 (the deep dives) and practice the 26 Q&A out loud.
- **For a troubleshooting question**: rehearse the Part 3 playbook.

## What's NOT covered (intentionally)

These came up but were deprioritized:
- Wasm extensions / WasmPlugin CRD (niche)
- Helm chart structure / IstioOperator deprecation (install history)
- HTTP/3 / QUIC (emerging)
- Mixer / Telemetry v2 migration (only matters for upgrades from Istio <1.5)
- Envoy bootstrap config / xDS protocol internals (too low-level)

If a future interview probes any of these, time to add them.
