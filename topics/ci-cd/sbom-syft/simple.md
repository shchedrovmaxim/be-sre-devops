# SBOM + Syft — the simple version (the ingredient label)

> Read this first. Once the analogy clicks, the [deep-dive](./deep-dive.md) is easy.

This doc explains **one idea**:

> **An SBOM is the ingredient label on your container. A vulnerability scan tells you if the current ingredients are safe. The SBOM tells you *what* the ingredients actually are — forever, even if the package changes later.**

---

## The food label vs the health inspection

When you buy a food product, there are two things:

1. **The health inspection** — a food-safety officer checked the kitchen and said "no active contamination today." That's your vulnerability scan (Trivy/Wiz). It's point-in-time.
2. **The ingredient label** — a permanent list of every ingredient, forever attached to the product. Even if the health inspector comes back next year, they can check the label. That's your SBOM.

Why does the label matter even if the inspection passed? Because:
- A new vulnerability is discovered 6 months later in `log4j`. You need to know: "which of our 500 images contains log4j?" You can answer instantly if you have SBOMs for all of them. Without SBOMs, you have to re-scan all 500 images — if you still have them.
- Regulators (US Executive Order 14028) want proof of what's in your software. The SBOM is the proof.
- A supplier ships you a container. You need to know what's in it without trusting their word. The SBOM (signed) is the receipt.

---

## What Syft does

[Syft](https://github.com/anchore/syft) is a CLI tool that:

1. Unpacks your image (same layer-walking as Trivy)
2. Finds every package and its version (dpkg, npm, pip, go modules, jars...)
3. Outputs a structured **SBOM document** in SPDX or CycloneDX format

```bash
# Generate an SBOM for an image
syft my-app:v1.2.3 -o spdx-json > sbom.spdx.json

# Or CycloneDX format
syft my-app:v1.2.3 -o cyclonedx-json > sbom.cdx.json
```

The output is a JSON/XML document listing every package, its version, its license, and where it was found in the image.

Then you can:
- Attach the SBOM to the image (via Cosign — see `cosign-sigstore.md`)
- Feed it into Grype to scan it: `grype sbom:sbom.spdx.json`
- Store it in an artifact store and query it later

---

## The two SBOM formats you need to know

| Format | Maintained by | Common in |
|---|---|---|
| **SPDX** | Linux Foundation | US government compliance (EO 14028), NTIA |
| **CycloneDX** | OWASP | Security tooling (Grype, Dependency-Track) |

They're both fine. Pick one per org. CycloneDX is more common in security-tooling pipelines; SPDX is what the US government asks for. Many tools accept both.

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| What is an SBOM? | A structured list of every package in your software, with versions and licenses. |
| Why do we need it if we already scan? | Scanning is point-in-time. SBOMs survive image deletion. New CVEs can be matched retroactively against old SBOMs. |
| What does Syft do? | Walks your image layers, discovers packages, writes out the SBOM document. |
| What's SPDX vs CycloneDX? | Two standard formats. SPDX = Linux Foundation / US gov. CycloneDX = OWASP / security tooling. Either works. |
| How does it connect to signing? | Cosign can attach the SBOM as an attestation on the image in the registry. Image + SBOM = verified receipt. |
| Is it a compliance thing? | Yes — US EO 14028 requires SBOMs for software sold to the US government. SSDF/NIST also reference them. |

---

## Self-test (the killer question)

Out loud:

> **"Why do we need SBOMs in addition to vulnerability scans?"**

**Reference answer (intuitive version):**

"Vulnerability scans are point-in-time — they tell you if the current packages are known-vulnerable today. But a new CVE can drop six months after you built the image. If you have an SBOM attached to every image, you can instantly query 'which images contain this package at this version?' without re-building or re-scanning them. SBOMs also survive image deletion — the SBOM stored in your artifact registry is your permanent record. And there's a compliance angle: US Executive Order 14028 requires software vendors to provide SBOMs. If you're selling to government, or building platform software that others depend on, the SBOM is the proof of what's inside. Think of it as the difference between 'we passed inspection' (scan) and 'here's the permanent ingredient label' (SBOM)."

---

## Further reading

- [Syft GitHub](https://github.com/anchore/syft) — start with the quickstart
- [CycloneDX spec](https://cyclonedx.org/) — the format most security tooling uses
- [SPDX spec](https://spdx.dev/) — the Linux Foundation format; US government preferred
- [NTIA SBOM guidance](https://www.ntia.gov/sbom) — the regulatory context (EO 14028)
- [Dependency-Track](https://dependencytrack.org/) — a tool for ingesting and querying SBOMs at scale

---

## Next: the deep-dive

When the ingredient-label analogy feels obvious, jump to [`sbom-syft.md`](./deep-dive.md). The deep-dive covers:

- Syft's component discovery per package manager
- SPDX vs CycloneDX in detail
- Supply-chain attestations and linking the SBOM to Cosign signing
- CI integration patterns
- The compliance angle (SSDF, EO 14028) in depth
- The "SBOM as documentation" senior framing
- 3 self-test drills + 4-dimensions framing
