# systemd basics + journalctl

> **Goal**: by the end you can answer the killer question — **"A node is misbehaving — kubelet is in a weird state. Walk me through how you'd diagnose at the systemd layer."** — covering unit types, service anatomy, journalctl filtering, the kubepods.slice hierarchy, cgroup-driver setup, and the full kubelet debug flow.

> Start with the [simple version](./simple.md) if you haven't — the factory supervisor analogy is the mental model.

---

## The senior framing

K8s engineers spend a lot of time in `kubectl`. When a node goes NotReady, the instinct is to look at K8s events and pod conditions. But most node-level failures originate below the Kubernetes layer — in systemd, in the kubelet service, in containerd. The fastest path to "why is this node broken" goes through `systemctl` and `journalctl`, not `kubectl describe node`.

The other senior insight: systemd is the cgroup manager for most Linux nodes, including K8s nodes. The `kubepods.slice` hierarchy that your pods run in was created by systemd. Understanding how systemd and the kubelet coordinate cgroup management explains several common misconfiguration failures.

---

## Unit types — systemd manages more than just processes

| Unit type | Extension | What it manages | K8s relevance |
|---|---|---|---|
| `service` | `.service` | A long-running process | kubelet, containerd, sshd |
| `socket` | `.socket` | A network or Unix socket (activates service on first connection) | containerd.socket |
| `timer` | `.timer` | A scheduled task (replaces cron) | Certificate rotation, log rotation |
| `slice` | `.slice` | A cgroup hierarchy node — groups of services | kubepods.slice, system.slice, user.slice |
| `mount` | `.mount` | A filesystem mount | /var/lib/kubelet.mount |
| `automount` | `.automount` | An automount trigger | Rare |
| `target` | `.target` | A group of units (like runlevels) | multi-user.target, network-online.target |
| `path` | `.path` | A filesystem path watcher | Rare in K8s contexts |

For Kubernetes node debugging: you primarily interact with `service` and `slice` units.

---

## Service unit anatomy

```ini
# /lib/systemd/system/kubelet.service (typical EKS/AL2023 layout)

[Unit]
Description=Kubernetes Kubelet
Documentation=https://kubernetes.io/docs/home/
After=network-online.target containerd.service
Wants=network-online.target
Requires=containerd.service

[Service]
User=root
ExecStartPre=/sbin/iptables -F KUBE-IPTABLES-HINT || true
ExecStart=/usr/bin/kubelet \
  --config=/etc/kubernetes/kubelet/kubelet-config.json \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --register-node=true \
  --v=2

Restart=always
RestartSec=5
StartLimitInterval=0

# Resource limits for the kubelet process itself
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

# Environment from file (useful for injecting node-specific config)
EnvironmentFile=-/etc/sysconfig/kubelet

KillMode=process
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

### Key fields explained

**`[Unit]` section:**

- `After=`: start this unit only after the listed units have started. Does not create a hard dependency — if `network-online.target` fails, kubelet still starts.
- `Wants=`: soft dependency — systemd tries to start the listed units alongside this one; failure doesn't block.
- `Requires=`: hard dependency — if `containerd.service` fails or stops, kubelet also stops.
- `BindsTo=`: stronger than Requires — if the bound unit stops for any reason, this unit is also stopped immediately.

**`[Service]` section:**

- `ExecStart=`: the main command. Only one allowed (unlike `ExecStartPre` which can have multiple).
- `ExecStartPre=`: commands to run before `ExecStart`. The `|| true` pattern suppresses non-zero exit so it doesn't block start.
- `Restart=always`: restart regardless of exit code (clean exit, crash, timeout, etc.). Use `on-failure` for "restart only on error."
- `RestartSec=5`: wait 5 seconds before restarting. Prevents tight crash loops.
- `StartLimitInterval=0`: disable the start limit — allows infinite restarts. Without this, systemd stops trying after N failures in a time window.
- `LimitNOFILE=1048576`: override the file descriptor limit for this process. Kubernetes needs many open FDs for pod sockets, cgroup files, etc. Default (1024) is too low.
- `LimitNPROC=infinity`: allow unlimited child processes.
- `OOMScoreAdjust=-999`: make the kubelet very unlikely to be OOM-killed by the kernel. Score range: -1000 (never kill) to +1000 (kill first). The kubelet and containerd both use negative scores so the kernel prefers to kill workloads before system daemons.
- `KillMode=process`: when stopping the unit, only send the signal to the main process (not the whole cgroup). Containerd uses `mixed` to also stop remaining processes.
- `EnvironmentFile=-/etc/sysconfig/kubelet`: load environment variables from a file. The leading `-` means "ignore if file doesn't exist." This is where you'd put `KUBELET_EXTRA_ARGS`.

**Overriding unit files safely:**

```bash
# Create a drop-in override without touching the original
systemctl edit kubelet
# Creates /etc/systemd/system/kubelet.service.d/override.conf
# Anything in the drop-in takes precedence

# Or create a full override:
cp /lib/systemd/system/kubelet.service /etc/systemd/system/kubelet.service
# Edit the copy; it takes precedence
# After any change:
systemctl daemon-reload
systemctl restart kubelet
```

---

## `journalctl` — reading the systemd journal

journalctl queries the structured binary journal that systemd maintains. Every unit's stdout/stderr is captured here automatically.

### Essential flags

```bash
# Last N lines of a specific unit
journalctl -u kubelet -n 100 --no-pager

# Follow live (like tail -f)
journalctl -u kubelet -f

# Since a specific time
journalctl -u kubelet --since "2026-06-05 10:00:00"
journalctl -u kubelet --since "30 minutes ago"

# Between time range
journalctl -u kubelet --since "2026-06-05 10:00:00" --until "2026-06-05 10:30:00"

# Filter by priority (levels: emerg=0, alert=1, crit=2, err=3, warning=4, notice=5, info=6, debug=7)
journalctl -u kubelet -p err           # errors and above
journalctl -u kubelet -p warning..err  # warnings and errors only

# JSON output (useful for jq parsing)
journalctl -u kubelet -n 50 -o json | jq '.MESSAGE'

# Multiple units at once
journalctl -u kubelet -u containerd --since "10 minutes ago"

# Kernel messages (dmesg equivalent)
journalctl -k --since "30 minutes ago"

# This boot only
journalctl -b -u kubelet

# Disk usage
journalctl --disk-usage

# Rotate / vacuum old logs
journalctl --vacuum-time=7d  # keep last 7 days
```

### Priority levels

| `-p` flag | syslog name | Typical usage |
|---|---|---|
| `-p 0` | `emerg` | System is unusable |
| `-p 1` | `alert` | Immediate action required |
| `-p 2` | `crit` | Critical condition |
| `-p 3` | `err` | Error conditions — use this for noise reduction |
| `-p 4` | `warning` | Warning conditions |
| `-p 5` | `notice` | Normal but significant |
| `-p 6` | `info` | Informational |
| `-p 7` | `debug` | Debug-level messages |

For kubelet debugging: start with `-p err`. If you need more context, drop to `-p warning` or remove the filter entirely.

---

## The kubelet as a systemd service — lifecycle

### Typical kubelet states

```bash
systemctl status kubelet
# ● kubelet.service - Kubernetes Kubelet
#    Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
#    Active: active (running) since 2026-06-05 09:00:00 UTC; 2h ago
#  Main PID: 1234 (kubelet)
#    CGroup: /system.slice/kubelet.service
#            └─1234 /usr/bin/kubelet ...
```

Fields to check:
- **`Active:`** — `active (running)` vs `activating (start)` vs `failed` vs `inactive (dead)`
- **`since ... ago`** — if it's `5s ago` and keeps changing, it's in a crash loop
- **`Main PID:`** — the actual process PID; use this for `strace`, `lsof`, etc.
- **`CGroup:`** — confirms kubelet is in `system.slice`, not `kubepods.slice`

### When kubelet is crash-looping

```bash
# See restart count and recent failures
systemctl status kubelet
# you'll see "start limit hit" or frequent restarts in the history

# View the systemd service state machine
systemctl show kubelet | grep -E 'ActiveState|SubState|Result|NRestarts'
# ActiveState=failed
# Result=exit-code
# NRestarts=47
```

If `StartLimitInterval` hasn't been set to `0` in the unit file, systemd will give up after some number of failures:

```bash
# Force a restart even after the start limit is hit
systemctl reset-failed kubelet
systemctl start kubelet
```

---

## The `kubepods.slice` cgroup hierarchy

systemd manages cgroups through the slice concept. For Kubernetes:

```
-.slice  (root)
├── system.slice
│   ├── kubelet.service           ← kubelet itself lives here
│   └── containerd.service        ← containerd lives here
├── kubepods.slice                ← all pods live here
│   ├── kubepods-guaranteed.slice
│   ├── kubepods-burstable.slice
│   │   └── kubepods-burstable-pod<uid>.slice
│   │       └── cri-containerd-<cid>.scope   ← individual container
│   └── kubepods-besteffort.slice
└── user.slice
```

The kubelet creates `kubepods.slice` and the QoS sub-slices at startup. Individual pod slices and container scopes are created per-pod by the kubelet (via systemd D-Bus when `--cgroup-driver=systemd`).

### Viewing the live cgroup tree

```bash
# Full cgroup tree
systemd-cgls

# Just the kubepods subtree
systemd-cgls /kubepods.slice

# With memory stats
systemd-cgtop  # like top, but for cgroups — shows CPU%, memory, IO per cgroup
```

`systemd-cgtop` is the fastest way to see which pod/slice is consuming the most resources at the cgroup level — faster than `kubectl top pod` when kubectl isn't available or the metrics server is down.

---

## The `--cgroup-driver` story in depth

Two players manage cgroups on a K8s node:
1. **systemd** — creates slices for its own services and the kubepods hierarchy
2. **kubelet + containerd** — create per-pod and per-container cgroups

The driver setting controls who creates the cgroup paths and how:

### `cgroupfs` driver (legacy, avoid on v2)

The kubelet directly creates directories under `/sys/fs/cgroup/` and writes control files. containerd does the same. Neither tells systemd. systemd and the runtime can create conflicting paths or not clean up after each other properly.

Risk: orphaned cgroup directories, resource accounting drift, systemd-managed services fighting with kubelet over the same cgroup paths.

### `systemd` driver (preferred)

The kubelet uses the systemd D-Bus API (`systemd.Unit.Start/Stop` etc.) to create transient units. containerd similarly uses systemd for scope creation. All cgroup creation goes through systemd's hierarchy manager. systemd owns the lifecycle: when a pod is deleted, systemd removes the scope and cleans up the cgroup.

Enabling on a node (both must match):

```bash
# Kubelet config (/etc/kubernetes/kubelet/kubelet-config.json or yaml):
{ "cgroupDriver": "systemd" }

# containerd config (/etc/containerd/config.toml):
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

# After changing either, reload and restart:
systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
```

### Detecting the current driver

```bash
# Kubelet:
ps aux | grep kubelet | grep cgroup-driver
# or:
cat /var/lib/kubelet/config.yaml | grep cgroupDriver
# or:
journalctl -u kubelet | grep "cgroup driver"

# containerd:
cat /etc/containerd/config.toml | grep SystemdCgroup
```

Mismatch (one says `systemd`, the other says `cgroupfs`) = the node will join the cluster but pods will fail to start with cgroup-related errors.

---

## Common kubelet debug flows

### Flow 1: Node is NotReady

```bash
# 1. Check kubelet service status
systemctl status kubelet

# 2. Last errors in kubelet logs
journalctl -u kubelet -p err -n 50

# 3. Is containerd healthy?
systemctl status containerd
journalctl -u containerd -p err -n 20

# 4. Can containerd pull images? (optional sanity check)
crictl pull nginx:alpine

# 5. Check node-level network
ip addr
ip route
# Can the node reach the API server?
curl -k https://<api-server>:443/healthz
```

### Flow 2: Kubelet crash-looping

```bash
# Find the error before each restart:
journalctl -u kubelet --since "1 hour ago" | grep -i "error\|fail\|panic"

# Common causes and their log signatures:
# - TLS cert expired:  "x509: certificate has expired"
# - Cgroup driver mismatch:  "failed to create ... cgroupfs" while driver is systemd
# - Missing CNI:  "cni plugin not initialized"
# - API server unreachable:  "Unable to connect to server"
# - Bad kubelet config:  "failed to load kubelet config"
```

### Flow 3: Pod stuck in Pending/ContainerCreating (node-side)

```bash
# Check containerd for runtime errors
journalctl -u containerd -n 100 --no-pager

# Check cgroup tree for the pod
systemd-cgls /kubepods.slice | grep <pod-uid>

# Check if the image was pulled
crictl images | grep <image-name>

# Check available disk space (image layer storage)
df -h /var/lib/containerd
```

### Flow 4: High memory on node, evictions

```bash
# See per-cgroup memory usage in real-time
systemd-cgtop -m  # -m for memory column

# Find which pod cgroup is largest
find /sys/fs/cgroup/kubepods.slice -name "memory.current" \
  -exec sh -c 'echo "$(cat $0): $1" "$0" "$(cat $1)"' {} {} \; 2>/dev/null \
  | sort -t: -k2 -rn | head -10
```

---

## The interview answer in 60 seconds

> "When a node is misbehaving and the kubelet is in a weird state, my first move is `systemctl status kubelet`. That single command tells me: is it running, is it crash-looping, how long has it been up, and the last few lines of output. If it's crash-looping I'll see a very recent start time.
>
> Next: `journalctl -u kubelet -p err -n 100 --no-pager` to get the last errors. I'm looking for the error just before each crash — it's usually one specific thing: a TLS cert issue, a cgroup driver mismatch between kubelet and containerd, a missing CNI plugin, or the API server being unreachable.
>
> If I see a cgroup driver mismatch, I check both configs: `cat /var/lib/kubelet/config.yaml | grep cgroupDriver` and `cat /etc/containerd/config.toml | grep SystemdCgroup`. Both must say `systemd` on modern nodes. Fix, then `systemctl daemon-reload && systemctl restart containerd && systemctl restart kubelet`.
>
> While the kubelet is running, `systemd-cgtop` gives me live cgroup resource usage — which pod slice is eating the most memory. `systemd-cgls` gives me the full cgroup tree so I can verify pod cgroups are being created correctly. If a pod never gets its cgroup slice, it won't start."

---

## Self-test drills

### 1. A node is misbehaving — kubelet is in a weird state. Walk me through your diagnosis at the systemd layer.

**Reference answer:**
- `systemctl status kubelet` — running? crash-looping? last start time?
- `journalctl -u kubelet -p err -n 100` — find the error before each crash
- Common causes: TLS cert expired, cgroup driver mismatch, missing CNI, API server unreachable
- Check containerd: `systemctl status containerd`, `journalctl -u containerd -p err`
- Cgroup mismatch: check kubelet config AND containerd config — both must match (`systemd` preferred)
- `systemctl reset-failed kubelet && systemctl start kubelet` if start limit was hit

### 2. What's the relationship between systemd slices and K8s pod cgroups?

**Reference answer:**
- systemd manages `kubepods.slice` at the top level, with QoS sub-slices: guaranteed/burstable/besteffort
- Each pod gets a transient slice: `kubepods-burstable-pod<uid>.slice`
- Each container gets a scope inside: `cri-containerd-<cid>.scope`
- With `--cgroup-driver=systemd`, kubelet calls systemd D-Bus to create these — systemd owns lifecycle, cleans up on delete
- With `cgroupfs` driver, kubelet creates paths directly — can conflict with systemd
- View with: `systemd-cgls /kubepods.slice`

### 3. What's `OOMScoreAdjust=-999` in the kubelet unit file doing?

**Reference answer:**
- The Linux OOM killer chooses what to kill based on `oom_score` (0–1000). Higher score = killed first.
- `OOMScoreAdjust` is added to the baseline score. `-999` makes the kubelet's score near-minimum.
- Effect: the kernel will exhaust all other processes before killing the kubelet — including workload containers
- containerd has a similar setting so both system daemons survive node memory pressure
- Without this, a memory spike from a rogue pod could OOM-kill the kubelet, making the node permanently NotReady until restarted

### 4. How do you view real-time resource usage per cgroup on a node?

**Reference answer:**
- `systemd-cgtop` — interactive top-like view sorted by CPU/memory/IO per cgroup. `-m` for memory focus.
- `systemd-cgls` — tree view of the cgroup hierarchy (static snapshot)
- For a specific pod: `cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/memory.current`
- `kubectl top pod` is the K8s abstraction — but requires metrics-server. `systemd-cgtop` works even when K8s API is unavailable.
- `find /sys/fs/cgroup/kubepods.slice -name memory.current` + sort to find the biggest consumer

---

## Further reading / watching

- **systemd man pages**: `man systemd.service`, `man systemd.unit`, `man journalctl` — comprehensive and well-written
- **systemd.io**: [systemd.io](https://systemd.io/) — official documentation site
- **Lennart Poettering's blog posts on systemd** — the design rationale behind slices and transient units
- **K8s node debugging**: [kubernetes.io/docs/tasks/debug/debug-cluster/](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

---

## The 4 dimensions (senior framing)

- **Tech**: unit types (service, slice, socket, timer); service anatomy (`After`, `Requires`, `Restart`, `LimitNOFILE`, `OOMScoreAdjust`); `journalctl -u -p err --since`; `kubepods.slice` hierarchy; `--cgroup-driver=systemd` for both kubelet and containerd; `systemd-cgls` + `systemd-cgtop` for live cgroup inspection.
- **People**: when a node goes NotReady, the first 10 minutes are "blame the app → blame K8s → finally check the node." Flip this order: `systemctl status kubelet` takes 2 seconds. Add this to your team's node debugging runbook explicitly — "Step 1: check kubelet service status." SREs who know systemd unblock incidents faster.
- **CI/CD**: in your node provisioning (Terraform, CloudFormation, userdata scripts), include assertions that verify: (1) `kubelet.service` is enabled, (2) `containerd.service` is enabled, (3) both `cgroupDriver` settings match. Catch this in CI before the node joins the cluster.
- **Operations**: `journalctl --vacuum-time=7d` should be in your node maintenance script — journals can fill `/var` on long-running nodes. Add `journalctl --disk-usage` to your node health check. Set up `systemctl --failed` to alert if any required service enters failed state — a containerd that stopped and wasn't restarted is a silent node degradation.
