# Linux namespaces — the 8 namespace types

> **Goal**: by the end you can answer the killer question — **"Walk me through what each namespace isolates and how `runc` actually creates a container."** — naming all 8 namespace types, describing what each hides, and explaining the `clone/unshare/setns` syscall sequence.

> Start with the [simple version](./namespaces-simple.md) if you haven't — the hotel analogy is the mental model for the detail below.

---

## The senior framing

A container is not a kernel primitive. "Container" is an application-level concept that maps to a composition of kernel primitives: **namespaces** (isolation), **cgroups** (resource limits), **capabilities** (privilege subsets), and **seccomp** (syscall filtering).

Namespaces specifically answer: "what does the process think the world looks like?" Each namespace type wraps one system resource and gives the process an isolated view of it. The kernel enforces the boundaries.

The practical value of knowing this: when a container does something unexpected (leaks a hostname to a neighbor, shares a network segment it shouldn't, sees the host's process tree), the debugging path goes through namespaces. You can't reason about container security or networking without this foundation.

---

## The 8 namespace types

### 1. `pid` — process IDs

**Clone flag**: `CLONE_NEWPID`
**File**: `/proc/<pid>/ns/pid`

Creates an isolated PID number space. The first process in the new namespace has PID 1; subsequent processes count from there. From the host, these processes have different PIDs (e.g., PID 18432 on the host is PID 1 inside the container).

**Key behaviors:**
- A pid namespace can be nested. A container can create child pid namespaces.
- `kill(2)` from outside the namespace works using the host PID. Inside the namespace, you only see your namespace's PIDs.
- Orphaned processes in the namespace are reaped by PID 1 of that namespace (not the host's PID 1). This is why PID 1 in containers must handle `SIGCHLD` — if it doesn't, zombies accumulate.

**The PID 1 trap:** Linux delivers `SIGTERM` to PID 1 only if PID 1 has registered a handler. A bare shell or simple application as PID 1 will not terminate on `SIGTERM`. After `terminationGracePeriodSeconds`, Kubernetes sends `SIGKILL`. Fix: use `tini` (a minimal init) or `dumb-init` as the container's entrypoint. Both reap zombies and forward signals.

```dockerfile
# Fix for the PID 1 problem
FROM ubuntu:22.04
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["myapp"]
```

### 2. `net` — network

**Clone flag**: `CLONE_NEWNET`
**File**: `/proc/<pid>/ns/net`

Creates a complete isolated network stack: interfaces, routing tables, iptables rules, conntrack table, sockets. The new namespace starts with only a loopback (`lo`) interface.

Container networking works by:
1. Creating a `net` namespace for the container
2. Creating a veth pair (virtual ethernet pair — like a cross-over cable in software): one end in the host namespace, one in the container namespace
3. The host-side veth connects to a bridge (e.g., `docker0` or a CNI bridge); the container side becomes `eth0` inside the container

```bash
# See the network namespace of a running container
docker inspect -f '{{.State.Pid}}' mycontainer
# → 1234
nsenter --target 1234 --net -- ip addr
# Shows the container's network interfaces, not the host's
```

**Senior gotcha:** Kubernetes shares the `net` namespace across all containers in a pod. That's why containers in the same pod can communicate via `localhost` and share port space. The pause container (also called the "infra" container) creates the net namespace and holds it open; app containers join with `setns`.

### 3. `mnt` — mount points

**Clone flag**: `CLONE_NEWNS` (historical name — this was the first namespace, hence "NS")
**File**: `/proc/<pid>/ns/mnt`

Isolates the filesystem view. Each mount namespace has its own mount table — bind mounts, proc, devpts, etc. Changes to the mount table in one namespace are not visible in other namespaces (with some nuances around `MS_SHARED` propagation).

Used by containers to: provide a different rootfs (the container image), hide host filesystems, and bind-mount volumes at specific paths.

`pivot_root(2)` changes the root filesystem for the process. `runc` uses this (not `chroot(2)`) because `pivot_root` is safer — it can't be escaped as easily as `chroot`. More in [`runc-internals.md`](./runc-internals.md).

### 4. `uts` — hostname and domain name

**Clone flag**: `CLONE_NEWUTS` (Unix Time Sharing — old name)
**File**: `/proc/<pid>/ns/uts`

Isolates `gethostname(2)` and `getdomainname(2)`. Inside a container, `hostname` returns what was set in the container's namespace (usually the pod name in K8s), not the host's hostname.

Simplest namespace — no surprising gotchas.

### 5. `ipc` — inter-process communication

**Clone flag**: `CLONE_NEWIPC`
**File**: `/proc/<pid>/ns/ipc`

Isolates System V IPC: shared memory (`shmget`), message queues (`msgget`), and semaphores (`semget`). Also isolates POSIX message queues (`/dev/mqueue`).

In practice: mostly relevant for databases and legacy applications that use shared memory for IPC. Two containers in different ipc namespaces cannot communicate via shared memory. Containers in the same K8s pod share the ipc namespace (same as net) — so they CAN use shared memory to communicate.

### 6. `user` — user and group IDs

**Clone flag**: `CLONE_NEWUSER`
**File**: `/proc/<pid>/ns/user`

The powerful one. Maps UIDs and GIDs inside the namespace to different UIDs/GIDs on the host. A process can be UID 0 (root) inside the user namespace while being UID 100000 on the host.

**The key privilege exception**: creating a user namespace does NOT require any privileges — an unprivileged process can do it. Once inside, it has all capabilities (within the user namespace). This is the foundation of rootless containers.

UID mapping is configured via `/proc/<pid>/uid_map`:

```bash
# Format: <namespace-uid> <host-uid> <count>
cat /proc/$(docker inspect -f '{{.State.Pid}}' mycontainer)/uid_map
# 0    100000    65536
# UID 0 inside maps to UID 100000 on the host; 65536 UIDs mapped consecutively
```

**Gotcha**: many operations still require real kernel-level privileges, not just user-namespace root. Mounting most filesystems, changing network settings outside the user namespace's net namespace, loading kernel modules — these require real `CAP_SYS_ADMIN` on the initial user namespace (the host). User namespaces are great for isolation but not a substitute for actual security hardening.

### 7. `cgroup` — cgroup root

**Clone flag**: `CLONE_NEWCGROUP` (Linux 4.6)
**File**: `/proc/<pid>/ns/cgroup`

Hides the host's cgroup tree. Inside a cgroup namespace, the process sees its own cgroup as the root (`/`). Without a cgroup namespace, a process inside a container would see:

```
/sys/fs/cgroup/kubepods/pod-<uid>/container-<cid>/
```

With a cgroup namespace, it sees:

```
/sys/fs/cgroup/
```

This is why `/proc/self/cgroup` inside a container shows `0::/` (v2) or `12:memory:/` (v1 with the cgroup namespace set up correctly) — the path is relative to the namespace root, not the host root.

Practical implication: a container cannot discover other containers' cgroup paths or monitor host-level cgroup usage. Security boundary.

### 8. `time` — system clock

**Clone flag**: `CLONE_NEWTIME` (Linux 5.6, 2020)
**File**: `/proc/<pid>/ns/time`

Allows per-namespace offsets for `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME`. Primarily useful for:
- Testing: simulate a container running at a different time without affecting the host clock
- VM-in-container: adjust clocks after live migration
- Checkpoint/restore (CRIU) workloads

Rarely used in practice outside of CRIU scenarios. Most container runtimes don't use it by default.

---

## The syscall trio: `clone`, `unshare`, `setns`

### `clone(2)` — create a process in new namespaces

```c
pid_t clone(int (*fn)(void *), void *child_stack,
            int flags, void *arg, ...);
```

The `flags` field combines `CLONE_NEW*` constants. Pass `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER` to create a process in 6 new namespaces simultaneously. That's what `runc` effectively does when starting a container.

### `unshare(2)` — detach from current namespace

```c
int unshare(int flags);
```

The calling process (or thread) creates new namespaces and moves itself into them. Doesn't fork. The shell command `unshare` wraps this:

```bash
# Create a new pid namespace and network namespace for this shell
sudo unshare --pid --net --fork --mount-proc bash
# Inside: PID 1 is your bash, no network interfaces visible
```

`--fork` is needed with `--pid` because the process calling `unshare` cannot be PID 1 in its own pid namespace — it needs to fork a child that becomes PID 1.

### `setns(2)` — join an existing namespace

```c
int setns(int fd, int nstype);
```

`fd` is a file descriptor from `/proc/<pid>/ns/<type>`. Opens another process's namespace and joins it. This is how `docker exec` works: it opens file descriptors for the target container's namespace files, then calls `setns` for each one before exec'ing the new process.

```bash
# Manual nsenter (wraps setns)
nsenter --target 1234 --pid --net --mount -- bash
# Opens /proc/1234/ns/{pid,net,mnt}, calls setns for each, runs bash
```

---

## Inspecting namespaces in production

Every process's namespaces are symlinks in `/proc/<pid>/ns/`:

```bash
ls -la /proc/1/ns/
# cgroup -> cgroup:[4026531835]
# ipc    -> ipc:[4026531839]
# mnt    -> mnt:[4026531840]
# net    -> net:[4026531992]
# pid    -> pid:[4026531836]
# pid_for_children -> pid:[4026531836]
# time   -> time:[4026531834]
# time_for_children -> time:[4026531834]
# user   -> user:[4026531837]
# uts    -> uts:[4026531838]
```

The inode number in brackets (`4026531992`) uniquely identifies the namespace. Two processes sharing the same inode number share that namespace.

```bash
# Compare host process and container process net namespace
cat /proc/1/ns/net       # host init's net namespace
docker inspect -f '{{.State.Pid}}' mycontainer  # → 8765
cat /proc/8765/ns/net    # container's net namespace — different inode
```

### Using `lsns` to list all namespaces on a node

```bash
lsns
# TYPE   NPROCS  PID USER  COMMAND
# net         1 8765 root  myapp
# pid         1 8765 root  myapp
# mnt         1 8765 root  myapp
```

### `nsenter` patterns

```bash
# Enter just the network namespace — useful to run curl from container's perspective
nsenter --target $(docker inspect -f '{{.State.Pid}}' mycontainer) --net -- curl http://10.0.0.5

# Enter all namespaces — equivalent to `docker exec`
nsenter --target $(docker inspect -f '{{.State.Pid}}' mycontainer) \
  --pid --net --mount --uts --ipc -- bash

# On a K8s node: nsenter into a pod
kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}'
# → containerd://abc123
crictl inspect abc123 | jq .info.pid   # → 9876
nsenter --target 9876 --net -- ss -tunlp
```

---

## User namespace — UID mapping in depth

The uid_map file format:

```
<inside-uid-start> <outside-uid-start> <count>
```

Example: `0 1000 1` maps UID 0 inside to UID 1000 outside (single mapping). `0 100000 65536` maps UIDs 0–65535 inside to UIDs 100000–165535 outside.

Security implications:
- Files owned by UID 100000 on the host appear owned by UID 0 inside the container
- A process running as UID 0 inside can read those files
- But on the host, UID 100000 is a regular unprivileged user — so the same process, from the host's perspective, can only access files owned by UID 100000

This is the rootless container security model: strong isolation within the user namespace, no host root.

**Capability scope**: capabilities (like `CAP_NET_BIND_SERVICE`) granted inside a user namespace are only effective within that namespace. A process can have `CAP_SYS_ADMIN` inside its user namespace without having any special powers on the host.

---

## The cgroup namespace and the `0::/` mystery

When you run `cat /proc/self/cgroup` inside a container on a v2 system:

```
0::/
```

That `/` is the root of the cgroup namespace — not the host's cgroup root. Without a cgroup namespace, you'd see:

```
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/cri-containerd-<cid>.scope
```

The cgroup namespace hides that path. The container can't see its position in the broader host cgroup tree, and can't infer the presence of other containers.

---

## Privilege requirements — which namespaces need root?

| Namespace | Unprivileged creation? | Notes |
|---|---|---|
| `user` | Yes | The exception. Foundation of rootless containers. |
| `pid` | No (needs `CAP_SYS_ADMIN` or parent user ns) | Can be created inside a user namespace |
| `net` | No | Requires `CAP_SYS_ADMIN` on the host unless inside a user namespace |
| `mnt` | No | Same |
| `uts` | No | Same |
| `ipc` | No | Same |
| `cgroup` | No | Same |
| `time` | No | Same |

The pattern: first create a user namespace, then you have the capabilities within it to create all other namespace types without needing host root.

---

## The interview answer in 60 seconds

> "There are 8 namespace types. The classic six: `pid` (process IDs — container gets its own PID 1), `net` (complete isolated network stack — how containers get their own IP), `mnt` (filesystem view — the container's rootfs from the image), `uts` (hostname isolation), `ipc` (shared memory and message queues), and `user` (UID/GID mapping — foundation of rootless containers, where container root maps to a non-root host UID). The newer two: `cgroup` (hides the host cgroup tree so the container sees itself at cgroup root) and `time` (clock offsets, mostly for CRIU checkpoint-restore workloads).
>
> When `runc` creates a container, it reads an OCI `config.json`, calls `clone(2)` with flags for each namespace type, which creates the child process already inside the new namespaces. Then it sets up the filesystem via bind mounts and `pivot_root`, applies cgroup limits (writes to `/sys/fs/cgroup/...`), drops capabilities to the configured set, applies the seccomp syscall filter, and execs the container's entrypoint. That process wakes up as PID 1 in its own isolated namespace bubble.
>
> `docker exec` is the `setns` story: it opens file descriptors from `/proc/<pid>/ns/` for the running container, calls `setns` for each, then execs the new command inside the container's existing namespaces — joining the hotel room without creating a new one."

---

## Self-test drills

### 1. Walk me through what each namespace isolates and how `runc` actually creates a container.

**Reference answer:**
- pid: own PID space, PID 1. net: isolated network stack. mnt: own filesystem view. uts: hostname. ipc: shared memory / message queues. user: UID/GID mapping. cgroup: hides host cgroup tree. time: clock offsets.
- runc: reads config.json → `clone()` with namespace flags → bind mount rootfs → `pivot_root` → apply cgroup limits → drop capabilities → apply seccomp → `exec` entrypoint
- K8s: pause container creates net/ipc namespace; app containers join via `setns`

### 2. How does `docker exec` work at the syscall level?

**Reference answer:**
- Opens file descriptors for `/proc/<target-pid>/ns/{pid,net,mnt,uts,ipc}` (and optionally user)
- Calls `setns(fd, 0)` for each, joining the target container's existing namespaces
- Then forks/execs the requested command in that namespace context
- The new process is in the existing container's namespaces but is a separate process (different PID from host view)
- Contrast with starting a new container: `clone()` creates new namespaces; `exec` joins existing ones

### 3. What are user namespaces and why do rootless containers need them?

**Reference answer:**
- User namespaces map UID/GID inside the namespace to different UID/GID on the host
- Unprivileged processes can create user namespaces — the key exception to the "need root" rule
- Inside a user namespace, the process has UID 0 (root) and full capabilities — but those map to a non-root UID on the host
- Rootless Docker/Podman: container daemon and all containers run under a non-root user; user namespace provides the "root inside container" experience without any host privilege
- Security property: `CAP_SYS_ADMIN` inside the user namespace ≠ `CAP_SYS_ADMIN` on the host

### 4. What does `/proc/self/cgroup` show inside a container versus on the host?

**Reference answer:**
- Host (v2): `0::/kubepods.slice/kubepods-burstable.slice/pod-<uid>.slice/container-<cid>.scope` — full path
- Inside container with cgroup namespace: `0:://` — the cgroup namespace makes the container's cgroup look like the root
- v1: one line per controller, each showing the controller name and cgroup path
- Why it matters: a container can't read its cgroup ancestors or infer other containers' existence. Security isolation.

---

## Further reading / watching

- **Liz Rice — "Containers from Scratch"** (YouTube, 45 min): [youtube.com/watch?v=8fi7uSYlOdc](https://www.youtube.com/watch?v=8fi7uSYlOdc)
- **man 7 namespaces**: [man7.org/linux/man-pages/man7/namespaces.7.html](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- **man 2 clone**: [man7.org/linux/man-pages/man2/clone.2.html](https://man7.org/linux/man-pages/man2/clone.2.html)
- **man 1 nsenter / man 1 unshare**: part of util-linux on any Linux host

---

## The 4 dimensions (senior framing)

- **Tech**: 8 namespace types and their `CLONE_NEW*` flags; `clone/unshare/setns` syscall sequence; `/proc/<pid>/ns/` inspection; user namespace UID mapping; cgroup namespace hides host cgroup tree; privilege requirements per namespace type.
- **People**: namespace isolation boundaries are your first mental model for "could container A affect container B?" — network namespace sharing (K8s pod), ipc namespace sharing (within-pod shared memory), user namespace (rootless daemon vs root daemon). Train your team to inspect namespaces before concluding "these containers can't interact."
- **CI/CD**: security scanners should check whether containers run with `--privileged` (bypasses namespace isolation entirely — all host namespaces shared). Enforce `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, and `runAsNonRoot: true` as default PSS (Pod Security Standards) restrictions. These map directly to namespace and capability controls.
- **Operations**: when debugging a "container can see/access something it shouldn't," your first tool is `lsns` on the node and `nsenter` to inspect the container's actual namespace contents. For a security incident, check whether the container has user namespace isolation (`cat /proc/<pid>/uid_map`) — rootless containers have mappings; privileged containers often don't.
