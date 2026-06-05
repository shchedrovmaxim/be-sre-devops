# ulimits, file descriptors, inodes — the simple version (the key ring analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc explains **one idea**:

> **"Too many open files" is a quota problem, not a "you used too many files" problem. Every process has a key ring with a limited number of key slots. Each open file, socket, or pipe takes one slot. When the slots are full, the next open() call fails. The fix is almost never "open fewer files" — it's "make the key ring bigger."**

That's it. The rest is figuring out which key ring is full (process, user, or system-wide) and how to enlarge it.

---

## The key ring analogy

Imagine every process carries a key ring. Each key on the ring represents an open file, socket, or pipe. The key ring has a maximum capacity — by default, **1,024 slots**.

When you try to open a 1,025th file:
- The kernel says: "your key ring is full" → `EMFILE: Too many open files`

But wait, there's also a system-wide key ring. The whole machine can only have so many total open files across all processes. If that system key ring is full, any process on the machine trying to open a new file gets:
- `ENFILE: Too many open files in system`

Most applications only ever see the first type (`EMFILE`) because the system limit is much larger. But high-connection-count services (databases, proxies, API gateways) hit the per-process limit regularly.

**Two limits. Same error message. Different fix.**

---

## The 2 confusing things

### 1. Soft vs hard limits — why you sometimes can't just change the limit

Every limit has two values:
- **Soft limit**: the current enforced limit. The process cannot open more files than this.
- **Hard limit**: the ceiling. The process can raise its own soft limit up to the hard limit — without root.

```bash
ulimit -Sn   # soft limit (what's enforced right now)
ulimit -Hn   # hard limit (ceiling a non-root process can reach)
```

A common default: soft = **1,024**, hard = **65,536** (or similar).

The trick: a process can call `setrlimit(2)` to raise its own soft limit up to the hard limit. No root needed. So if your app's soft limit is 1,024 but the hard limit is 65,536, you can just raise it in your code's init or in the launch script. If you need more than the hard limit, you need root (or systemd's `LimitNOFILE=`).

### 2. The inheritance story — containers inherit limits from their parent

When a container starts, it inherits the FD limits from the container runtime (containerd → runc → container process). If the container runtime was started by systemd with `LimitNOFILE=1048576` (as it typically is), the container inherits that limit.

But: whatever you set in the container's `ulimit -n`, it can only go up to the inherited hard limit. So if runc was started with a hard limit of 65,536, you cannot raise the container's limit above 65,536 without changing the containerd/runc systemd unit.

---

## The 3 limit types — where to look

| Layer | How to view | How to change |
|---|---|---|
| Per-process | `cat /proc/<pid>/limits` | `ulimit -n <value>` in shell, or `setrlimit(2)` in code |
| Per-user (persistent) | `ulimit -n` after login | `/etc/security/limits.conf` or systemd `LimitNOFILE=` |
| System-wide | `cat /proc/sys/fs/file-max` | `sysctl -w fs.file-max=2000000` |

For a high-connection-count service, you usually want to fix the **per-process** limit (via systemd unit file `LimitNOFILE=`) and verify the system-wide limit is not the ceiling.

---

## The inode exhaustion trap

Files consume two things: **disk space** and **inodes** (index nodes — the metadata records). You can run out of either independently.

The trap: `/var/log` fills up with millions of tiny log files. Each file takes one inode. The disk still has 40 GB of free space, but all the inodes are gone. Any new file creation fails with "No space left on device" even though `df -h` shows space available.

```bash
df -h    # check disk space — looks fine
df -i    # check inodes — aha, 100% used!
# Filesystem      Inodes  IUsed  IFree IUse%
# /dev/nvme0n1p1  3276800 3276800    0  100% /
```

The `df -i` check is the one most engineers forget. Add it to your "disk full" runbook.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| What is a file descriptor? | A numbered slot in the process's table for an open file, socket, or pipe. |
| What's the default per-process FD limit? | Soft limit 1,024. Hard limit often 65,536 or 1,048,576 depending on distro config. |
| What error means "per-process limit hit"? | `EMFILE: Too many open files` |
| What error means "system limit hit"? | `ENFILE: Too many open files in system` |
| How do I see a process's current limits? | `cat /proc/<pid>/limits` |
| How do I see how many FDs a process has open? | `ls /proc/<pid>/fd | wc -l` |
| How do I fix it for a systemd service? | Add `LimitNOFILE=1048576` to the service unit's `[Service]` section. |
| What's the system-wide FD limit? | `cat /proc/sys/fs/file-max` — usually in the millions on modern systems. |
| How do I check for inode exhaustion? | `df -i` — look at `IUse%`. 100% = you're out of inodes. |
| What fills up inodes in K8s? | `/var/log/pods/` or `/var/log/containers/` accumulating millions of log files when log rotation isn't configured. |

---

## Self-test (the killer question)

Out loud:

> **"A high-connection-count service is crashing with 'too many open files.' Walk me through the layers."**

**Reference answer (intuitive version):**

"First I'd identify which layer is the bottleneck. I'd check the process's own limits with `cat /proc/<pid>/limits | grep 'Max open files'`. If the current count (from `ls /proc/<pid>/fd | wc -l`) is near the soft limit, that's the first problem.

If the soft limit is already high but we're still hitting the error, I'd check the system-wide ceiling with `cat /proc/sys/fs/file-max` and `cat /proc/sys/fs/file-nr` to see current usage.

For the fix: if this is a container, I'd check the systemd unit file for containerd and raise `LimitNOFILE=`. I'd also check whether the container's Kubernetes PodSpec has a `securityContext.sysctls` or container `resources` that affects FD limits. The modern way to set FD limits for a containerized service is the systemd unit file for the container runtime — that's the actual ceiling the container inherits.

I'd also check inode usage with `df -i` — sometimes 'no space left on device' errors disguise as FD errors in logs when the real problem is inode exhaustion from log file accumulation."

---

## Further reading / watching

- **man 5 proc**: `man 5 proc` — the `/proc/<pid>/limits` format reference
- **man 2 setrlimit**: the syscall that programs use to change their own resource limits
- **ulimits reference**: `man bash` → search for `ulimit`

---

## Next: the deep-dive

When the key ring analogy feels obvious, jump to [`ulimits-fds.md`](./deep-dive.md). The deep-dive covers:

- All the FD limit layers (process, user, system-wide) with real file paths and numbers
- The soft/hard limit distinction and how to change each one
- `/etc/security/limits.conf` vs systemd `LimitNOFILE=` — which wins
- How K8s and container runtimes set FD limits for containers
- The default FD limit for containerd and how to override
- Inode exhaustion — `df -i`, what fills inodes in K8s clusters
- Socket and pipe FD usage patterns in high-concurrency services
- 4 self-test drills
- The 4 dimensions framing
