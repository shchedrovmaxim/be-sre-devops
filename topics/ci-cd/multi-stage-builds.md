# Multi-stage Docker builds + BuildKit

> **Goal**: by the end you can answer the rejection-gap question — **"Why is multi-stage with a distroless final image more secure than single-stage on `node:20`?"** — naming concrete numbers (CVE counts, image size, attack surface) and the senior reason behind each.

> This was an explicit rejection feedback gap.

---

## The bridge from what you already know

You've used **Wiz** (image scanning + posture) and **Kyverno** (admission policy) in production. So you already know the *outcome* image hygiene drives: scans pass faster, fewer CVE waivers, policies that block runAsRoot or images >X size are easier to comply with. What we're going to do here is unpack the **mechanics** under those outcomes — what actually makes an image scan-clean and policy-compliant.

The core lever is the **Dockerfile itself.** A bad Dockerfile makes Wiz light up red and forces Kyverno waivers; a good one makes both invisible.

---

## The single-stage problem — what's wrong with the obvious approach

The "obvious" Dockerfile for a Node app:

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install              # installs all deps including devDependencies
COPY . .
RUN npm run build            # produces dist/
CMD ["node", "dist/server.js"]
```

This works. It's also a senior-interview red flag. Here's why:

| Problem | Numbers |
|---|---|
| **Massive base image** | `node:20` is ~1.1 GB. Most of that is the Debian userspace you don't run in production (`bash`, `apt`, `dpkg`, `gcc`, `python3`, ...) |
| **CVEs from the base userspace** | Typically **150-300 OS-level CVEs** at any moment (most low-severity, but Wiz still flags them) |
| **Dev dependencies in the runtime** | `npm install` installs `devDependencies` (test frameworks, type checkers, bundlers). All shipped to prod. |
| **Build tools in the runtime** | Compiler toolchains used for `node-gyp` deps stay in the image. Useful for an attacker who lands a shell. |
| **Source code in the runtime** | `COPY . .` ships `.git`, `.env.example`, `Dockerfile`, READMEs, tests. Sometimes secrets. |
| **Shell access for attackers** | `bash` + `sh` are sitting right there. Any RCE → instant shell → instant network egress. |

Senior interpretation: this image isn't an *application*. It's a **whole Linux distribution that happens to contain your application.** Attack surface is everything else.

---

## The multi-stage refactor — the senior version

Two stages: a **builder** that has all the dev tools, and a **runner** that has only the artifact + the runtime.

```dockerfile
# ---- Stage 1: builder ----
FROM node:20 AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci                       # deterministic install, devDependencies included

COPY . .
RUN npm run build                # produces dist/

RUN npm prune --production       # remove devDependencies from node_modules

# ---- Stage 2: runner ----
FROM gcr.io/distroless/nodejs20-debian12 AS runner
WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist        ./dist
COPY --from=builder /app/package.json ./

USER nonroot
CMD ["dist/server.js"]
```

What changed:

| Change | Effect |
|---|---|
| `FROM ... AS builder` for stage 1 | Build artifacts isolated. Anything not COPY'd into stage 2 is **discarded**. |
| `gcr.io/distroless/nodejs20-debian12` for stage 2 | ~150 MB. No shell, no package manager, no apt, no curl. Just the Node runtime + libc + minimal Debian userspace. |
| `npm prune --production` in builder | Strips devDependencies from `node_modules` before we copy it across. |
| Selective `COPY --from=builder` | Only the artifact + production deps + package.json land in runner. No source, no `.git`, no test files. |
| `USER nonroot` | Distroless ships a `nonroot` user (UID 65532). Kyverno's `runAsNonRoot` policy is now free. |
| `CMD ["dist/server.js"]` (no "node" prefix) | Distroless sets `node` as the entrypoint. You just pass args. |

### Concrete numbers (Node example)

| | Single-stage | Multi-stage + distroless |
|---|---|---|
| Image size | ~1.1 GB | ~180 MB |
| OS CVEs (Trivy scan) | ~180 typical | **0-5** typical |
| Layers | ~8 | ~12 (slightly more — fine, layers aren't size) |
| Has a shell? | Yes (`/bin/sh`, `/bin/bash`) | **No** |
| Runs as root by default? | Yes | **No** (`nonroot` user) |

That's the senior answer to the rejection-gap question. ~150 CVEs and ~1 GB of attack surface gone for the cost of 20 lines of Dockerfile.

---

## BuildKit cache mounts — the senior performance trick

Naive Dockerfile pain: every time `package.json` changes, `npm ci` re-downloads everything. Even worse for Go modules, Maven, Cargo.

BuildKit (the modern Docker builder) lets you mount a **persistent cache directory** into the build that isn't part of the final image:

```dockerfile
# syntax=docker/dockerfile:1.7

FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./

RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

The cache directory persists across builds. On a warm cache, `npm ci` reuses tarballs from `/root/.npm` instead of re-downloading. **10× faster typical** for big dep trees.

Crucial detail: **the cache directory is NOT in any image layer.** It's mounted at build time, then unmounted. So your final image stays small.

### Per-language cache mount patterns

```dockerfile
# Node (npm)
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Node (pnpm)
RUN --mount=type=cache,target=/pnpm/store \
    pnpm install --frozen-lockfile

# Python (pip)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Go modules + build cache (two mounts)
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app ./...

# Maven
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B package -DskipTests

# Cargo (Rust)
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

### How to actually use BuildKit

BuildKit is the default builder on Docker 23+ and Docker Desktop. The first line of the Dockerfile activates the modern syntax:

```dockerfile
# syntax=docker/dockerfile:1.7
```

In CI you typically also set `DOCKER_BUILDKIT=1` or use `docker buildx build`. For multi-platform images you pretty much have to use buildx anyway.

**Cache scope in CI** is a separate question — local cache, registry cache (`--cache-to=type=registry,...`), GitHub Actions cache, S3 cache. Pick one based on where your runners live. We're not going deep on that here; the mechanism above is the building block.

---

## Choosing the final base — the decision table

Once you have multi-stage, your runner stage can be tiny. Which "tiny" do you pick?

| Base | Size | Has shell? | When to use | Trap |
|---|---|---|---|---|
| `scratch` | 0 B | No | Statically-linked binaries (Go, Rust with musl). The purest option. | Nothing inside — no libc, no certs. You must COPY them in (e.g. `/etc/ssl/certs/ca-certificates.crt`). No glibc, so dynamically-linked binaries don't run. |
| `gcr.io/distroless/static` | ~2 MB | No | Same as scratch but with CA certs, timezone data, `/etc/passwd` (`nonroot` user). The right choice for Go binaries. | Same — only static binaries. |
| `gcr.io/distroless/base` | ~20 MB | No | Dynamically-linked binaries (Python with C extensions, some Go with cgo). Has glibc + libssl. | Bigger than static; needed only when you actually need a libc. |
| `gcr.io/distroless/nodejs20` etc. | ~180 MB | No | Language-runtime apps (Node, Python, Java). Distroless does the runtime + minimal libc. | Still smaller than alpine + a runtime, and 0-5 CVEs typical. |
| `alpine:3.20` | ~7 MB | Yes (`sh`) | When you genuinely need shell + package manager but want tiny. | **musl libc** (not glibc) causes weird incompatibilities — DNS resolution edge cases, Python wheels not matching, etc. Tempting but bites you. |
| `debian:12-slim` / `ubuntu:22.04` | ~75 MB | Yes (`bash`) | When you need a real shell + apt for ops or debugging. | Larger; comes with the standard CVE surface even slim'd. |

### Senior decision heuristic

- **Static-compiled Go / Rust** → `gcr.io/distroless/static`. Smallest, safest, ships in seconds.
- **Dynamic Go / Python with C extensions / Java** → matching distroless variant (`base`, `python3`, `java`, `nodejs`).
- **You absolutely need a shell** (debug builds, init containers that run scripts) → `debian:12-slim`. Don't ship to prod.
- **alpine** → think twice. The size saving rarely justifies the musl pain. Distroless gives you a smaller *or* comparable image without musl issues.

### The "but I need to debug a prod container" rebuttal

Common pushback: "if there's no shell, I can't `kubectl exec` to debug."

The senior counter: **`kubectl debug --image=alpine ... --target=<container>`** attaches an ephemeral debug container with shell, networking, and tooling, sharing the target's namespaces. Your prod image stays clean. Ephemeral containers (GA in K8s 1.25+) exist exactly to solve this.

---

## Security — why this matters beyond image size

You already know this from Wiz scans and Kyverno policies, but to articulate it in an interview:

1. **Reduced attack surface, not just smaller** — the size win is incidental; the security win is the *count of binaries that can be exploited*. A shell, `curl`, `tar`, `python` all in the image means an attacker who lands an RCE can pivot. Distroless: nothing to pivot with.
2. **Fewer CVEs to triage** — fewer packages = fewer Wiz findings. Critical CVE on `bash` is a real problem; not having `bash` is not.
3. **No package manager** — even if an attacker has a shell, they can't `apt install` more tools. Egress detection wins time.
4. **Forced non-root** — distroless's `nonroot` user makes `runAsNonRoot: true` Kyverno policies trivial to comply with. No `securityContext` arguments to win in code review.
5. **No build artifacts** — Compilers, source code, test files, `.git` history — none of those leak to prod.

### The Wiz / Kyverno connection

You've seen Wiz block image promotion on critical CVEs and Kyverno block pods that run as root. Multi-stage + distroless is what makes those gates *easy to pass without compromise*. Teams that ship `FROM node:20` are constantly negotiating waivers; teams that ship distroless never need to.

Senior framing: **"a clean Dockerfile is the cheapest security investment you can make."** Every Wiz finding you don't have is a finding you don't have to triage.

---

## Hands-on — see the contrast yourself

Build the same trivial Node app two ways and compare.

### Step 1 — create the app

```bash
mkdir multi-stage-demo && cd multi-stage-demo
cat > package.json <<'EOF'
{
  "name": "demo",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "build": "echo built" },
  "dependencies": { "express": "^4.18.0" },
  "devDependencies": { "typescript": "^5.0.0" }
}
EOF

cat > server.js <<'EOF'
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('hello'));
app.listen(3000);
EOF
```

### Step 2 — single-stage Dockerfile

```bash
cat > Dockerfile.single <<'EOF'
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
EOF

docker build -f Dockerfile.single -t demo:single .
```

### Step 3 — multi-stage Dockerfile

```bash
cat > Dockerfile.multi <<'EOF'
# syntax=docker/dockerfile:1.7

FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev
COPY server.js ./

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/server.js ./
USER nonroot
CMD ["server.js"]
EOF

docker build -f Dockerfile.multi -t demo:multi .
```

### Step 4 — compare sizes

```bash
docker images | grep demo

# REPOSITORY  TAG     SIZE
# demo        multi   ~180 MB     ← distroless + production deps only
# demo        single  ~1.1 GB     ← node:20 + everything
```

That's a ~6× drop, on a hello-world app where the source code is one file. On a real app the absolute saving is bigger.

### Step 5 — confirm there's no shell in the multi-stage image

```bash
docker run --rm -it demo:multi sh
# → "exec: sh: not found" or similar

docker run --rm -it demo:single sh
# → root@xxxxxxx:/app#   ← root shell, with apt, curl, etc.
```

That shell-vs-no-shell contrast is *the* security delta you describe in an interview.

### Step 6 (optional) — quick CVE scan

If you want a concrete number to cite in interviews, run Trivy locally:

```bash
brew install trivy

trivy image demo:single | tail -5
# Total: ~180 vulnerabilities (CRITICAL: 2  HIGH: 8  MEDIUM: 30  LOW: 140)

trivy image demo:multi | tail -5
# Total: ~0 vulnerabilities
```

(Trivy = OSS local equivalent of what Wiz does in cloud. Don't dwell; you already know image scanning at a higher tier. The point is just to confirm the CVE delta is real.)

### Cleanup

```bash
docker rmi demo:single demo:multi
```

---

## The interview answer in 60 seconds

> "A single-stage `FROM node:20` image is around 1.1 GB and ships the entire Debian userspace — `bash`, `apt`, `curl`, compilers, the lot. Trivy lights up with 150+ OS CVEs. The image runs as root by default, so any RCE lands an attacker with `bash` and a working `apt` to install more tools. Multi-stage moves all of that into a discarded builder stage. The runtime stage is `distroless/nodejs20` — about 180 MB, no shell, no package manager, runs as `nonroot`. CVE count drops to roughly zero. Beyond size, the point is that **attack surface = number of things an attacker can use after they land a shell**. With distroless, there's nothing — they can't even spawn `/bin/sh`. Wiz scans pass, Kyverno's `runAsNonRoot` is free. The trade-off is debugging: you can't `kubectl exec sh`. Modern answer is **ephemeral debug containers** — `kubectl debug --image=alpine --target=<container>` — which attach a debug image without polluting the prod one."

---

## Self-test

### 1. Why is multi-stage with a distroless final image more secure than single-stage on `node:20`?

**Reference answer:**
- Attack-surface argument: distroless ships ~5 binaries vs ~500 in node:20. No shell, no `apt`, no `curl` = no pivot tools if an attacker lands an RCE.
- CVE argument: typically 0-5 CVEs vs 150+ for node:20 base. Fewer scans-blocking findings (Wiz), fewer waivers to negotiate.
- Privilege argument: distroless runs as `nonroot` by default. Kyverno `runAsNonRoot` policies are trivially satisfied.
- Build artifact argument: builder stage and all its dev tools / source files / `.git` are *discarded* — they don't ship to prod.
- Bonus: smaller images push/pull faster, which matters during rollouts.

### 2. What's a BuildKit cache mount and why doesn't it bloat the final image?

**Reference answer:**
- `RUN --mount=type=cache,target=<path>` mounts a directory at build time only. It's NOT part of any image layer.
- Used for package-manager caches: `~/.npm`, `~/.cache/pip`, `~/.m2`, `/go/pkg/mod`.
- Persists across builds (per builder cache), so `npm ci` reuses tarballs and runs ~10× faster on warm cache.
- The mount is unmounted before the layer is committed. Final image size is unaffected.
- Activates with the modern syntax directive: `# syntax=docker/dockerfile:1.7`.

### 3. "I can't `kubectl exec` to debug if there's no shell." How do you respond?

**Reference answer:**
- Use ephemeral debug containers: `kubectl debug -it <pod> --image=alpine --target=<container>`. K8s attaches a sidecar with shell + tooling that shares namespaces with the target — full network, /proc, filesystem visibility on the target.
- Requires K8s 1.25+ (GA). Available on every managed K8s by now.
- Senior point: your *prod image* shouldn't be designed for ad-hoc debugging. Debugging is a separate concern with a proper tool. Shipping a shell "just in case" trades persistent security for occasional convenience — bad ratio.

### 4. When would you pick scratch vs distroless/static vs distroless/base?

**Reference answer:**
- `scratch` — for purely static binaries that need nothing else. Zero bytes of base. You have to COPY in CA certs, timezone data, etc. — usually not worth the manual work.
- `distroless/static` — same niche as scratch but ships CA certs, timezone data, `/etc/passwd`, `nonroot` user. **The right answer for static Go binaries.**
- `distroless/base` — adds glibc + libssl. Use when your binary is dynamically linked (cgo Go, some Rust, etc.).
- Don't use `distroless/static` for dynamic binaries — they'll fail to load shared libraries at runtime.

---

## The 4 dimensions (senior framing)

- **Tech**: multi-stage with builder + distroless runner; BuildKit cache mounts per language; `runAsNonRoot` via distroless's `nonroot` user; ephemeral debug containers replace shell-in-prod-image.
- **People**: Dockerfiles are usually owned by app teams. Give them a base template (or a Helm-style scaffold) so they don't have to learn distroless cold. Pair-review a few PRs to lock the pattern in.
- **CI/CD**: enforce in policy — Kyverno or admission webhook that **blocks** images bigger than X MB, images with `runAsRoot`, or images failing scan. Block at PR time via a CI check, not at deploy time. Use Wiz at the deploy gate, Trivy/Grype as the local-dev fast feedback loop.
- **Operations**: monitor image pull duration as a deploy-time SLI (smaller images = faster rollouts under pressure). When investigating a "weird prod issue," use `kubectl debug` ephemeral containers instead of building a "debug" image variant. Document the multi-stage pattern in the platform-team runbook so newcomers don't reinvent it badly.
