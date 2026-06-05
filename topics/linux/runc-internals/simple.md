# runc internals — the simple version (the flat-pack assembly analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **runc is like building flat-pack furniture. You get a box (rootfs) and an instruction sheet (config.json). runc follows the instructions — clone a process, set up walls (namespaces), install shelves (bind mounts + pivot_root), put a lock on the door (cgroups + capabilities + seccomp) — and hands you back an assembled container.**

The furniture is the OCI spec. runc is the assembler. containerd is the project manager who calls runc and gives it the box.

---

## The flat-pack analogy

You order flat-pack furniture. You get:
- **The box of parts** (rootfs — the container image unpacked into a directory)
- **The instruction sheet** (config.json — the OCI runtime spec)

An assembler (runc) reads the instructions:
1. "Put these four walls up" → create namespaces (isolation barriers)
2. "Install these shelves in the right spots" → bind-mount volumes at specific paths
3. "Swap the room's entry point" → pivot_root (change the process's `/` to the container rootfs)
4. "Apply the load rating labels" → cgroup limits (memory.max, cpu.max, io.max)
5. "Remove the power tools from the toolbox, leave only the right ones" → drop capabilities + apply seccomp
6. "Person moves in" → exec the container's entrypoint as PID 1

The furniture is now assembled. The assembler (runc) walks away. containerd watches it from a distance.

---

## The 2 confusing things

### 1. `pivot_root` vs `chroot` — why runc uses pivot_root

Both `chroot` and `pivot_root` change what the process sees as `/`. The difference:

`chroot(2)` just changes the process's root directory. But a privileged process can still escape it — there are well-known `chroot` escape techniques involving `open("/proc/1/root")` or `fchdir`. It's a soft boundary.

`pivot_root(2)` is harder to escape. It atomically swaps the old root filesystem for the new one and moves the old root to a location you specify (so you can unmount it). Once you unmount the old root, the process has no path back to the host filesystem. runc does exactly this: pivot_root → unmount old root → the container only sees its own rootfs.

**Simple way to remember it**: `chroot` is a "set a different working root" hint. `pivot_root` is "cut off access to the old root." runc uses `pivot_root` because security.

### 2. The containerd → runc relationship

You might hear "Docker runs containers" or "Kubernetes runs containers." The actual call chain is longer:

```
kubectl / docker CLI
        ↓
  Kubernetes / Docker daemon
        ↓
  containerd   (manages container lifecycle, image pull, snapshots)
        ↓
  runc         (the OCI runtime — makes the actual syscalls)
        ↓
  Linux kernel  (clone, pivot_root, cgroup writes, seccomp, etc.)
```

**containerd** manages the high-level lifecycle: pull image, unpack to rootfs, manage snapshots, restart logic, streaming logs. It knows nothing about kernel syscalls.

**runc** does the actual kernel work: namespace creation, filesystem pivot, cgroup setup, capability dropping. It then exits — it hands the container to the kernel to manage. runc is not a daemon.

Think of containerd as the general contractor and runc as the specialist who builds the foundation. The specialist does one job and leaves; the GC manages the ongoing project.

---

## What `docker exec` is doing (different from `docker run`)

`docker run` → containerd calls runc → runc creates all new namespaces, sets up rootfs, execs PID 1.

`docker exec` → containerd opens `/proc/<container-pid>/ns/<type>` files, calls `setns` for each namespace → runs the new command inside the *existing* namespaces. No new namespaces created. No rootfs setup. The new process joins the container's existing namespace bubble.

That's why `docker exec` is fast — it's not building the furniture from scratch. It's opening the door to the already-assembled room and adding a new person.

---

## Rootless runc — the quick story

Normal runc needs root on the host to create namespaces (except user namespace). Rootless runc:
1. Creates a **user namespace** first (no privilege needed)
2. Inside that user namespace, it has full capabilities — and can use them to create the other namespace types
3. For networking, it uses **slirp4netns**: a userspace network stack that tunnels traffic without needing real root. Slower than normal veth but doesn't require host network access.

Rootless mode is how rootless Docker, Podman, and Buildah work. It's significantly more secure: if the container runtime is compromised, the attacker has a non-root UID on the host.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What is runc? | The OCI-compliant runtime that makes the actual kernel syscalls to create containers. |
| What are the inputs to runc? | A rootfs directory (unpacked container image) + a config.json (OCI runtime spec). |
| What's config.json? | The instruction sheet: namespaces, mounts, cgroups, capabilities, seccomp, entrypoint. |
| Why pivot_root and not chroot? | pivot_root cuts off access to the old root; chroot is escapable. Security. |
| What's runc's relationship to containerd? | containerd is the manager; runc is the specialist. runc does one job and exits. containerd persists. |
| How does `docker exec` differ from `docker run`? | exec: joins existing namespaces via `setns`. run: creates new namespaces via `clone`. |
| What is rootless runc? | runc using user namespaces so container runtime doesn't need host root. Uses slirp4netns for networking. |

---

## Self-test (the killer question)

Out loud:

> **"From a config.json + rootfs, walk me through what syscalls runc makes to build a container."**

**Reference answer (intuitive version):**

"runc reads config.json to understand the target state: which namespaces to create, what to mount where, what cgroup limits to apply, what capabilities the process gets, and what the entrypoint is.

Then it starts the execution sequence. It calls `clone(2)` with flags for each namespace type — `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS` (mounts), `CLONE_NEWUTS`, `CLONE_NEWIPC`, and optionally `CLONE_NEWUSER` for rootless. The child process wakes up inside all those new namespaces.

Next, it sets up the filesystem. It bind-mounts the volumes specified in config.json, sets up `/dev`, `/proc`, `/sys`. Then it calls `pivot_root(2)` to swap the process's root to the container's rootfs directory and unmounts the old root — the container now can only see its own filesystem.

Then it applies resource limits — writes to `/sys/fs/cgroup/<pod>/{memory.max, cpu.max, io.max}`. Drops capabilities to just what config.json allows (using `capset(2)`). Applies the seccomp filter (`prctl(PR_SET_SECCOMP, ...)` or `seccomp(2)`). Finally calls `exec(2)` to replace the runc process with the container's entrypoint. runc is gone; the container is running."

---

## Further reading / watching

- **Liz Rice — "Containers from Scratch"** (YouTube, 45 min): [youtube.com/watch?v=8fi7uSYlOdc](https://www.youtube.com/watch?v=8fi7uSYlOdc) — she builds a mini-runc live
- **OCI runtime spec**: [github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec) — config.json format
- **runc source code**: [github.com/opencontainers/runc](https://github.com/opencontainers/runc) — readable Go

---

## Next: the deep-dive

When the flat-pack analogy feels obvious, jump to [`runc-internals.md`](./deep-dive.md). The deep-dive covers:

- config.json structure in detail (namespaces, mounts, capabilities, seccomp, hooks)
- The full clone/pivot_root/exec sequence with the actual syscalls
- `pivot_root` vs `chroot` in depth
- How containerd invokes runc (the shim process, the socket hand-off)
- How `docker exec` uses `setns` to join existing namespaces
- Rootless runc and slirp4netns in depth
- 4 self-test drills
- The 4 dimensions framing
