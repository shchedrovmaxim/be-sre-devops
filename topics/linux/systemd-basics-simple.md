# systemd basics + journalctl — the simple version (the process manager analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./systemd-basics.md) becomes easy.

This doc explains **one idea**:

> **systemd is like a supervisor at a factory. It starts processes (services) in the right order, keeps them running, restarts them when they crash, and writes a log of everything. When Kubernetes nodes break, the kubelet is a service that this supervisor manages — and `journalctl` is the log book you read to understand what happened.**

That's the whole thing. systemd manages services. When a K8s node is sick, the kubelet service is sick. And the logs are in journalctl.

---

## The factory supervisor analogy

A factory (your Linux node) has many machines (services: kubelet, containerd, docker, sshd, etc.). The supervisor (systemd) is responsible for:

- **Starting machines in the right order** (you can't start the assembly line before the power plant is running)
- **Restarting machines when they break** (machine crashed? Supervisor restarts it automatically)
- **Keeping the log book** (every machine's output goes to a structured log the supervisor maintains)
- **Shutting machines down cleanly** when the factory closes

You talk to the supervisor using `systemctl` ("start machine X", "show me machine X's status"). You read the log book using `journalctl`.

---

## The unit types — what systemd manages

systemd doesn't just manage processes. It manages "units" — different types of things it knows how to start/stop:

| Unit type | What it is | Example |
|---|---|---|
| `service` | A long-running process | `kubelet.service`, `sshd.service` |
| `socket` | A network socket (activates a service on demand) | `sshd.socket` |
| `timer` | A cron-like scheduled task | `certbot.timer` |
| `slice` | A cgroup resource scope for a group of services | `kubepods.slice`, `system.slice` |
| `mount` | A filesystem mount point | `/home.mount` |
| `target` | A group of units (like runlevels) | `multi-user.target` |

For K8s debugging: you care mainly about `service` and `slice`. kubelet runs as a `service`; pods run inside the `kubepods.slice` cgroup hierarchy.

---

## The 2 most important things

### 1. How to diagnose a sick kubelet in 3 commands

```bash
# Step 1: Is kubelet running at all?
systemctl status kubelet

# Step 2: What went wrong? (last 100 lines of logs)
journalctl -u kubelet -n 100 --no-pager

# Step 3: Any errors in the last 30 minutes?
journalctl -u kubelet --since "30 minutes ago" -p err
```

That sequence tells you 90% of what you need. Status tells you the current state. The logs tell you what happened. The error filter cuts through noise.

### 2. The `kubepods.slice` / cgroup-driver story

When Kubernetes runs on a node, systemd and the kubelet work together on cgroups:

- systemd creates `kubepods.slice` — the top-level cgroup for all pods
- Inside that, it creates sub-slices for each QoS class: `kubepods-guaranteed.slice`, `kubepods-burstable.slice`, `kubepods-besteffort.slice`
- Each pod gets a slice: `kubepods-burstable-pod<uid>.slice`
- Each container gets a scope inside: `cri-containerd-<cid>.scope`

The key config: `--cgroup-driver=systemd` on the kubelet tells it to delegate cgroup creation to systemd (via D-Bus). This is the preferred setup. Without it, the kubelet tries to manage cgroup paths directly and conflicts with systemd's own management.

```bash
# See the whole cgroup tree (like `tree` but for cgroups)
systemd-cgls
# -.slice
# ├─kubepods.slice
# │ ├─kubepods-burstable.slice
# │ │ └─kubepods-burstable-pod<uid>.slice
# │ │   └─cri-containerd-<cid>.scope
# ...
```

---

## Reading service unit files — the anatomy

A service unit file has three sections:

```ini
[Unit]
Description=Kubernetes Kubelet
Documentation=https://kubernetes.io/docs/
After=containerd.service    ← start after containerd
Requires=containerd.service ← if containerd dies, so do I

[Service]
ExecStart=/usr/bin/kubelet --config=/etc/kubernetes/kubelet.conf
Restart=always
RestartSec=5s
LimitNOFILE=65536   ← override ulimit for this service

[Install]
WantedBy=multi-user.target  ← start at this runlevel
```

- `[Unit]` — metadata + ordering dependencies
- `[Service]` — what to run, restart policy, environment, resource limits
- `[Install]` — what target enables this unit

Find unit files at `/lib/systemd/system/` (package defaults) or `/etc/systemd/system/` (local overrides). Always override in `/etc/systemd/system/` — don't edit `/lib/systemd/system/` directly or package updates will overwrite your changes.

---

## Intuition cheat sheet

| Question | Answer |
|---|---|
| How do I check if kubelet is running? | `systemctl status kubelet` |
| How do I see kubelet's logs? | `journalctl -u kubelet -n 100` |
| How do I see only errors? | `journalctl -u kubelet -p err` |
| How do I restart kubelet? | `systemctl restart kubelet` |
| How do I see live logs? | `journalctl -u kubelet -f` (follow, like `tail -f`) |
| What's a slice? | A cgroup scope systemd manages — `kubepods.slice` is the parent cgroup for all pods |
| What does `--cgroup-driver=systemd` do? | Tells kubelet to use systemd D-Bus to create cgroup paths, instead of writing directly |
| How do I see the full cgroup tree? | `systemd-cgls` |
| Where are unit files? | `/lib/systemd/system/` (defaults), `/etc/systemd/system/` (overrides) |

---

## Self-test (the killer question)

Out loud:

> **"A node is misbehaving — kubelet is in a weird state. Walk me through how you'd diagnose at the systemd layer."**

**Reference answer (intuitive version):**

"First I'd check `systemctl status kubelet` — this tells me immediately whether the service is running, how long it's been running (or when it last crashed), and the last few lines of log output. If it's in a crash loop, I'd see 'active (running) for 3s' repeatedly.

Then I'd pull the full logs with `journalctl -u kubelet -n 200 --no-pager` to see what happened. I'm looking for the error message just before each crash — usually it's a specific configuration issue, a missing file, a TLS cert problem, or a cgroup setup issue.

If the logs are noisy, I'd filter with `journalctl -u kubelet -p err --since '30 minutes ago'` to see only error-level entries.

Common things I'd see: the kubelet failing because containerd isn't started yet (check `systemctl status containerd`), or a cgroup driver mismatch (kubelet config says `systemd`, containerd says `cgroupfs`). For the cgroup mismatch I'd check `cat /etc/kubernetes/kubelet.conf | grep cgroupDriver` and `cat /etc/containerd/config.toml | grep SystemdCgroup`.

If the kubelet is running but the node is NotReady, I'd also look at `journalctl -u kubelet -f` live to see what it's currently complaining about — often it's a certificate expiration, a missing CNI plugin, or a failed volume mount."

---

## Further reading / watching

- **systemd man pages**: `man systemd.service`, `man journalctl`
- **systemd.io — The Freedesktop docs**: [systemd.io](https://systemd.io/) — comprehensive reference
- **K8s node debugging guide**: [kubernetes.io/docs/tasks/debug/debug-cluster/](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

---

## Next: the deep-dive

When the factory supervisor analogy feels obvious, jump to [`systemd-basics.md`](./systemd-basics.md). The deep-dive covers:

- Unit types in depth with real examples
- Service unit anatomy — all the common fields (Restart, RestartSec, LimitNOFILE, EnvironmentFile, ExecStartPre/Post)
- journalctl filtering — by unit, time, priority, field, JSON output
- The `kubepods.slice` hierarchy and cgroup-driver story in detail
- `systemd-cgls` for live cgroup tree exploration
- Common kubelet debug flows (status → logs → cgroup → restart)
- 4 self-test drills
- The 4 dimensions framing
