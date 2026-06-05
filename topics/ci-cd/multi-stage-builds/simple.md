# Multi-stage Docker builds — the simple version (the kitchen analogy)

> Read this first. Once the intuition clicks, the [deep-dive doc](./deep-dive.md) becomes obvious.

This doc only explains **one idea**:

> **Build your app in a fat container. Ship it from a thin one.**

That's the whole concept. Everything else (BuildKit, distroless, cache mounts) is just details on top.

---

## The kitchen vs the dining room

You're running a restaurant. A customer orders pasta.

**The kitchen** is messy:
- Knives, pots, a giant stove, raw flour, raw eggs, the cookbook, the dishwasher, your sous-chef, half-eaten ingredients from earlier orders, recipe drafts on the wall.

**The dining room** is clean:
- Just a plate of pasta on a table. That's it.

You **build the pasta in the kitchen**. You **serve the pasta in the dining room**.

You do not bring the kitchen into the dining room. You don't carry your pots out, you don't drop the flour bag on the customer's table, and you definitely don't bring your sous-chef to stand next to them while they eat.

That's multi-stage Docker builds.

---

## The same thing, but for containers

A Docker image is just a folder full of files that ship to production. When your container starts, it runs *inside* that folder.

The **single-stage** way: you do all your work inside one folder, then ship that whole folder.

```
Single-stage image contents:
├── your app                    ← what you want
├── node:20                     ← the cooking equipment
├── npm                         ← the cookbook
├── apt-get                     ← the dishwasher
├── bash                        ← the sous-chef
├── /usr/include (compilers)    ← the raw flour
├── your source code, .git      ← recipe drafts
└── 500 MB of unused Debian     ← yesterday's mess
```

The **multi-stage** way: you have two folders. The kitchen folder ("builder") has all the tools. The dining-room folder ("runner") is empty. When the meal is ready, you carry **only the plate** from the kitchen to the dining room. Then you throw the kitchen out.

```
Multi-stage image contents:
├── your app                    ← just the plate
└── nothing else
```

That's it. That's why multi-stage images are ~6× smaller and have ~100× fewer security vulnerabilities — there's nothing else in there to be vulnerable.

---

## The simplest possible example

Same Node app, two Dockerfiles.

### Single-stage (everything in one folder)

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

Final image: ~1.1 GB. Has bash, apt, compilers, your test files, your `.git`.

### Multi-stage (kitchen + dining room)

```dockerfile
# --- KITCHEN: do the messy work here ---
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install

# --- DINING ROOM: serve the plate ---
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/server.js ./
COPY --from=builder /app/node_modules ./node_modules
CMD ["server.js"]
```

Final image: ~180 MB. Has only Node + your code + your dependencies. No shell. No apt. No git. Nothing else.

The magic line is:

```dockerfile
COPY --from=builder /app/server.js ./
```

That says: **"go to the kitchen, grab just the plate, bring it here."** Everything else in the kitchen — the pots, the knives, the flour bag — stays behind and gets thrown out.

---

## What is "distroless"?

Distroless is just a fancy name for **"a dining room with no kitchen equipment in it."**

Most base images give you a complete mini-operating-system: a shell, package manager, common tools. That's like having a kitchen attached to every dining room.

A distroless base image gives you the bare minimum: just enough to run your app's runtime (Node, Python, Java, etc.) and nothing else. No `bash`. No `apt`. No `curl`. Just the food and the plate.

**Why does that matter?** If a hacker breaks into your container, they can't run `bash` (it's not there), can't install tools (`apt` is not there), can't poke around (`curl`, `wget`, `nc` are not there). They're stuck inside a dining room with literally nothing they can use.

---

## "But I need a shell to debug production"

Common pushback. The senior answer is:

> "I use `kubectl debug` for that. It attaches a temporary debug container to the running pod — fully equipped with shell and tools — without making my production image carry that weight forever."

```bash
kubectl debug -it my-pod --image=alpine --target=my-app
```

That spawns a debug container *next to* your prod container, sharing its network and process namespaces. You get your shell. Your prod image stays clean.

It's the difference between "I keep a chef sleeping in the dining room just in case" and "I call the kitchen when I need them."

---

## The intuition cheat sheet

| Question | Answer |
|---|---|
| "What is multi-stage?" | Build in one container, ship from another. Throw away the build container. |
| "Why?" | The build container has tools (compilers, dev deps, shell) you don't want in production. |
| "What is distroless?" | A base image with only your runtime — no shell, no package manager. |
| "Won't it be hard to debug?" | Use `kubectl debug` to attach a temporary container with tools. |
| "What's `COPY --from=builder`?" | The line that grabs *only the plate* from the kitchen. Skip it for anything you don't actually need to ship. |
| "Will it break anything?" | If your app needs files it doesn't ask for explicitly, you'll find out fast (it'll crash on startup). Add them to the COPY list. |

---

## Self-test (one question — the one that matters)

Out loud, as if to an interviewer:

> **"Why is multi-stage with distroless more secure than single-stage on `node:20`?"**

**Reference answer (intuitive version):**

"Because the single-stage image is essentially a whole Linux distribution that happens to have my app inside it. It ships with bash, apt, curl, compilers, my dev dependencies, my source code, my git history — hundreds of binaries an attacker can use if they ever get a shell. Multi-stage moves all of that into a 'builder' container that's thrown away. The final image only contains my app, my production dependencies, and the runtime — that's it. With distroless, there's not even a shell. So even if an attacker exploits my app, they have nothing to pivot with: no `bash` to run, no `curl` to phone home, no `apt` to install more tools. The image goes from ~1.1 GB with ~180 CVEs down to ~180 MB with essentially zero — and the security win is way bigger than the size win. Smaller attack surface, fewer scan findings, easier to pass admission policies."

If you got that out cleanly, you've internalized it. The deep-dive doc just adds the per-language cache mount patterns and the base-image decision table.

---

## Next: the deep-dive

When the kitchen analogy feels obvious, jump to [`multi-stage-builds.md`](./deep-dive.md). The deep-dive covers:

- The single-stage problem with concrete numbers
- BuildKit cache mounts (`--mount=type=cache`) per language
- The base-image decision table (scratch / distroless variants / alpine / debian-slim)
- The "musl libc" alpine trap
- Ephemeral debug containers in detail
- Hands-on Dockerfile pair you can actually build
- 4 self-test drills with reference answers
- The 4-dimensions framing (tech, people, CI/CD, ops)

The deep-dive is the reference. This doc is the mental model. Don't bother with the deep-dive until you can explain the kitchen analogy to a friend over coffee.
