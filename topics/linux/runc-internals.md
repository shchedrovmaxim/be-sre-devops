# runc internals — how a container is actually built

> **Goal**: by the end you can answer the killer question — **"From a config.json + rootfs, walk me through what syscalls runc makes to build a container."** — covering the OCI runtime spec, the clone/unshare sequence, pivot_root vs chroot, cgroup application, capability drop, seccomp setup, and the containerd → runc → kernel call chain.

> Start with the [simple version](./runc-internals-simple.md) if you haven't — the flat-pack analogy is the mental model for the detail below.

---

## The senior framing

Every time a senior engineer says "the container runtime creates a container," there's a specific, auditable sequence of syscalls behind that statement. Knowing that sequence matters because:

1. **Security review**: you can reason about what attack surfaces exist at each step. `pivot_root` vs `chroot` is a real security difference. Dropping capabilities after setup vs before exec is a real sequencing issue.
2. **Debugging**: when a container fails to start with a cryptic error, the runc log (and the OCI spec it tried to execute) tells you exactly where in the sequence it failed.
3. **Interview signal**: every K8s candidate knows "containers use namespaces and cgroups." Fewer can walk through the actual runc lifecycle.

---

## The OCI runtime spec — config.json

The Open Container Initiative (OCI) runtime spec defines what a container runtime receives as input. Two inputs:

1. **A rootfs** — a directory containing the container's filesystem (unpacked from the container image layers)
2. **config.json** — a JSON document describing everything else

### config.json structure (the important sections)

```json
{
  "ociVersion": "1.1.0",
  "process": {
    "terminal": false,
    "user": { "uid": 0, "gid": 0 },
    "args": ["/bin/myapp", "--port", "8080"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/",
    "capabilities": {
      "bounding":  ["CAP_NET_BIND_SERVICE", "CAP_KILL"],
      "effective": ["CAP_NET_BIND_SERVICE", "CAP_KILL"],
      "permitted": ["CAP_NET_BIND_SERVICE", "CAP_KILL"],
      "ambient":   ["CAP_NET_BIND_SERVICE"]
    },
    "rlimits": [
      { "type": "RLIMIT_NOFILE", "hard": 1024, "soft": 1024 }
    ],
    "noNewPrivileges": true
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "mounts": [
    { "destination": "/proc", "type": "proc", "source": "proc" },
    { "destination": "/dev",  "type": "devtmpfs", "source": "devtmpfs" },
    { "destination": "/data", "type": "bind", "source": "/host/data",
      "options": ["rbind", "rw"] }
  ],
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "mount" },
      { "type": "uts" },
      { "type": "ipc" },
      { "type": "user" }
    ],
    "uidMappings": [
      { "hostID": 100000, "containerID": 0, "size": 65536 }
    ],
    "gidMappings": [
      { "hostID": 100000, "containerID": 0, "size": 65536 }
    ],
    "cgroupsPath": "/kubepods/pod-<uid>/container-<cid>",
    "resources": {
      "memory": { "limit": 268435456 },
      "cpu":    { "quota": 100000, "period": 100000 }
    },
    "seccomp": {
      "defaultAction": "SCMP_ACT_ERRNO",
      "architectures": ["SCMP_ARCH_X86_64"],
      "syscalls": [
        { "names": ["read", "write", "open", "close", "..."],
          "action": "SCMP_ACT_ALLOW" }
      ]
    }
  },
  "hooks": {
    "prestart":  [{ "path": "/usr/local/bin/netsetup", "args": [...] }],
    "poststart": [{ "path": "/usr/local/bin/healthcheck", "args": [...] }]
  }
}
```

Key sections:
- `process`: the entrypoint, user, environment, capability set, rlimits
- `root`: where the rootfs directory is
- `mounts`: what to mount before exec (bind mounts, proc, dev, etc.)
- `linux.namespaces`: which namespace types to create (or join an existing one by providing a `path` to `/proc/<pid>/ns/<type>`)
- `linux.resources`: cgroup limits (memory, cpu quotas)
- `linux.seccomp`: syscall filter definition
- `hooks`: lifecycle hooks (prestart, createRuntime, startContainer, poststart, poststop)

---

## The runc execution sequence

### Phase 1: container creation (`runc create`)

1. **Parse config.json and validate** — runc validates the spec version and required fields.

2. **Create a pipe** for synchronization between parent and child processes.

3. **Fork a child with `clone(2)`:**
   ```c
   clone(child_func, stack, 
         CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS |
         CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER | SIGCHLD,
         arg);
   ```
   The child wakes up inside all the specified namespaces. The parent process (runc) continues in the host namespace.

4. **UID/GID mapping (user namespace):** The parent writes to `/proc/<child-pid>/uid_map` and `/proc/<child-pid>/gid_map` to establish the UID/GID mapping. This must happen BEFORE the child does anything that needs UIDs.

5. **Apply cgroup limits (parent side):** The parent writes the container's PID into the cgroup directory and sets resource limits:
   ```bash
   echo $child_pid > /sys/fs/cgroup/<path>/cgroup.procs
   echo "268435456" > /sys/fs/cgroup/<path>/memory.max
   echo "100000 100000" > /sys/fs/cgroup/<path>/cpu.max
   ```

6. **Child process setup (inside new namespaces):**

   a. **Bind-mount volumes** specified in `mounts` — each mount in the config.json's mounts array, in order. Typically `/proc`, `/dev`, `/sys`, then any data volumes.

   b. **`pivot_root(2)`** — swap the process root to the container's rootfs:
   ```c
   // Move into the rootfs directory
   chdir("/path/to/rootfs");
   // Create a temporary directory for the old root
   mkdir("old_root", 0755);
   // Swap rootfs and put old root at ./old_root
   pivot_root(".", "old_root");
   // Now "/" is the container rootfs, old host "/" is at /old_root
   chdir("/");
   // Unmount the old root — container can no longer access host filesystem
   umount2("/old_root", MNT_DETACH);
   rmdir("/old_root");
   ```

   c. **Set hostname** (`sethostname(2)`) from the config's `hostname` field.

   d. **Drop capabilities** — call `capset(2)` to reduce the process's capability set to only what's in `process.capabilities`.

   e. **Set `no_new_privs`** — call `prctl(PR_SET_NO_NEW_PRIVS, 1)` if `noNewPrivileges` is true. After this, `execve` can never grant the process more privileges via setuid or capabilities on the binary.

   f. **Apply seccomp filter** — call `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &filter)` or `seccomp(SECCOMP_SET_MODE_FILTER, ...)`. The filter is compiled from the `linux.seccomp` spec. After this, the process can only make the syscalls in the allowlist.

   g. **Signal parent** via the pipe: "I'm ready."

7. **Parent runs prestart hooks** — executables defined in `hooks.prestart` run in the host namespace (typically network setup, like CNI plugins). They use the container's PID to set up a veth pair and connect it to the bridge.

8. **Container is "created"** — it exists in a paused state (child is blocked on the sync pipe). The OCI `created` state.

### Phase 2: container start (`runc start`)

9. **Parent signals the child** via the pipe: "start."

10. **Child calls `exec(2)`** to replace itself with the container's entrypoint:
    ```c
    execve("/bin/myapp", argv, envp);
    ```
    The runc child process is now gone — replaced by the container's actual process. PID 1 in the container namespace is now running the application.

11. **Parent runs poststart hooks** if defined.

12. **runc parent exits** — runc is not a daemon. Its job is done. containerd's shim process (containerd-shim-runc-v2) keeps watching the container's PID for exit.

---

## `pivot_root` vs `chroot` — the security argument

### `chroot(2)` — the soft boundary

`chroot` just updates the process's root directory in the kernel's `task_struct`. The old filesystem is still mounted; the process just doesn't traverse above the new root normally.

Known escape: a privileged process (`CAP_SYS_CHROOT`) can escape a chroot by:
```c
// Classic chroot escape
mkdir("/tmp/escape");
chroot("/tmp/escape");
// now chdir to root relative:
for (int i = 0; i < 256; i++) chdir("..");
// We're now at the real filesystem root
chroot(".");  // re-root ourselves at the real /
```

Even without this specific exploit, chroot jails are considered a weak boundary. Security-focused runtimes don't rely on chroot alone.

### `pivot_root(2)` — the hard cut

`pivot_root` does two things atomically:
1. Makes the new directory the root filesystem
2. Moves the old root filesystem to a specified "put-old" directory

After `pivot_root`, runc unmounts the old root (`MNT_DETACH`). The process literally has no path to the host filesystem — the mnt namespace's mount table no longer contains the host root. There's no "go up to get back" because the path doesn't exist in this mount namespace.

Combined with the mnt namespace (`CLONE_NEWNS`), this is a true filesystem boundary. The kernel maintains separate mount tables; the container's table doesn't include the host root after the unmount.

**Why `chroot` still exists in runc**: for rootless containers where `pivot_root` requires specific conditions (the new root must be a mount point), runc falls back to a "chroot with additional hardening" approach, but this is a fallback, not the default.

---

## The containerd → runc relationship in detail

The actual invocation chain when Kubernetes starts a pod:

```
kubelet
  → CRI gRPC call → containerd
      → pulls and unpacks image (if not cached)
      → creates OCI bundle (rootfs + config.json)
      → forks containerd-shim-runc-v2 (the shim process)
          → shim forks runc create (passing bundle path)
              → runc does all the syscalls above
              → runc exits (!)
          → shim adopts the container PID (watches for exit)
          → shim serves stdio over a Unix socket (container logs)
      → containerd returns container ID to kubelet
```

Key detail: **runc exits after the container starts.** The shim process (containerd-shim-runc-v2) is what persists and manages the container's lifecycle. This allows containerd itself to be restarted without disrupting running containers — the shims survive independently.

The shim's responsibilities:
- Keep the container's exit status until someone reads it
- Manage container stdio (multiplexed over a socket)
- Forward signals to the container
- If configured, handle container restarts

The shim is a small Go binary; you can see it in `ps aux | grep shim` on a node running containers.

---

## How `docker exec` works (joining existing namespaces)

When you run `docker exec mycontainer bash`:

1. Docker/containerd finds the running container's init PID (e.g., 9876)
2. Opens file descriptors for each namespace:
   ```c
   int pid_ns_fd = open("/proc/9876/ns/pid", O_RDONLY);
   int net_ns_fd = open("/proc/9876/ns/net", O_RDONLY);
   int mnt_ns_fd = open("/proc/9876/ns/mnt", O_RDONLY);
   // etc.
   ```
3. Forks a new process (not using `clone` with namespace flags)
4. The new process calls `setns(fd, 0)` for each namespace fd:
   ```c
   setns(pid_ns_fd, CLONE_NEWPID);
   setns(net_ns_fd, CLONE_NEWNET);
   setns(mnt_ns_fd, CLONE_NEWNS);
   // etc.
   ```
5. Optionally applies cgroup (to run in the same resource group) by writing to `cgroup.procs`
6. Applies capability drop and seccomp from the exec config
7. Calls `exec(2)` with the requested command

Result: `bash` runs as a new process inside the container's existing namespaces. It shares the network, filesystem, PID space, etc. — but it's a separate process. It does NOT use `clone` to create new namespaces; it joins the existing ones via `setns`.

This is why `kubectl exec` feels like SSH into the container but is fundamentally different: it's a new process, not a persistent connection; there's no sshd; the process exits when the command exits.

---

## Rootless runc and slirp4netns

Standard runc requires `CAP_SYS_ADMIN` to create most namespace types (except user). Rootless runc avoids this:

**Step 1**: Create a user namespace first (no privilege required):
```c
// Parent (unprivileged user):
clone(child_func, stack, CLONE_NEWUSER | SIGCHLD, NULL);
// Write uid_map: 0 (container root) → 1000 (our real UID)
write_uid_map(child_pid, 0, 1000, 1);
```

**Step 2**: Inside the user namespace, the process has `CAP_SYS_ADMIN` (user-namespace scoped). Use it to create the other namespaces (`CLONE_NEWPID`, `CLONE_NEWNS`, `CLONE_NEWUTS`, `CLONE_NEWIPC`).

**Step 3**: For networking, kernel veth pairs need host network access (requires real `CAP_NET_ADMIN` on the host). Instead, rootless runc uses **slirp4netns**: a userspace TCP/IP stack that runs as a sibling process, tunneling traffic through a Unix socket. It provides network connectivity without requiring host network capabilities.

slirp4netns limitations:
- Performance: userspace network stack is slower than kernel veth (extra copy in/out of userspace)
- Port forwarding: binding ports < 1024 inside a rootless container doesn't require root, but the host-side exposure does
- No multicast/broadcast (kernel networking limitation of the slirp stack)

For Kubernetes: rootless containerd + runc is available but requires kernel 5.11+ for proper user namespace support. As of 2026, most production EKS nodes use standard (non-rootless) runc, with rootless being an option for specific security-sensitive workloads.

---

## The interview answer in 60 seconds

> "runc gets two things: a rootfs directory (the unpacked container image) and config.json (the OCI runtime spec describing namespaces, mounts, cgroup limits, capabilities, and seccomp). The OCI spec is the instruction sheet.
>
> runc's sequence: it calls `clone(2)` with flags for each namespace type — `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC` — which creates a child process already inside all those namespaces. The parent runc writes cgroup limits to `/sys/fs/cgroup/<path>/{memory.max,cpu.max}`.
>
> Inside the new namespaces, the child sets up mounts (bind-mounting volumes per the spec), then calls `pivot_root` — not chroot — to swap the process root to the container's rootfs directory and unmount the old host root. Pivot_root is the choice because it actually severs access to the host filesystem, while chroot can be escaped. Then the child drops capabilities to the configured set, sets `no_new_privs`, applies the seccomp syscall filter, and calls `exec(2)` on the container's entrypoint. runc then exits — it's not a daemon. The containerd shim watches the container PID.
>
> The contrast with `docker exec`: exec uses `setns(2)` to join the running container's existing namespaces — it opens `/proc/<pid>/ns/*` file descriptors and calls setns for each — rather than creating new namespaces."

---

## Self-test drills

### 1. From a config.json + rootfs, walk me through what syscalls runc makes to build a container.

**Reference answer:**
- Parse config.json, fork child with `clone(CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC[|CLONE_NEWUSER])`
- Parent: write uid_map/gid_map (if user namespace), write cgroup limits to `/sys/fs/cgroup/<path>`
- Child: bind-mount volumes per mounts spec, call `pivot_root` to swap rootfs and unmount old root
- Child: `sethostname`, `capset` to drop capabilities, `prctl(PR_SET_NO_NEW_PRIVS)`, apply seccomp filter via `prctl(PR_SET_SECCOMP)` or `seccomp()`
- Child: `execve(entrypoint, args, env)` — runc child is gone; container is running
- Parent: runs poststart hooks, then exits. containerd-shim-runc-v2 watches the container PID.

### 2. Why does runc use `pivot_root` instead of `chroot`?

**Reference answer:**
- `chroot` only changes the root pointer; the old filesystem is still mounted in the namespace; a privileged process can escape by traversing up with `chdir("..")`-after-nested-chroot tricks
- `pivot_root` atomically swaps the root and puts the old root at a specified path; runc then unmounts that old root with `MNT_DETACH`; the container's mount table no longer contains the host filesystem at all
- Combined with the mnt namespace, the host filesystem literally does not exist in the container's namespace — no path leads there
- `chroot` is an advisory boundary; `pivot_root` in a new mount namespace is a hard kernel-enforced one

### 3. What is the containerd-shim-runc-v2 process and why does it exist?

**Reference answer:**
- runc exits after starting the container — it's not a daemon
- The shim process (forked by containerd, persists after runc exits) watches the container PID for exit, keeps the exit status, manages stdio
- Allows containerd to be restarted (upgraded, crashed) without affecting running containers — the shim is an independent process
- Can be seen with `ps aux | grep shim` on a running node; there's one shim per container
- Shim serves container logs over a Unix socket that containerd reconnects to

### 4. How does rootless runc achieve container isolation without host root?

**Reference answer:**
- Creates a user namespace first (no privilege needed) — gets scoped `CAP_SYS_ADMIN` inside
- Uses that to create other namespace types (pid, net, mnt) from within the user namespace
- For networking: uses slirp4netns (userspace TCP/IP stack tunneled over a Unix socket) instead of kernel veth pairs — avoids needing `CAP_NET_ADMIN` on the host
- UID mapping: container UID 0 maps to the user's real UID on the host — rootless container "root" is actually a non-root user from the host's perspective
- Tradeoffs: networking performance is lower (slirp4netns overhead), port binding < 1024 requires additional setup

---

## Further reading / watching

- **Liz Rice — "Containers from Scratch"** (YouTube, 45 min): [youtube.com/watch?v=8fi7uSYlOdc](https://www.youtube.com/watch?v=8fi7uSYlOdc) — builds it live in Go
- **OCI runtime spec**: [github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)
- **runc source (libcontainer)**: [github.com/opencontainers/runc/tree/main/libcontainer](https://github.com/opencontainers/runc/tree/main/libcontainer) — `container_linux.go` is the startup sequence
- **Rootless containers guide**: [rootlesscontaine.rs](https://rootlesscontaine.rs/) — comprehensive coverage of rootless runc, containerd, Podman

---

## The 4 dimensions (senior framing)

- **Tech**: OCI spec (config.json + rootfs) as the interface; `clone → pivot_root → capset → seccomp → exec` as the sequence; `pivot_root` vs `chroot` as the security difference; containerd-shim-runc-v2 for shim-per-container resilience; `setns` vs `clone` for exec vs start.
- **People**: when a container fails to start with "permission denied" or "operation not permitted" in runc logs, the team needs to know which phase failed — was it namespace creation (user ns missing?), pivot_root (rootfs permissions?), capset (invalid capability?), or seccomp (syscall blocked?). Have a runbook for each phase.
- **CI/CD**: the OCI runtime spec `config.json` is generated by containerd — but you can inspect it. Add a CI step that runs `runc spec` to generate a baseline config and diffs it against your expected security posture (capabilities, seccomp profile, noNewPrivileges). Automating this catches security regressions before they reach production.
- **Operations**: if a pod fails to start with a cryptic runc error, check: `journalctl -u containerd | grep -A5 "runc error"`, then look at `/run/containerd/runc/<namespace>/<id>/` for the container state file. If you need to debug a container that crashed before its entrypoint: `runc run` with a debug shell instead of the original entrypoint, in the same bundle.
