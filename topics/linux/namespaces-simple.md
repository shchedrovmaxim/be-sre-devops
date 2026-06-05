# Linux namespaces — the simple version (the hotel room analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./namespaces.md) becomes easy.

This doc explains **one idea**:

> **A Linux namespace is a wrapper around a system resource that makes one process see an isolated version of it, different from what other processes see. A container is just a process that lives inside a bundle of 8 different namespaces at once.**

That's it. The container runtime isn't magic — it calls three syscalls (`clone`, `unshare`, `setns`) to assign namespaces, drops the process inside, and the kernel does the rest.

---

## The hotel room analogy

You check into a hotel. Your room has its own:
- **Door number** (you think you're at room 101, but hotel staff see you at floor 3, wing B, room 42)
- **Wifi network** (guests see only the hotel's guest SSID; the management VLAN is invisible)
- **TV guide** (your room's TV shows channel 5; another floor shows a different channel 5)
- **Mini-bar** (stocked just for your room — you can't raid the neighboring room's)

You, the guest, feel like you have a complete private room. But from the hotel management perspective, you're just one entity in a shared building, subject to the building's fire exits, power grid, and security cameras.

A Linux container is the same thing. The process inside thinks it's on its own computer with its own PID 1, its own network, its own filesystem. From the host's perspective, it's one process in a shared kernel, isolated by namespace wrappers.

---

## The 8 namespace types — one-line each

| Namespace | What it isolates | Hotel analogy |
|---|---|---|
| `pid` | Process IDs. Inside the namespace, the first process is PID 1. | Your room's phone is extension 1 — the hotel has thousands of extension 1s. |
| `net` | Network interfaces, routing tables, iptables, ports. | Your room's wifi is a private SSID. |
| `mnt` | Mount points and filesystem view. | Your mini-bar contents — the neighboring room has a different one. |
| `uts` | Hostname and domain name. | Your room's welcome card says "Welcome, Guest" — another room's says something else. |
| `ipc` | Shared memory, message queues, semaphores. | Your room's intercom only connects to your own phone line. |
| `user` | User and group IDs (UID/GID mapping). | You're "guest" in your room; the front desk sees your real name. |
| `cgroup` | The cgroup root the process sees. | Your room's thermostat shows just your room's temperature settings, not the whole HVAC. |
| `time` | System clock and monotonic clock offsets. | Your room can show a different timezone clock (rarely used). |

The six "classic" ones are pid, net, mnt, uts, ipc, user. `cgroup` and `time` were added later (Linux 4.6 and 5.6 respectively).

---

## The 2 most confusing ones

### pid namespace — why PID 1 matters in containers

In a pid namespace, the first process gets PID 1. Inside the container, it looks like there's only one process tree. But the host sees it with a completely different PID (say, PID 18432).

Why does PID 1 matter? Because Linux sends `SIGTERM` to PID 1 when a process tree needs to shut down, and **PID 1 is special: it doesn't receive signals unless it explicitly installs signal handlers**. A naive shell or app running as PID 1 won't respond to `SIGTERM` and will be force-killed with `SIGKILL` after the grace period. This is the "zombie reaping / SIGTERM in containers" problem. Using `tini` or `dumb-init` as PID 1 fixes it.

### user namespace — how rootless containers work

Normally, creating namespaces requires `CAP_SYS_ADMIN`. User namespaces are special: **an unprivileged process can create a user namespace and become root inside it** — without being root on the host.

How? Through UID/GID mapping. Inside the user namespace, the process has UID 0 (root). But the kernel maps that to a real, unprivileged UID on the host (e.g., UID 100000). Rootless Docker and Podman both use this: the container daemon runs as a normal user, creates a user namespace, and runs containers with "root" inside — that root maps to an unprivileged UID outside.

---

## Inspecting namespaces live

Every process's namespaces are visible at `/proc/<pid>/ns/`:

```bash
ls -la /proc/$$/ns/
# lrwxrwxrwx ... cgroup -> cgroup:[4026531835]
# lrwxrwxrwx ... ipc    -> ipc:[4026531839]
# lrwxrwxrwx ... mnt    -> mnt:[4026531840]
# lrwxrwxrwx ... net    -> net:[4026531992]
# lrwxrwxrwx ... pid    -> pid:[4026531836]
# lrwxrwxrwx ... user   -> user:[4026531837]
# lrwxrwxrwx ... uts    -> uts:[4026531838]
```

Two processes sharing the same namespace number (e.g., `net:[4026531992]`) share that resource. A container process and the host process have different numbers for most namespaces.

Enter a container's namespace without starting a new process:

```bash
nsenter --target <pid> --net -- ip addr  # see the container's network
nsenter --target <pid> --pid --mount -- ps aux  # see the container's process list
```

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What's a namespace? | A kernel wrapper that makes a process see an isolated version of one system resource. |
| How many namespace types? | 8: pid, net, mnt, uts, ipc, user, cgroup, time. |
| What syscall creates a namespace? | `clone(2)` (at fork) or `unshare(2)` (in the current process). |
| What syscall joins an existing namespace? | `setns(2)`. |
| How does `docker exec` work? | It calls `setns` to join the target container's namespaces, then runs the new process there. |
| What's special about the user namespace? | Unprivileged processes can create them. UID 0 inside maps to a real non-root UID on the host. |
| Why does PID 1 in a container matter? | It won't handle signals unless it installs handlers. Use `tini` or `dumb-init` to handle this. |

---

## Self-test (the killer question)

Out loud:

> **"Walk me through what each namespace isolates and how `runc` actually creates a container."**

**Reference answer (intuitive version):**

"There are 8 namespace types. The core ones: `pid` gives the container its own PID space (container's PID 1 is host's PID 18432 or whatever). `net` gives it its own network interfaces, routing table, and ports. `mnt` gives it its own filesystem view via a chrooted rootfs. `uts` gives it its own hostname. `ipc` isolates shared memory and message queues. `user` maps UIDs — this is how rootless containers work, mapping container root to a non-root UID on the host. `cgroup` hides the host's full cgroup tree so the container sees itself at the root.

When `runc` creates a container, it reads an OCI `config.json`, then calls `clone(2)` with flags for each namespace it needs to create — `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | ...`. That one call creates the process in all the new namespaces at once. Then runc sets up the filesystem (bind mounts + `pivot_root`), applies cgroup limits, sets capabilities, applies the seccomp filter, and finally execs the container's entrypoint. The container process wakes up as PID 1 in its own isolated world."

---

## Further reading / watching

- **Liz Rice — "Containers from Scratch"** (YouTube, 45 min) — builds this live in Go. If you watch one thing, this is it.
- **man 7 namespaces**: [man7.org/linux/man-pages/man7/namespaces.7.html](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- **man 2 clone**: [man7.org/linux/man-pages/man2/clone.2.html](https://man7.org/linux/man-pages/man2/clone.2.html)

---

## Next: the deep-dive

When the hotel analogy feels obvious, jump to [`namespaces.md`](./namespaces.md). The deep-dive covers:

- All 8 namespace types with their kernel flags and real examples
- The `clone/unshare/setns` syscall sequence
- How to inspect `/proc/<pid>/ns/` and use `nsenter`
- User namespace internals — UID/GID mapping files
- The gotcha that some namespaces require privilege (and why user ns is the exception)
- How the cgroup namespace hides paths from containers
- 4 self-test drills
- The 4 dimensions framing
