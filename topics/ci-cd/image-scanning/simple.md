# Image scanning (Trivy, Grype) — the simple version (the ingredients check)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **A container image is a recipe. Scanning it means checking every ingredient against a known list of bad ones — before you serve it to customers.**

That's it. Everything else (NVD, GHSA, layer-walking, severity thresholds) is precision on top.

---

## The restaurant health inspector

Imagine a restaurant health inspector. Before the restaurant opens, they walk through the kitchen and check:

- Every ingredient against a list of recalled products (CVE databases)
- Every brand of tool against known safety violations
- Every supplier's certification

They don't taste every dish. They check the **list of ingredients and tools** against **known-bad databases**.

That's image scanning. Your image is the kitchen. The scanner (Trivy/Grype) walks every shelf (every layer) and checks every package against vulnerability databases (NVD, GHSA, OS advisories). It produces a report: "ingredient X is on the recall list, severity HIGH, fixed in version Y."

---

## How scanners actually walk an image

A Docker image is a stack of layers. Each layer is a tar archive of filesystem changes.

The scanner unpacks every layer and reads the package databases inside — `dpkg`, `rpm`, `apk`, `npm`, `pip`, `go.sum`, etc. For each package it finds, it looks up the package name + version in its local copy of CVE databases and says: "is this combination known-vulnerable?"

That's it. No code execution. Pure **static analysis of what's installed**.

---

## Trivy vs Wiz — the practical difference

You know Wiz from production. Here's how they relate:

| | Trivy (OSS, local) | Wiz (cloud, centralized) |
|---|---|---|
| Where it runs | In your CI pipeline, on your laptop | Wiz's cloud, scans your registry |
| Feedback loop | Immediate, on every PR | After push, centralized visibility |
| Policy enforcement | `--exit-code 1 --severity CRITICAL` | Wiz policy gates image promotion |
| False positives | More, because no OS context filtering | Fewer — Wiz has richer context |
| Coverage | Containers + IaC (Terraform, Helm) | Containers + cloud posture + CSPM |

Trivy = the fast local check that gives devs instant feedback.
Wiz = the authoritative gate that enforces org-wide policy.

They're complementary. Trivy catches things before push. Wiz is the final say.

---

## The false-positive problem

The biggest scanner frustration: a CVE is flagged but either:
- It's already fixed in the installed version (the database is stale), or
- The vulnerable code path isn't actually reachable in your app.

The practical response for a senior engineer:

1. **Filter by `--ignore-unfixed`** — if the OS vendor hasn't released a fix yet, there's nothing you can do today. Don't block CI on it.
2. **Filter by severity** — `--severity HIGH,CRITICAL` on the CI gate; LOW/MEDIUM go to a ticket, not a block.
3. **Maintain a `.trivyignore`** file for accepted-risk CVEs (with a review date comment).

Wiz has similar exception mechanisms. Same principle.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What does a scanner actually check? | Package name + version against CVE databases. Static analysis, no code execution. |
| What's NVD? | National Vulnerability Database — the US government's CVE list. The main source. |
| What's GHSA? | GitHub Security Advisories. Better for npm/PyPI/Go packages than NVD. |
| What's a false positive? | CVE flagged but already fixed, or not reachable in your code. Filter with `--ignore-unfixed`. |
| Trivy vs Grype? | Both OSS local scanners. Trivy covers more (IaC, Helm, secrets). Grype has a simpler UX. Pick one; Trivy is more common in production. |
| Trivy vs Wiz? | Trivy = fast local loop; Wiz = authoritative cloud gate. Use both. |

---

## Self-test (the killer question)

Out loud:

> **"How does Trivy actually find vulnerabilities in your image, and what's the trade-off vs Wiz/cloud scanners?"**

**Reference answer (intuitive version):**

"Trivy unpacks every layer of the image — the tar archives that make up the filesystem — and reads the package databases it finds: dpkg, rpm, npm, pip, go.sum, etc. For each package-plus-version it finds, it looks it up in local copies of NVD, GHSA, and OS-level advisories. No code runs. Pure static analysis of what's installed.

The trade-off vs Wiz: Trivy gives you fast, local, per-PR feedback — devs catch problems before they even push. Wiz gives you centralized policy with richer context — it knows your cloud environment, can filter out CVEs that genuinely aren't exploitable in your config, and enforces org-wide gates. In production you want both: Trivy in CI for fast feedback, Wiz as the authoritative gate before an image gets promoted to prod."

---

## Further reading

- [Trivy docs](https://aquasecurity.github.io/trivy/) — start with the container scanning quickstart
- [Grype](https://github.com/anchore/grype) — Anchore's scanner; worth knowing the name
- [NVD](https://nvd.nist.gov/) — the US CVE database Trivy hits
- [GHSA](https://github.com/advisories) — GitHub's advisory database, better for ecosystem packages

---

## Next: the deep-dive

When the health-inspector analogy feels obvious, jump to [`image-scanning.md`](./deep-dive.md). The deep-dive covers:

- Exactly how Trivy walks layers and which databases it queries
- The fixed/unfixed CVE distinction and why it matters for CI gates
- The `--ignore-unfixed` + severity filtering pattern for reducing noise
- `.trivyignore` and exception management
- CI integration (fail-on-severity, SARIF output, PR annotations)
- The OSS-vs-cloud-scanner trade-off in depth (the Wiz framing)
- 3 self-test drills + 4-dimensions framing
