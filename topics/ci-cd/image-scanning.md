# Image scanning (Trivy, Grype) — the deep-dive

> **Goal**: by the end you can answer the killer question — **"How does Trivy actually find vulnerabilities in your image, and what's the trade-off vs Wiz/cloud scanners?"** — naming the databases queried, the layer-walking mechanic, the false-positive problem and mitigations, and the OSS-vs-cloud trade-off.

> Start with [image-scanning-simple.md](./image-scanning-simple.md) if you haven't. The health-inspector analogy is the spine of this.

---

## The senior framing — scanning is signal quality, not just coverage

Mid-level: "we run Trivy in CI and fail on CRITICAL."
Senior: "we run Trivy for fast local feedback, filter noise aggressively, track fixed-only CVEs in the gate, and use Wiz as the authoritative policy gate — because different tools give you different signal quality at different points in the pipeline."

The interview moment is explaining *why* you'd use both, not which one is "better."

---

## How Trivy actually works — the layer walk

A container image is an OCI manifest: a JSON index pointing to a list of **layer blobs** (tar archives of filesystem diffs). When you run `trivy image`, here's what happens:

1. **Pull the manifest** — Trivy fetches the image manifest (or reads the local daemon's layers).
2. **Unpack each layer** — Each tar blob is extracted into a virtual filesystem.
3. **Discover package databases** — Trivy looks for known package-manager artifacts in the virtual filesystem:
   - Debian/Ubuntu: `/var/lib/dpkg/status`
   - Alpine: `/lib/apk/db/installed`
   - RHEL/CentOS: `/var/lib/rpm/Packages`
   - Node: `package-lock.json`, `yarn.lock`, `node_modules/*/package.json`
   - Python: `site-packages/*/METADATA`, `requirements.txt`, `Pipfile.lock`
   - Go: `go.sum`, embedded build info (`go version -m <binary>`)
   - Java: `pom.xml`, jar manifests (`META-INF/MANIFEST.MF`, `pom.properties`)
   - Rust: `Cargo.lock`
4. **Look up each package** — For each package-name + version pair, Trivy queries its local vulnerability DB (a bundled copy updated by `trivy db update`).
5. **Emit findings** — Each match is a CVE ID + severity + affected version range + fixed version (if one exists).

No code executes. This is pure **static inventory-against-database** matching. That's also the source of false positives: the database doesn't know whether the vulnerable code path is reachable in your specific usage.

### The databases Trivy queries

| Database | What it covers | Notes |
|---|---|---|
| **NVD** (National Vulnerability Database) | All CVEs, all ecosystems | US government, authoritative, can lag on ecosystem packages |
| **GHSA** (GitHub Security Advisories) | npm, PyPI, Go, Maven, Cargo, Composer | Better for ecosystem packages; faster than NVD for these |
| **OS advisories** | Debian Security, Alpine SecDB, RHEL/CentOS OVAL, Ubuntu USN | OS-vendor assessments; often know about backported fixes that NVD doesn't |
| **OSV** (Open Source Vulnerabilities) | Merged view of GHSA + others | Google-run; cross-ecosystem |

Trivy also has a **secrets scanner** (finds API keys, tokens in image layers) and **IaC scanner** (Terraform, Helm, K8s manifests) — but the CVE path above is the core for image scanning.

### Grype — the alternative

Grype (by Anchore) works the same way but uses Anchore's own database (backed by NVD + GHSA + OS advisories). Key differences:

| | Trivy | Grype |
|---|---|---|
| Scope | Images + IaC + secrets + repos | Images + SBOMs |
| SBOM input | Can consume | Can produce + consume |
| UX | `trivy image <name>` | `grype <name>` |
| Plugin ecosystem | Larger | Smaller |
| Common in prod | More common | Common in Anchore-integrated workflows |

Pick Trivy unless you're in an Anchore shop. The knowledge transfers — same concept, different CLI.

---

## The fixed/unfixed CVE distinction — the most important gate knob

Every Trivy finding has a `Fixed Version` column. That field determines what you can actually act on.

- **Fixed** — a patched version exists in your package manager. You can upgrade. CI should block on these at the relevant severity.
- **Unfixed** — no upstream fix exists yet. You cannot upgrade your way out. CI blocking on unfixed CVEs creates noise with no action path.

The production pattern:

```yaml
# .github/workflows/scan.yaml
- name: Scan image
  run: |
    trivy image \
      --exit-code 1 \
      --ignore-unfixed \
      --severity HIGH,CRITICAL \
      --format sarif \
      --output trivy-results.sarif \
      $IMAGE_TAG

- name: Upload SARIF to Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif
```

`--ignore-unfixed` is not "ignore security." It's "don't block on things you can't fix today." LOW and MEDIUM go to a ticket, not a gate. `--format sarif` feeds GitHub's Security tab with PR annotations.

---

## The false-positive problem

Three common false positive types:

### 1. Backported fixes (OS vendor patched without version bump)

OS vendors (Debian, Ubuntu, RHEL) often backport security fixes to older package versions without bumping the version number. NVD says the version is vulnerable; the OS advisory says it's patched. Trivy uses OS advisories to resolve this — but only if your image is OS-based. Distroless images can sometimes still show stale NVD findings.

Mitigation: check the OS advisory first. If Debian's tracker says "fixed" for your version, treat it as a false positive with a note.

### 2. Unreachable code path

A CVE is in a library, but the vulnerable function is never called by your app (or only exercised by test code that's not in the image). Static scanners can't know this.

Mitigation: this requires human triage. Accept-and-document in `.trivyignore` with a comment explaining the reasoning and a review date.

### 3. Stale database

Trivy's DB is a snapshot. If a CVE was published this morning and you haven't run `trivy db update`, it won't appear. Conversely, if a CVE was marked "rejected" in NVD but your local DB hasn't updated, you'll get a stale finding.

Mitigation: run `trivy db update` before each scan in CI (`--download-db-only` in a separate step for caching).

### The `.trivyignore` file

```bash
# .trivyignore
# CVE-2024-XXXX: unreachable code path in lib/util/crypto.go - reviewed 2026-06-05, next review 2026-09-05
CVE-2024-XXXX

# CVE-2024-YYYY: no fix available, tracked in https://jira.example.com/PLAT-4321
CVE-2024-YYYY
```

Keep this file in the repo, reviewed quarterly. Each entry must have a comment explaining why it's accepted and a review date.

---

## CI integration patterns

### Pattern 1 — fail fast (simplest)

```yaml
- run: trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed $IMAGE
```

Blocks the PR if CRITICAL or HIGH unfixed CVEs exist. Fast. No noise. Good starting point.

### Pattern 2 — non-blocking scan with SARIF reporting

```yaml
- run: |
    trivy image \
      --exit-code 0 \
      --severity CRITICAL,HIGH,MEDIUM \
      --format sarif \
      --output trivy.sarif \
      $IMAGE
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy.sarif
```

Reports everything to the Security tab without blocking. Good for introducing scanning to an existing codebase without breaking builds.

### Pattern 3 — separate gate per severity

```yaml
# Block on CRITICAL
- run: trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE

# Warn on HIGH (exit 0 so it doesn't block, but shows in SARIF)
- run: trivy image --exit-code 0 --severity HIGH --ignore-unfixed --format sarif --output high.sarif $IMAGE
```

Different severity levels get different actions. CRITICAL = hard block. HIGH = visible warning + ticket. LOW/MEDIUM = weekly digest.

### Caching the DB in CI

```yaml
- name: Cache Trivy DB
  uses: actions/cache@v4
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ steps.date.outputs.date }}
    restore-keys: trivy-db-

- name: Update Trivy DB
  run: trivy db update
```

Without caching, Trivy re-downloads the DB (~20 MB) on every run. Cache by date so you get a fresh DB once per day.

---

## OSS vs cloud scanner trade-off — the Wiz framing

You have production Wiz experience, so this is a comparison from lived context:

| Dimension | Trivy/Grype (OSS, local) | Wiz (cloud, centralized) |
|---|---|---|
| **Where it runs** | In CI, on dev laptop | Wiz's cloud, scanning your registry/runtime |
| **Feedback loop** | Immediate, per PR/commit | After push, minutes delay |
| **False positive rate** | Higher — no cloud context | Lower — knows your cloud config, runtime context |
| **Policy enforcement** | `--exit-code 1` gate in CI | Image gate before promotion, runtime posture |
| **Coverage** | Container filesystem + IaC | Container + cloud config + runtime posture + network exposure |
| **Secrets scanning** | Yes (Trivy) | Yes + cloud credentials in use |
| **Cost** | Free | Expensive |
| **Who owns it** | Platform team, per-repo config | Security team, org-wide policy |
| **SBOM integration** | Yes (Trivy can output SBOM) | Yes |

The senior answer on which to use: **both, with clear division of responsibility.**

- Trivy: developer feedback loop. Fast, cheap, catches issues before they reach the registry. Configured per-repo in CI.
- Wiz: authoritative gate. Enforces org-wide policy (minimum severity threshold, required base images, runtime posture). Security team sets policy; devs can't bypass it.

The trap to avoid: running both with different policies so a "Wiz clean" image still fails Trivy and vice versa. Align the severity thresholds between them.

---

## The interview answer in 60 seconds

> "Trivy unpacks every layer of the image — the OCI tar blobs — and inventories every package it finds: dpkg, rpm, npm, pip, go.sum, jar manifests. For each package-plus-version, it looks up the combination in local copies of NVD, GHSA, and OS vendor advisories. No code executes — it's pure static analysis.
>
> The main friction is false positives. The two big mitigations: `--ignore-unfixed` drops CVEs with no available fix (nothing you can do today), and severity filtering keeps the CI gate focused on HIGH and CRITICAL. Accepted-risk CVEs go into a `.trivyignore` file with a review date.
>
> The OSS-vs-cloud trade-off: Trivy gives developers fast, local feedback on every PR — catch it before it reaches the registry. Wiz (which I've used in production) is the authoritative policy gate — centralized, org-wide, with runtime and cloud-posture context that reduces false positives. You want both: Trivy for speed, Wiz for authority. The mistake is letting them disagree on policy."

---

## Self-test drills

### 1. How does Trivy actually find vulnerabilities in your image?

**Reference answer:**
- Unpacks each OCI layer (tar blob) into a virtual filesystem.
- Discovers package databases: dpkg, rpm, apk, npm/yarn lock, pip, go.sum, jar manifests.
- For each package-version pair, looks up in NVD, GHSA, OS advisories, OSV.
- Emits CVE ID + severity + fixed version if available. No code execution — pure static inventory matching.

### 2. What's the difference between `--ignore-unfixed` and ignoring a CVE entirely?

**Reference answer:**
- `--ignore-unfixed` drops all CVEs that have no upstream fix available. Policy decision: "we can't act on this today, so don't block CI." Still visible in the SARIF report.
- `.trivyignore` drops specific CVE IDs regardless of fix status. Used for accepted-risk findings with documented justification.
- Never use `.trivyignore` for something that has a fix — that's a drift risk. `--ignore-unfixed` for the unfixable; explicit `.trivyignore` entries (reviewed quarterly) for the genuinely accepted.

### 3. Why might Trivy flag a CVE that Wiz doesn't?

**Reference answer:**
- Wiz has runtime context: it knows if the vulnerable package is actually loaded at runtime, whether the network path to it is exposed, and what the cloud configuration looks like.
- Trivy only has the static filesystem. It can't know whether the vulnerable function is called.
- OS vendor backport awareness: Wiz may have more up-to-date OS advisory integration than Trivy's local DB.
- The practical answer: when Wiz accepts something Trivy flags, verify the OS advisory, then add a `.trivyignore` entry with a note referencing the advisory.

### 4. How would you reduce scan noise in a CI pipeline that's getting too many false positives?

**Reference answer:**
1. Add `--ignore-unfixed` — eliminates unfixable CVEs from the gate.
2. Raise the severity threshold to `CRITICAL,HIGH` — LOW/MEDIUM go to a weekly digest, not a block.
3. Audit and trim `.trivyignore` — every accepted CVE needs a justification comment + review date.
4. Use `--format sarif` + GitHub Security tab for LOW/MEDIUM so they're visible but not blocking.
5. Run `trivy db update` daily in CI so the DB reflects current advisory states.
6. For OS-vendor backport false positives, cross-reference the OS security tracker (e.g., [security.debian.org](https://security.debian.org)) and document if it's a confirmed false positive.

---

## Further reading

- [Trivy documentation](https://aquasecurity.github.io/trivy/) — especially the "Container Image" and "CI Integration" sections
- [Grype (Anchore)](https://github.com/anchore/grype) — the OSS alternative; worth knowing
- [NVD](https://nvd.nist.gov/) and [GHSA](https://github.com/advisories) — the primary databases Trivy queries
- [OSV.dev](https://osv.dev/) — Google's cross-ecosystem vulnerability database
- [Trivy SARIF output → GitHub Security tab](https://aquasecurity.github.io/trivy/latest/docs/integrations/github-actions/) — how to wire it into PR annotations

---

## The 4 dimensions (senior framing)

- **Tech**: Trivy layer-walk → package inventory → DB lookup. `--ignore-unfixed` + severity threshold + `.trivyignore` with review dates. SARIF output to GitHub Security tab. Trivy for local dev loop; Wiz for org gate.
- **People**: false positive noise kills scanner adoption. Invest time up-front aligning Trivy thresholds with Wiz policy so devs see consistent signal. Educate teams on `--ignore-unfixed` so they don't disable scanning entirely when it gets noisy.
- **CI/CD**: scan in the build stage before push, not after. Use `--exit-code 1` only for HIGH/CRITICAL unfixed. Run `db update` daily; cache the DB between runs. Add SARIF uploads for Security tab visibility without blocking on lower severities.
- **Operations**: schedule weekly full-scan sweeps of the image registry (not just on build) — a new CVE can affect an old image that hasn't been rebuilt. Track `.trivyignore` entries in a quarterly review. Monitor the lag between NVD publication and Trivy DB update for SLA-sensitive environments.
