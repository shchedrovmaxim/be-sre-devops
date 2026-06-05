# securityContext — the deep-dive

> **Goal**: by the end you can answer the killer question — **"Walk me through every securityContext setting you'd insist on for a stateless web service in production."** — naming each field, why it matters, and how Kyverno enforces it.

> Start with the [simple version](./security-context-simple.md) if you haven't read it. The contractor badge analogy is the spine.

---

## The senior framing — containers are not VMs

In a VM, a process running as root is root *inside the VM*. The hypervisor provides a hard boundary. In a container, a process running as root is root inside the container's namespace but shares the same kernel. A kernel exploit from within the container runs as root on the host node.

**Principle of least privilege at the container level** is about shrinking the damage surface of:
1. A compromised application (attacker execs into your container)
2. A supply chain attack (malicious code in your dependency)
3. A container escape exploit (kernel CVE used to escape to the host)

Every securityContext field reduces the blast radius of one or more of these.

---

## Pod-level securityContext

Pod-level settings apply to **all containers** in the pod (including init containers), unless overridden at the container level.

```yaml
spec:
  securityContext:
    runAsUser: 10001            # UID for all containers (unless overridden)
    runAsGroup: 10001           # GID for all containers
    fsGroup: 10001              # GID added to all volume mounts for ownership
    fsGroupChangePolicy: OnRootMismatch   # K8s 1.20+: faster volume permission fix
    supplementalGroups: [5000]  # additional GIDs added to the process
    sysctls:                    # kernel parameters (requires PSS baseline or higher)
    - name: net.ipv4.tcp_keepalive_time
      value: "300"
    seccompProfile:             # can also be set at pod level
      type: RuntimeDefault
```

### fsGroup — the volume ownership gotcha

When a Pod mounts a PersistentVolume, the files are owned by whatever UID/GID the storage created them with. If your container runs as UID 10001 but the files are owned by root, the container can't write to them.

`fsGroup` tells K8s to chown the volume's root directory to `<fsGroup>` before the container starts. The container then runs as UID 10001 with supplemental GID 10001 — and can read/write the volume.

```yaml
spec:
  securityContext:
    runAsUser: 10001
    fsGroup: 10001              # ensures mounted volumes are chowned to GID 10001
```

**The `fsGroupChangePolicy` gotcha**: by default, K8s recursively chowns the entire volume on every pod start. For large volumes (thousands of files), this adds minutes of startup time. `fsGroupChangePolicy: OnRootMismatch` only chowns if the root directory's GID doesn't match — much faster. Available since K8s 1.20.

---

## Container-level securityContext

Container-level fields override pod-level for that container.

```yaml
containers:
- name: api
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    privileged: false           # explicit false is good documentation; true = root on the node
    capabilities:
      drop: ["ALL"]
      add: ["NET_BIND_SERVICE"] # only if the process binds to port < 1024
    seccompProfile:
      type: RuntimeDefault
    procMount: Default          # Default = normal /proc masking; Unmasked = full proc (dangerous)
```

---

## Field-by-field breakdown

### runAsNonRoot

```yaml
runAsNonRoot: true
```

- Kubernetes refuses to start the container if the resolved UID (from image + securityContext) is 0.
- Does NOT set the UID — it only enforces that whatever UID is used is not root.
- Combine with `runAsUser: 10001` to both enforce and set a specific UID.

**Gotcha**: if the image's `USER` instruction sets a non-root UID, `runAsNonRoot: true` alone is enough. But if the image has no `USER` instruction, `runAsNonRoot: true` will reject the container at start because the default is root.

### readOnlyRootFilesystem

```yaml
readOnlyRootFilesystem: true
```

- Makes the container's root filesystem (`/`) read-only.
- Any write to `/usr`, `/bin`, `/tmp`, `/var/run`, etc. will fail with EROFS.
- **Attacker can't write a reverse shell binary to `/usr/bin`** or persist malware.
- Combined with emptyDir volumes for writable directories:

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: run
  mountPath: /var/run
- name: cache
  mountPath: /app/cache

volumes:
- name: tmp
  emptyDir: {}
- name: run
  emptyDir: {}
- name: cache
  emptyDir: {}
```

**Practical tip**: run your container locally with `docker run --read-only` to find all the paths it writes to before adding this to production. Some frameworks write PID files, socket files, or JIT caches that you need to account for.

### allowPrivilegeEscalation

```yaml
allowPrivilegeEscalation: false
```

- Blocks the `no_new_privs` flag at the kernel level. Prevents the process from calling `setuid()` or executing setuid binaries.
- With this set, even if an attacker places a setuid binary (e.g., a modified `sudo`) on the filesystem, it can't be used to escalate privileges.
- **Must be set explicitly** — the default is `true` (escalation is allowed).
- The `restricted` PSS level requires this to be `false`.

### privileged

```yaml
privileged: false   # (default) — always set explicitly as documentation
```

`privileged: true` gives the container **all Linux capabilities and full access to the host's device filesystem**. It's equivalent to running as root on the node with no container isolation. Never use it for application workloads. The only legitimate use is certain system-level DaemonSets (e.g., a CNI plugin that needs to configure host networking).

### capabilities

Linux capabilities break the root privilege into discrete abilities. Instead of "root can do everything," capabilities let you say "this process can bind to port 80, but can't load kernel modules."

The full list: `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_CHOWN`, `CAP_KILL`, `CAP_NET_RAW`, `CAP_MKNOD`, `CAP_AUDIT_WRITE`, `CAP_SETUID`, `CAP_SETGID`, `CAP_NET_BIND_SERVICE`, etc.

**The only correct default for applications**:

```yaml
capabilities:
  drop: ["ALL"]
  add: []       # add only what you need
```

**When to add capabilities**:

| Capability | When you legitimately need it |
|---|---|
| `NET_BIND_SERVICE` | Process binds to port < 1024 (e.g., NGINX on port 80) |
| `SYS_PTRACE` | Debugging tools (pprof, strace) — never in production |
| `NET_ADMIN` | CNI plugins, iptables manipulation — system DaemonSets only |
| `CHOWN` | If your app changes file ownership (rare for web services) |

**The `NET_RAW` gotcha**: the Docker default set includes `NET_RAW`, which allows raw socket access and can be used for certain network attacks (Ping of Death style). Drop it explicitly. `drop: [ALL]` handles this, but if you're selectively adding capabilities, don't add `NET_RAW`.

### seccompProfile

Seccomp (secure computing mode) filters the system calls a process can make. Without it, a container can call any kernel syscall — including ones with known exploit paths.

```yaml
seccompProfile:
  type: RuntimeDefault      # use the container runtime's default filter (recommended)
# or
  type: Localhost
  localhostProfile: profiles/custom.json   # custom seccomp profile from node filesystem
# or
  type: Unconfined          # no filter — dangerous, avoid
```

`RuntimeDefault` applies the default seccomp profile from containerd or CRI-O, which blocks ~300+ syscalls that are not commonly needed by application processes. It's conservative enough to not break legitimate workloads and strict enough to meaningfully reduce the kernel attack surface.

**Before K8s 1.19**: seccomp was set via a pod annotation (`seccomp.security.alpha.kubernetes.io/pod`). The annotation approach is deprecated — use the field.

### AppArmor (not securityContext, but adjacent)

AppArmor profiles are set via pod annotations, not securityContext:

```yaml
annotations:
  container.apparmor.security.beta.kubernetes.io/api: runtime/default
```

`runtime/default` uses the container runtime's AppArmor profile (same philosophy as RuntimeDefault for seccomp). AppArmor provides MAC (Mandatory Access Control) on top of seccomp's syscall filtering. The two are complementary but either alone provides significant protection.

---

## The complete production securityContext

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
spec:
  template:
    metadata:
      annotations:
        container.apparmor.security.beta.kubernetes.io/api: runtime/default
    spec:
      securityContext:
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      automountServiceAccountToken: false    # pod doesn't call K8s API
      containers:
      - name: api
        image: my-registry.example.com/payments-api:v1.2.3
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          privileged: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

---

## Kyverno enforcement — making it stick

Knowing the right settings is step one. Ensuring every team actually uses them is step two. This is where Kyverno ClusterPolicies (which you use in production) come in.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-security-context
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: check-containers
    match:
      any:
      - resources:
          kinds: ["Pod"]
          namespaceSelector:
            matchExpressions:
            - key: pod-security.kubernetes.io/enforce
              operator: In
              values: ["restricted"]
    validate:
      message: "All containers must have a restrictive securityContext."
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
              readOnlyRootFilesystem: true
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                  "?(ALL)": "*"
```

The `background: true` flag means Kyverno also audits existing pods (not just new ones). PolicyReports let you query the current violation state:

```bash
kubectl get policyreport -A
kubectl describe policyreport <name> -n <namespace>
```

---

## The interview answer in 60 seconds

> "For a stateless web service, I'd insist on six settings. `runAsNonRoot: true` with a specific `runAsUser: 10001` — process is not root, so a container escape doesn't give the attacker root on the node. `readOnlyRootFilesystem: true` — attacker can't write malware or persistence mechanisms; combined with emptyDir volumes for `/tmp` and any writable directories the app needs. `allowPrivilegeEscalation: false` — blocks setuid escalation. `capabilities: { drop: [ALL] }` — strips all Linux capabilities; add back only `NET_BIND_SERVICE` if the process listens on a port below 1024. `seccompProfile: { type: RuntimeDefault }` — syscall filter from the container runtime, reduces the kernel attack surface without tuning a custom profile. At the pod level I'd also set `automountServiceAccountToken: false` if it doesn't call the K8s API.
>
> I'd enforce all of this via a Kyverno ClusterPolicy with `validationFailureAction: Enforce` in production namespaces, so it's not optional — if a team's Deployment doesn't have these settings, the pod is rejected at admission."

---

## Self-test drills

### 1. Walk me through every securityContext setting you'd insist on for a stateless web service.

**Reference answer**: `runAsNonRoot: true`, `runAsUser: 10001`, `readOnlyRootFilesystem: true` (+ emptyDir for writable paths), `allowPrivilegeEscalation: false`, `capabilities: { drop: [ALL] }`, `seccompProfile: { type: RuntimeDefault }`, `automountServiceAccountToken: false`. Enforce via Kyverno. See full YAML above.

### 2. What's the difference between `runAsNonRoot` and `runAsUser`?

**Reference answer**: `runAsNonRoot: true` is a constraint — "reject the pod if it would run as root." It doesn't set the UID. `runAsUser: N` sets the UID explicitly. Use both together: `runAsUser` sets it; `runAsNonRoot` is the safety net if someone removes `runAsUser`.

### 3. What problem does `readOnlyRootFilesystem: true` solve, and how do you handle apps that need to write files?

**Reference answer**: prevents an attacker from writing binaries, scripts, or persistence mechanisms to the container's filesystem. Handle writable paths with emptyDir volumes mounted at exactly the directories the app needs (`/tmp`, `/app/cache`, `/var/run`). emptyDir lives on the node but is ephemeral — destroyed when the pod is removed. Run the container locally with `docker run --read-only` first to discover all writes.

### 4. What does `capabilities: { drop: ["ALL"] }` actually remove, and when would you add one back?

**Reference answer**: drops all ~40 Linux capabilities the container would inherit from the runtime's default set. This includes `NET_RAW` (raw sockets), `SYS_CHROOT`, `AUDIT_WRITE`, `SETUID`, `SETGID`, etc. Add back: `NET_BIND_SERVICE` if the process must bind to port < 1024 (e.g., NGINX on port 80). Never add back `SYS_ADMIN` (nearly unrestricted), `NET_ADMIN` (iptables, interface config), or `SYS_PTRACE` (process tracing) for application workloads.

---

## Further reading

- [securityContext API reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context)
- [Linux capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Seccomp in Kubernetes](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Kyverno policies library](https://kyverno.io/policies/)

---

## The 4 dimensions (senior framing)

- **Tech**: six container-level settings (runAsNonRoot, runAsUser, readOnlyRootFilesystem, allowPrivilegeEscalation, capabilities.drop ALL, seccompProfile RuntimeDefault) + pod-level (runAsUser, fsGroup, fsGroupChangePolicy). emptyDir pattern for writable directories. AppArmor as complementary MAC. Kyverno enforcement for cluster-wide adoption.
- **People**: security hardening meets friction. Teams don't add securityContext because their app "worked without it." Make secure the default in your Helm chart scaffolding — ship a baseline securityContext in every chart template. Run warn mode first so teams see what they're missing without being blocked. Pair-review new workloads for securityContext completeness before they go to prod.
- **CI/CD**: Kyverno Audit mode generates PolicyReports — visualize them in a dashboard, track violation trends. Gate staging and prod deployments on Kyverno compliance (use the Kyverno GitHub Actions or Argo CD sync wave with Kyverno check). Use `docker run --read-only` in your local dev Makefile target so engineers discover missing writable paths before CI. Wiz scanning detects what's actually running in prod and flags containers still violating these settings.
- **Operations**: alert on containers running as root in production (Falco has a built-in rule for this: `spawned_process_using_docker` + `container.privileged`). Wiz can surface this as a detective control — complement your Kyverno preventive controls. Quarterly: run `kubectl get pods -A -o json | jq` to audit actual securityContexts in running pods; Kyverno's background scan PolicyReports make this easy. Flag any newly privileged containers as a security event.
