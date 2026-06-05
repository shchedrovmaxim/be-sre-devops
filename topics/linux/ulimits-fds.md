# ulimits, file descriptors, and inodes

> **Goal**: by the end you can answer the killer question — **"A high-connection-count service is crashing with 'too many open files' — walk me through the layers."** — distinguishing per-process, per-user, and system-wide limits, the soft/hard distinction, how container runtimes inherit and override limits, inode exhaustion, and the correct fix at each layer.

> Start with the [simple version](./ulimits-fds-simple.md) if you haven't — the key ring analogy makes the layering clear.

---

## The senior framing

"Too many open files" is one of the most common production failures for high-throughput services, and it's almost always fixable in 5 minutes once you know which layer the limit lives at. The failure is common because:

1. The default per-process limit (1,024) was set decades ago for interactive shells and is far too low for a service handling 10,000 concurrent connections.
2. There are four independent limits that can be the actual ceiling, and engineers typically only know one of them.
3. Container runtimes add another inheritance layer that complicates the picture.

The senior answer demonstrates all four layers and the inheritance chain.

---

## File descriptors — what they actually are

Every process has a **file descriptor table** in the kernel's `task_struct`. Each entry in the table is a numbered slot (0, 1, 2, 3, ...) pointing to an open file description. The slot number is the file descriptor (FD).

FD 0, 1, and 2 are always stdin, stdout, stderr. Every subsequent `open(2)`, `socket(2)`, `pipe(2)`, `epoll_create(2)`, `eventfd(2)`, `signalfd(2)` call takes the next available slot.

```bash
# How many FDs does process 1234 currently have open?
ls /proc/1234/fd | wc -l

# What are they?
ls -la /proc/1234/fd
# lr-x------ 1 root root 64 ... 0 -> /dev/null            ← stdin
# lrwx------ 1 root root 64 ... 1 -> socket:[123456]      ← stdout → socket
# lrwx------ 1 root root 64 ... 2 -> socket:[123457]      ← stderr
# lrwx------ 1 root root 64 ... 3 -> /etc/ssl/certs/...   ← cert file
# lrwx------ 1 root root 64 ... 4 -> socket:[789012]      ← client conn 1
# ...
```

For a service with 10,000 concurrent connections: at minimum 10,000 FDs for the sockets, plus FDs for config files, TLS cert files, log files, epoll instances, internal pipes, etc. Typical overhead is 50–200 FDs per service beyond the connection sockets.

---

## The 4 layers of FD limits

### Layer 1: per-process soft limit

The limit currently enforced for a specific process. Any `open(2)` or `socket(2)` call that would exceed this limit returns `EMFILE` ("Too many open files").

```bash
# View all resource limits for a running process:
cat /proc/<pid>/limits
# Limit                     Soft Limit   Hard Limit   Units
# Max open files            1024         1048576      files

# Or for your current shell:
ulimit -Sn   # soft limit
ulimit -Hn   # hard limit
```

A process can raise its own soft limit up to the hard limit without any privileges:
```c
struct rlimit rl;
getrlimit(RLIMIT_NOFILE, &rl);
rl.rlim_cur = rl.rlim_max;  // raise soft to hard
setrlimit(RLIMIT_NOFILE, &rl);
```

Many high-performance servers (nginx, postgres, redis) do this at startup. It's safe and requires no root.

### Layer 2: per-process hard limit

The ceiling that a non-root process can raise its soft limit to. Only root (or `CAP_SYS_RESOURCE`) can raise the hard limit.

Default hard limits vary by distro and how the process was started. Common values:
- Started by systemd with default settings: hard limit = **4,096** (low!) or **1,048,576** depending on distro version and configuration
- containerd systemd unit (typical modern setup): hard limit = **1,048,576**
- A shell started by a user with default PAM limits: hard limit = **65,536** or **65,536**

```bash
# On Ubuntu 22.04 default:
cat /proc/1/limits | grep "Max open files"
# Max open files  1073741816 1073741816  ← systemd PID 1 has very high limits
# A normal user process:
cat /proc/$(pgrep -n myapp)/limits | grep "Max open files"
# Depends on how myapp was started
```

### Layer 3: per-user limit (PAM + limits.conf)

Applies to interactive user sessions (SSH login, su). Configured in:

```bash
# /etc/security/limits.conf (or /etc/security/limits.d/*.conf):
# <domain>   <type>   <item>   <value>
ubuntu         soft     nofile   65536
ubuntu         hard     nofile   1048576
*              soft     nofile   65536
*              hard     nofile   1048576
```

These apply at login time through PAM (`pam_limits.so`). They do NOT apply to services started by systemd — systemd bypasses PAM entirely and uses its own limit configuration.

**The classic mistake**: adding limits to `/etc/security/limits.conf` and expecting it to fix a service that's started by systemd. It won't. Systemd services need `LimitNOFILE=` in the unit file.

### Layer 4: system-wide file limit

The total number of open file descriptions the kernel allows across all processes on the machine.

```bash
cat /proc/sys/fs/file-max        # max total open files system-wide
# 9223372036854775807            ← very large on modern systems (usually not the bottleneck)

cat /proc/sys/fs/file-nr         # three numbers: open, free, max
# 87456  0  9223372036854775807  ← 87,456 currently open, 0 free in free list, max=huge
```

When this limit is hit: any process on the system getting `ENFILE` ("Too many open files in system"). Different error code than `EMFILE` (per-process). Very rare on modern systems because the default is enormous. But it's configurable:

```bash
# Set system-wide limit (takes effect immediately, not persistent):
sysctl -w fs.file-max=2000000

# Make persistent:
echo "fs.file-max = 2000000" >> /etc/sysctl.d/99-file-limits.conf
sysctl -p /etc/sysctl.d/99-file-limits.conf
```

---

## Fixing FD limits at each layer

### For a service started by systemd

The correct and clean fix:

```ini
# /etc/systemd/system/myapp.service.d/override.conf (drop-in)
[Service]
LimitNOFILE=1048576
```

```bash
systemctl daemon-reload
systemctl restart myapp
# Verify:
cat /proc/$(pgrep myapp)/limits | grep "Max open files"
# Max open files  1048576  1048576
```

`LimitNOFILE=1048576` sets both soft and hard to 1,048,576. You can specify soft and hard separately: `LimitNOFILE=65536:1048576` (soft:hard).

The special value `infinity` sets the limit to `RLIM_INFINITY`. For `NOFILE` this typically maps to a very large finite number (kernel-dependent) rather than literally unlimited.

### For a container

FD limits for containers come from the container runtime. The chain:

```
systemd (LimitNOFILE for containerd.service)
  → containerd process (inherits systemd's limits)
    → runc (inherits containerd's limits OR reads config.json's rlimits)
      → container process (inherits runc's limits, OR overridden by config.json)
```

The `config.json` (OCI runtime spec) can set per-container rlimits:

```json
"process": {
  "rlimits": [
    { "type": "RLIMIT_NOFILE", "hard": 1048576, "soft": 1048576 }
  ]
}
```

In Kubernetes, you can't set this directly per-pod via standard PodSpec (as of Kubernetes 1.30 — KEP tracking this exists but isn't GA). The effective limit for a container is what containerd inherited from systemd, unless you configure containerd's own default:

```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    # No standard nofile config here — comes from systemd unit
```

In practice: set `LimitNOFILE=1048576` in containerd's systemd unit. All containers will then inherit that limit and can use up to 1,048,576 FDs.

### For a shell/user (interactive sessions)

```bash
# Temporary (this shell only):
ulimit -n 65536

# Permanent (next login):
# Add to /etc/security/limits.conf:
echo "ubuntu soft nofile 65536" >> /etc/security/limits.conf
echo "ubuntu hard nofile 1048576" >> /etc/security/limits.conf
# Log out and back in to take effect
```

---

## `ulimit` for other resources — beyond file descriptors

`ulimit` controls multiple resources. The ones that matter for K8s/containers:

| `ulimit` flag | Resource | Common production concern |
|---|---|---|
| `-n` | Open file descriptors (NOFILE) | High-connection services |
| `-u` | Max user processes (NPROC) | Fork bombs, thread count |
| `-s` | Stack size | Deeply recursive code |
| `-c` | Core dump size | Set to `unlimited` for debugging; `0` to disable |
| `-v` | Virtual memory size | Rarely hit; use cgroups instead |
| `-l` | Locked memory (mlock) | High-performance networking (DPDK), eBPF programs |

In K8s pod specs: you can't set most of these via standard fields. They flow through the container runtime.

---

## Inode exhaustion — the hidden "disk full"

### What is an inode?

Every file and directory on a filesystem has an **inode** — a metadata record that stores:
- File ownership and permissions
- Timestamps (atime, mtime, ctime)
- File size
- Pointers to the actual data blocks on disk

Inodes are pre-allocated at filesystem creation time. A filesystem has a fixed number of inodes regardless of available disk space. When all inodes are used, you cannot create new files — even if the disk has gigabytes free.

```bash
df -h        # disk space — looks fine
df -i        # inode usage — may be 100%
# Filesystem      Inodes   IUsed   IFree IUse% Mounted on
# /dev/nvme0n1p1 3276800 3276800       0  100% /
```

### What fills inodes in K8s clusters?

**Common culprits:**

1. `/var/log/pods/` and `/var/log/containers/` — Kubernetes creates symlinks and log files per container restart. With frequent restarts and no log rotation, millions of files accumulate.

2. `/tmp` — some applications create temporary files and don't clean up. High-throughput services may create thousands per second.

3. `/var/lib/kubelet/pods/` — volume directories and config maps. Each secret and config map key is a separate file.

4. Container image layers — each layer unpacked to `/var/lib/containerd/` creates many files.

```bash
# Find which directory has the most inodes used:
find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head -20
# Shows which parent directories have the most files

# Faster approach: check each mount point
for dir in /var /tmp /var/log /var/lib/kubelet /var/lib/containerd; do
  echo -n "$dir: "
  find $dir -maxdepth 1 2>/dev/null | wc -l
done
```

### Fixing inode exhaustion

- **Log rotation**: add a `logrotate` job for `/var/log/pods/` and `/var/log/containers/`; or configure containerd's built-in log rotation (`max-size`, `max-file` in containerd config)
- **Node lifecycle**: replace nodes rather than letting them accumulate state (immutable infrastructure approach — K8s node groups should be cycled regularly)
- **Filesystem choice**: `ext4` pre-allocates inodes; `xfs` uses dynamic inode allocation (inode exhaustion is less common on xfs)
- **Adjust inode ratio**: at filesystem creation time, `mkfs.ext4 -i 4096` creates more inodes (1 per 4096 bytes instead of the default 1 per 16KB). Only at creation time.

---

## Socket and pipe FDs — what high-connection services actually use

For a service handling N concurrent connections:
- N FDs for the client sockets
- 1 FD for the listening socket
- 1-2 FDs for epoll/kqueue/io_uring (event loop)
- M FDs for upstream connections (to databases, caches, etc.)
- Remaining for files: config, certs, logs

A rough rule: `N_clients + N_upstream_conns + 50` overhead. For 10,000 concurrent clients with 100 upstream connections, you need ~10,150 FDs. Default 1,024 limit → crash at client number 974.

```bash
# How many sockets does a process currently have?
cat /proc/<pid>/net/sockstat
# sockets: used 10234
# TCP: inuse 9876 orphan 0 tw 12 alloc 9876 mem 1234
# UDP: inuse 5 mem 0
```

---

## The interview answer in 60 seconds

> "There are four layers. The first is the per-process soft limit — the one actually enforced. Default is 1,024, which is laughably low for a high-connection service. Check it with `cat /proc/<pid>/limits | grep 'Max open files'`. The process can raise its soft limit up to the hard limit without any privileges.
>
> The hard limit is the ceiling a non-root process can reach. It's set when the process starts. For systemd services, it's set by `LimitNOFILE=` in the unit file. For containers, they inherit from the container runtime — so fix it in containerd's systemd unit, not the container itself.
>
> Third: per-user limits via `/etc/security/limits.conf`. These apply to interactive logins but systemd bypasses PAM, so they don't affect services started by systemd. Classic mistake.
>
> Fourth: system-wide limit in `/proc/sys/fs/file-max`. Usually enormous on modern systems — rarely the bottleneck.
>
> The correct fix for a containerized service: `LimitNOFILE=1048576` in the systemd unit for containerd. The container inherits that. `systemctl daemon-reload && systemctl restart containerd`.
>
> One more thing: I'd also check `df -i` for inode exhaustion — the error message can be identical to FD exhaustion, but the fix is completely different (clean up `/var/log/pods/` or cycle the node)."

---

## Self-test drills

### 1. A high-connection-count service is crashing with "too many open files." Walk me through the layers.

**Reference answer:**
- Layer 1: per-process soft limit — `cat /proc/<pid>/limits`. Default 1,024; service raises it to hard limit at startup (or should)
- Layer 2: hard limit — ceiling the process can reach. For containers: from container runtime (containerd) systemd unit `LimitNOFILE=`
- Layer 3: per-user via `/etc/security/limits.conf` — PAM-applied at login; does NOT affect systemd services
- Layer 4: system-wide `/proc/sys/fs/file-max` — usually `sysctl -w fs.file-max=N`; rarely the bottleneck
- Fix: `LimitNOFILE=1048576` in containerd.service unit, daemon-reload, restart
- Also check: `df -i` for inode exhaustion (same error symptom, different cause)

### 2. Why doesn't adding a line to `/etc/security/limits.conf` fix the issue for a systemd service?

**Reference answer:**
- `/etc/security/limits.conf` is applied by PAM (`pam_limits.so`) at login time — SSH sessions, `su`, interactive logins
- systemd starts services directly, bypassing the PAM login stack — limits.conf is never applied to systemd-managed services
- For systemd services: use `LimitNOFILE=` in the `[Service]` section of the unit file (or a drop-in override)
- Verify: `cat /proc/<service-pid>/limits | grep 'Max open files'` before and after the change

### 3. How do containers inherit FD limits, and how do you override them?

**Reference answer:**
- Container runtime (containerd) starts as a systemd service — inherits `LimitNOFILE=` from its unit file
- runc reads the OCI spec `config.json` `process.rlimits` field — can set per-container limits; in vanilla K8s this isn't exposed per-pod
- Container process inherits whatever runc sets — which inherits from containerd unless overridden in config.json
- Fix: raise `LimitNOFILE=1048576` in containerd.service (or drop-in). All containers on the node then have a 1M FD ceiling.
- Verify per-container: `ls /proc/<container-pid>/fd | wc -l` and `cat /proc/<container-pid>/limits`

### 4. What is inode exhaustion and how does it manifest in K8s?

**Reference answer:**
- Inodes are per-file metadata records, pre-allocated at filesystem creation. A fixed total count.
- When all inodes used: `ENOSPC` ("No space left on device") on file creation — even with disk space available
- `df -h` shows space; `df -i` shows inodes — the two can diverge completely
- In K8s: `/var/log/pods/` and `/var/log/containers/` accumulate millions of log symlinks from restarted containers
- Also: `/var/lib/kubelet/pods/` with large numbers of secret/configmap keys
- Fix: configure log rotation in containerd, clean up `/var/log/pods/` on old nodes, cycle nodes regularly (immutable infra), or use xfs (dynamic inode allocation avoids exhaustion)

---

## Further reading / watching

- **man 2 setrlimit / getrlimit**: the syscalls programs use to change their own limits
- **man 5 limits.conf**: `/etc/security/limits.conf` format
- **man 8 systemd.exec**: `LimitNOFILE` and all other limit directives in systemd unit files
- **Brendan Gregg — USE method**: [brendangregg.com/usemethod.html](https://www.brendangregg.com/usemethod.html) — utilization/saturation/errors for all resources including FDs

---

## The 4 dimensions (senior framing)

- **Tech**: four FD limit layers (per-process soft, per-process hard, per-user PAM, system-wide); `cat /proc/<pid>/limits` for current state; `ls /proc/<pid>/fd | wc -l` for current count; `LimitNOFILE=` in systemd unit is the canonical fix for services; `df -i` for inode exhaustion; `EMFILE` vs `ENFILE` error codes.
- **People**: "too many open files" is confusing because there are four independent limits and no error message tells you which layer you hit. Document the investigation flowchart in your runbook: check process limits → check if service is started by systemd → check if limits.conf even applies → check system-wide. Saves 30 minutes of wrong debugging.
- **CI/CD**: load tests should verify FD behavior under concurrent connection peak. Add a check in your service's startup: log the current FD limit (`/proc/self/limits`) and warn if soft limit < N (where N is your expected peak connection count × 1.5). Catching a misconfigured limit in CI beats paging at 3am.
- **Operations**: add `df -i` to your node health monitoring alongside `df -h`. Alert on inode usage > 80% — the 20% buffer gives you time to clean up before the node starts failing to create files. On long-running nodes (> 30 days), check `/var/log/pods/` inode count: `find /var/log/pods -type f | wc -l` — if it's in the millions, schedule log cleanup or node rotation.
