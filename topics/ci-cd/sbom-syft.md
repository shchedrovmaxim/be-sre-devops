# SBOM + Syft — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Why do we need SBOMs in addition to vulnerability scans?"** — explaining point-in-time vs persistent inventory, retroactive CVE matching, the compliance angle, and how SBOMs connect to Cosign attestations.

> Start with [sbom-syft-simple.md](./sbom-syft-simple.md) if you haven't. The ingredient-label analogy frames everything here.

---

## The senior framing — SBOM as the persistent source of truth

Mid-level: "we generate an SBOM in CI and store it somewhere."
Senior: "the SBOM is the authoritative record of what shipped, permanently attached to the image, queryable retroactively when a new CVE drops — independent of whether the image still exists in the registry."

The key insight: **a vulnerability scan is a question answered at a point in time. An SBOM is the answer key that lets you answer the question at any future point.**

---

## What Syft actually discovers — per package manager

Syft walks the image filesystem and reads package metadata from every package manager it recognizes:

| Ecosystem | Where Syft looks |
|---|---|
| Debian/Ubuntu | `/var/lib/dpkg/status` |
| Alpine | `/lib/apk/db/installed` |
| RHEL/CentOS | `/var/lib/rpm/Packages` |
| Node.js | `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `node_modules/*/package.json` |
| Python | `site-packages/*/METADATA`, `RECORD`, `PKG-INFO`, `requirements.txt`, `Pipfile.lock`, `poetry.lock` |
| Go | `go.sum`, binary build info (reads `go version -m <binary>` — packages embedded in the binary, not just the source tree) |
| Java | `pom.xml`, `*.jar` manifest (`META-INF/MANIFEST.MF`, `META-INF/maven/*/pom.properties`), nested jars inside fat jars |
| Rust | `Cargo.lock` |
| Ruby | `Gemfile.lock` |
| PHP | `composer.lock` |

The **Go binary build info** case is important: Syft can extract the module list from a compiled Go binary without the source. This means it works on distroless images that contain only the binary — the source and go.sum don't need to be in the image.

Similarly for Java: Syft inspects fat jars (Spring Boot's nested jars, shaded jars) recursively.

---

## SBOM formats — SPDX vs CycloneDX

### SPDX (Software Package Data Exchange)

- Maintained by the Linux Foundation, ratified as ISO/IEC 5962:2021.
- The format explicitly required by the US government (NTIA guidance for EO 14028).
- SPDX 2.3 / 3.0 adds richer relationship modelling (DESCRIBES, CONTAINS, DEPENDS_ON, GENERATED_FROM).
- Common output: `spdx-json`, `tag-value`, `rdf-xml`.

```json
{
  "spdxVersion": "SPDX-2.3",
  "name": "my-app:v1.2.3",
  "packages": [
    {
      "name": "express",
      "versionInfo": "4.18.2",
      "supplier": "Organization: npmjs.com",
      "downloadLocation": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "filesAnalyzed": false,
      "externalRefs": [
        { "referenceType": "cpe23Type", "referenceLocator": "cpe:2.3:a:expressjs:express:4.18.2:*:*:*:*:node.js:*:*" },
        { "referenceType": "purl", "referenceLocator": "pkg:npm/express@4.18.2" }
      ]
    }
  ]
}
```

### CycloneDX

- Maintained by OWASP.
- More popular in security tooling (Grype, Dependency-Track, OWASP ZAP).
- Supports VEX (Vulnerability Exploitability eXchange) — an extension that lets you embed "this CVE is not exploitable in this context" assertions directly in the SBOM.
- Common output: `cyclonedx-json`, `cyclonedx-xml`.

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "components": [
    {
      "type": "library",
      "name": "express",
      "version": "4.18.2",
      "purl": "pkg:npm/express@4.18.2",
      "licenses": [{ "license": { "id": "MIT" } }]
    }
  ]
}
```

**The purl (Package URL)** format (`pkg:npm/express@4.18.2`, `pkg:golang/github.com/gin-gonic/gin@v1.9.1`) is the cross-format identifier that vulnerability databases use to match packages. Both SPDX and CycloneDX include purls. This is the linchpin between an SBOM and a CVE database.

### Which to use?

- **US government / NTIA compliance** → SPDX 2.3+
- **Security tooling pipeline (Grype, Dependency-Track)** → CycloneDX
- **Both** → generate both from Syft, store both. Syft makes this trivial.

---

## CI integration — the canonical pipeline

```yaml
# .github/workflows/sbom.yaml
jobs:
  build-and-sign:
    steps:
      - name: Build image
        run: docker build -t $IMAGE_TAG .

      - name: Push image
        run: docker push $IMAGE_TAG

      - name: Generate SBOM (CycloneDX)
        run: |
          syft $IMAGE_TAG -o cyclonedx-json > sbom.cdx.json

      - name: Generate SBOM (SPDX)
        run: |
          syft $IMAGE_TAG -o spdx-json > sbom.spdx.json

      - name: Scan SBOM for vulnerabilities
        run: |
          grype sbom:sbom.cdx.json \
            --fail-on high \
            --only-fixed

      - name: Attest SBOM to image (via Cosign)
        run: |
          cosign attest \
            --predicate sbom.cdx.json \
            --type cyclonedx \
            $IMAGE_TAG
        # This attaches the SBOM as a signed attestation in the registry
        # See cosign-sigstore.md for the full keyless signing setup

      - name: Upload SBOM as artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: |
            sbom.cdx.json
            sbom.spdx.json
```

The `cosign attest` step is the senior move — the SBOM is permanently attached to the image in the registry, cryptographically signed, queryable without having the original files. See [cosign-sigstore.md](./cosign-sigstore.md) for the full signing setup.

### Scanning via SBOM (vs scanning the image directly)

```bash
# Option A: scan the image directly
trivy image my-app:v1.2.3

# Option B: generate SBOM first, scan the SBOM
syft my-app:v1.2.3 -o cyclonedx-json > sbom.cdx.json
grype sbom:sbom.cdx.json

# Option C: use Trivy with its own SBOM generation
trivy image --format cyclonedx --output sbom.cdx.json my-app:v1.2.3
trivy sbom sbom.cdx.json
```

Scanning via SBOM is important for the retroactive case: if a new CVE drops and the image was deleted, you still have the SBOM in your artifact store. You can query it offline: `grype sbom:stored-sbom.cdx.json`.

---

## Supply-chain attestations — the link to Cosign

An **attestation** is a signed statement about an artifact. When you run `cosign attest`, it:

1. Signs the SBOM JSON with your signing key (or keyless OIDC — see [cosign-sigstore.md](./cosign-sigstore.md))
2. Stores the signature + SBOM as an OCI artifact in the same registry namespace as the image
3. Creates a verifiable chain: image digest → SBOM → signature → signing identity

Later, in your Kubernetes admission flow (Kyverno `verifyImages`), you can verify:
- The image is signed (Cosign signature present)
- The image has an attached SBOM attestation (Cosign attestation present)
- The SBOM attestation is signed by a trusted identity

```yaml
# Kyverno policy: require SBOM attestation
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-sbom-attestation
spec:
  rules:
    - name: check-sbom-attestation
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          attestations:
            - type: https://cyclonedx.org/bom
              conditions:
                - all:
                    - key: "{{ bomFormat }}"
                      operator: Equals
                      value: "CycloneDX"
```

---

## The compliance angle — EO 14028 and SSDF

**US Executive Order 14028** (May 2021) on improving cybersecurity requires federal software vendors to:
- Provide an SBOM for software delivered to the US government
- Use one of the three NTIA-endorsed formats: SPDX, CycloneDX, or SWID

**SSDF (Secure Software Development Framework)** (NIST SP 800-218) references SBOM as part of PS.3 (Protect Software Releases). Not "required" but strongly recommended for attestation.

The practical upshot: if you're building software that is sold to, runs in, or depends on US government infrastructure, SBOMs are now a procurement requirement, not a nice-to-have.

Even outside government: for any enterprise customer with a mature supply-chain program, "do you provide SBOMs?" is increasingly a vendor due-diligence checkbox.

---

## The "SBOM as documentation" senior framing

Beyond compliance, there are operational wins:

**License auditing**: the SBOM lists every package's declared license. Legal can scan for GPL or AGPL dependencies that would require source disclosure, without having a developer read every transitive dependency manually.

**Dependency sprawl visibility**: the SBOM shows you how many packages are actually in production. Teams are often surprised — a "simple" Node app has 400+ transitive packages. That's a useful forcing function for dependency hygiene.

**Incident response**: when `log4shell` dropped, organizations with SBOMs answered "which of our services are affected?" in minutes. Without SBOMs, it took days of manual scanning.

**Supplier risk**: if you consume a third-party container (database, message broker, etc.), request the SBOM. You now know what's in their image without trusting their assertion.

---

## The interview answer in 60 seconds

> "A vulnerability scan is point-in-time: it tells you what's known-vulnerable today. An SBOM is the persistent inventory: the complete list of every package in the image, attached permanently. When a new CVE drops — say log4shell — you can retroactively query your SBOM store across all 200 images in your registry and know within seconds which ones contain the affected version. Without SBOMs, you have to re-scan every image, and if any were deleted, you've lost the evidence.
>
> We use Syft to generate SBOMs in CI — it walks every layer and discovers packages across all package managers, including Go binary build info without source. We output CycloneDX for security tooling and SPDX for compliance. We attach the SBOM as a Cosign attestation so it lives in the registry alongside the image, cryptographically tied to the image digest.
>
> The compliance angle: US EO 14028 requires SBOMs for government software. More broadly, it's the industry direction — customers will start asking for them as a procurement requirement."

---

## Self-test drills

### 1. Why do we need SBOMs in addition to vulnerability scans?

**Reference answer:**
- Scans are point-in-time. New CVEs drop after scans run. SBOMs let you retroactively match new CVEs against historical inventory without re-scanning.
- Images get deleted (registry cleanup policies). The SBOM stored in the artifact registry survives image deletion — it's the permanent record.
- Incident response: "which images contain package X at version Y?" is instant with a SBOM store; without one it's a multi-day manual scan.
- Compliance: EO 14028, NTIA, SSDF all reference or require SBOMs.
- Bonus: license auditing — the SBOM lists declared licenses; legal can scan for GPL/AGPL without developer manual review.

### 2. How does a purl connect an SBOM to a CVE database?

**Reference answer:**
- A purl (Package URL) is a standardized identifier: `pkg:npm/express@4.18.2`, `pkg:pypi/requests@2.31.0`.
- CVE databases (NVD, GHSA, OSV) use purls as package identifiers.
- When you scan an SBOM with Grype, it takes each component's purl and looks it up in the vulnerability DB — the same lookup as scanning an image directly, but the inventory step (walking the image) is pre-computed.
- The purl is the glue between "what's in my software" and "what's known-vulnerable."

### 3. What's the difference between a Cosign signature and a Cosign attestation?

**Reference answer:**
- A **signature** (`cosign sign`) signs the image digest — proves the image was signed by a trusted identity. The payload is the digest + signature.
- An **attestation** (`cosign attest`) signs a *statement about* the image — an SBOM, a SLSA provenance record, a test result. The payload is the predicate (e.g., the CycloneDX JSON) + signature.
- You can have both: sign the image (proves identity + integrity) and attest an SBOM (proves what's in it). Kyverno's `verifyImages` can require both.
- In practice: signature = "someone I trust built this." Attestation = "and here's their documented evidence of what's in it."

---

## Further reading

- [Syft GitHub](https://github.com/anchore/syft) — the tool; quickstart takes 5 minutes
- [CycloneDX spec](https://cyclonedx.org/) + [SPDX spec](https://spdx.dev/) — the formats
- [NTIA SBOM guidance](https://www.ntia.gov/sbom) — the US regulatory context
- [SLSA framework](https://slsa.dev/) — the broader supply-chain attestation model (SBOMs are one piece)
- [Dependency-Track](https://dependencytrack.org/) — an OSS platform for ingesting and querying SBOMs at scale; integrates with CycloneDX natively
- [VEX specification](https://www.cisa.gov/resources-tools/resources/minimum-requirements-vulnerability-exploitability-exchange-vex) — how to embed "not exploitable" statements in CycloneDX SBOMs

---

## The 4 dimensions (senior framing)

- **Tech**: Syft discovers packages across all package managers including Go binary build info. CycloneDX for tooling, SPDX for compliance. Cosign attestation attaches the SBOM to the image in-registry. Retroactive CVE matching via SBOM store (Dependency-Track, or simple artifact registry with Grype queries).
- **People**: the "which images contain log4j?" incident question is the convincing argument for SBOM investment. Run it as a table-top exercise — "if log4shell dropped today, how long to answer?" Without SBOMs: days. With: minutes. That lands.
- **CI/CD**: generate SBOM post-build, pre-push. Scan via SBOM (not image scan) for speed. Attest via Cosign. Store SBOM as artifact. Fail CI on HIGH/CRITICAL unfixed in SBOM scan (same policy as direct image scan — keep them aligned).
- **Operations**: maintain a SBOM store (Dependency-Track or OCI registry with artifact queries). Configure retention: keep SBOMs longer than images (images expire, SBOMs are the compliance record). When a critical CVE drops, the first SRE action is "query SBOM store for affected images" — make that runbook entry explicit.
