# Linux fundamentals

Containers are not a thing the kernel knows about — they are a **composition of Linux primitives**: namespaces (isolation), cgroups (resource limits), capabilities (privilege subsets), and `chroot`-like filesystem views. Every production K8s issue worth debugging eventually lands here: a pod is OOMKilled because its cgroup hit `memory.max`; DNS resolution fails intermittently because the conntrack table is full; a service is mysteriously slow because CFS throttling is hitting it at sub-second windows.

If you can't reason about these primitives, you're stuck operating Kubernetes at the YAML layer — fine until something breaks. This area is the foundation that lets every other topic in the repo click.

---

## Start here (curated external resources)

Read or watch in this order — each is free and high-signal.

1. **[Liz Rice — "Containers from Scratch" (45-min talk)](https://www.youtube.com/watch?v=8fi7uSYlOdc)** — builds a container live in Go using raw syscalls. Once you've seen this, you can never un-see what a container actually is.
2. **[Julia Evans — "What happens when you run a process in a container?"](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)** — short, friendly intro to the whole picture.
3. **[Kernel docs — cgroup v2 unified hierarchy](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)** — the canonical reference. You don't read this end-to-end; you grep it when you need a specific knob.
4. **[`man 7 cgroups`](https://man7.org/linux/man-pages/man7/cgroups.7.html)** and **[`man 7 namespaces`](https://man7.org/linux/man-pages/man7/namespaces.7.html)** — the authoritative man pages. Read once carefully.
5. **[LWN — "The current state of kernel page-table isolation" and adjacent series](https://lwn.net/Kernel/Index/)** — deeper kernel context when you want it. Use the index.
6. **[Brendan Gregg — Linux performance](https://www.brendangregg.com/linuxperf.html)** — the definitive resource for `perf`, `bpftrace`, and CPU profiling on Linux. Bookmark it.

If you only have 90 minutes total: watch the Liz Rice talk + read both man pages.

---

## Self-test

Try each out loud as if explaining to an interviewer. If you fumble, that's the topic to drill.

1. Walk me through what happens **at the cgroup level** when a pod exceeds its memory limit. Why is it killed, and how does the kernel choose what to kill?
2. What's the difference between **cgroup v1 and v2**? Why did v2 win?
3. Which **Linux namespaces** does a container live in? What does each isolate?
4. Why might a pod stay below its CPU limit but still feel slow? (Hint: CFS quota period.)
5. What's the difference between `memory.max`, `memory.high`, and `memory.low`?
6. Explain how `runc` actually creates a container from a config.json + rootfs. What syscalls are involved?
7. Why does intermittent DNS resolution sometimes correlate with high traffic? (Hint: UDP + conntrack.)
8. What does `/proc/self/cgroup` tell you? `/sys/fs/cgroup/<group>/memory.current`?
9. What's the difference between a **capability** and a **privilege**? Give an example.
10. Why does running with `--cap-add NET_ADMIN` not require `--privileged`?

---

## What we've covered (will grow as we do sessions)

| Sub-topic | File | Date covered |
|---|---|---|
| cgroups memory — `memory.max`, OOM mechanics, `oom_score_adj`, QoS mapping | [`cgroups-memory.md`](./cgroups-memory.md) | 2026-06-01 |
| cgroups CPU — `cpu.max`, `cpu.weight`, CFS quota+period, throttling, GOMAXPROCS trap | [`cgroups-cpu.md`](./cgroups-cpu.md) | 2026-06-02 |
| cgroups CPU — simple version (buffet analogy, single-vs-multi-thread intuition) | [`cgroups-cpu-simple.md`](./cgroups-cpu-simple.md) | 2026-06-02 |
| cgroups v1 vs v2 — unified hierarchy, PSI, kubelet cgroup-driver story, distro defaults | [`cgroups-v1-vs-v2.md`](./cgroups-v1-vs-v2.md) | 2026-06-05 |
| cgroups v1 vs v2 — simple version (USB hub analogy, PSI intuition) | [`cgroups-v1-vs-v2-simple.md`](./cgroups-v1-vs-v2-simple.md) | 2026-06-05 |
| cgroups IO — `io.max`, `io.stat`, blkio vs io, EBS gp2/gp3 throttling, CloudWatch signals | [`cgroups-io.md`](./cgroups-io.md) | 2026-06-05 |
| cgroups IO — simple version (toll booth analogy, EBS burst credit trap) | [`cgroups-io-simple.md`](./cgroups-io-simple.md) | 2026-06-05 |
| Linux namespaces — all 8 types, clone/unshare/setns, user ns UID mapping, cgroup ns | [`namespaces.md`](./namespaces.md) | 2026-06-05 |
| Linux namespaces — simple version (hotel room analogy, PID 1 trap, rootless containers) | [`namespaces-simple.md`](./namespaces-simple.md) | 2026-06-05 |
| runc internals — OCI spec, clone/pivot_root/exec sequence, containerd→runc chain, rootless | [`runc-internals.md`](./runc-internals.md) | 2026-06-05 |
| runc internals — simple version (flat-pack analogy, pivot_root vs chroot, docker exec) | [`runc-internals-simple.md`](./runc-internals-simple.md) | 2026-06-05 |
| iptables / nftables — 5 tables, 5 chains, kube-proxy DNAT chains, conntrack, nftables, Cilium | [`iptables-nftables.md`](./iptables-nftables.md) | 2026-06-05 |
| iptables / nftables — simple version (post office analogy, Service DNAT intuition) | [`iptables-nftables-simple.md`](./iptables-nftables-simple.md) | 2026-06-05 |
| systemd basics + journalctl — unit types, service anatomy, kubepods.slice, cgroup-driver, debug flows | [`systemd-basics.md`](./systemd-basics.md) | 2026-06-05 |
| systemd basics — simple version (factory supervisor analogy, 3-command kubelet diagnosis) | [`systemd-basics-simple.md`](./systemd-basics-simple.md) | 2026-06-05 |
| ulimits, file descriptors, inodes — 4 FD limit layers, inheritance, inode exhaustion | [`ulimits-fds.md`](./ulimits-fds.md) | 2026-06-05 |
| ulimits, file descriptors, inodes — simple version (key ring analogy, inode trap) | [`ulimits-fds-simple.md`](./ulimits-fds-simple.md) | 2026-06-05 |

---

## Recommended session order

⚠️ = explicit rejection gap

1. **cgroups v1 vs v2** — controller hierarchy, unified mode
2. ⚠️ **cgroups: memory** — `memory.max`, `memory.high`, OOMKilled mechanics, OOM score
3. **cgroups: CPU** — `cpu.weight`, `cpu.max`, CFS quota + period, throttling vs starvation
4. **cgroups: IO** — `io.max`, blkio, EBS throttling on EKS
5. **Namespaces** — pid, net, mnt, uts, ipc, user, cgroup; what each isolates
6. **How `runc` builds a container** — putting namespaces + cgroups + caps + chroot together
7. **iptables / nftables** — chains, tables, NAT, the relationship to kube-proxy and Istio init
8. **conntrack** — connection tracking, UDP DNS implications, intermittent timeouts at scale
9. **systemd** — units, services, journalctl basics (when you're debugging a node)
10. **File descriptors, ulimits, inodes** — the limits that bite production workloads

---

## Hands-on environment

Most exercises need a real Linux box (macOS can't fully demo cgroups or namespaces — Docker Desktop runs a Linux VM under the hood; you can exec into it).

Options in increasing order of fidelity:
- **Docker Desktop on macOS** — `docker run --rm -it --privileged debian bash` gives you a Linux shell. Limited but useful for cgroup file exploration.
- **Multipass / Lima** — local Linux VM, full namespace and cgroup access.
- **A small EC2 instance** — closest to production; needed for kernel-level experiments.
- **Inside a running pod** — `kubectl exec -it <pod> -- sh`; explore `/proc/<pid>/cgroup`, `/sys/fs/cgroup`, `/proc/<pid>/ns/`.

For exercises that need namespace creation (`unshare`, `nsenter`), you need at least Multipass — Docker Desktop's VM doesn't always cooperate.
