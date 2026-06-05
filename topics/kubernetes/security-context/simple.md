# securityContext — the simple version (the contractor badge analogy)

> Read this first. Once the analogy clicks, the [deep-dive doc](./deep-dive.md) becomes easy.

This doc only explains **one idea**:

> **securityContext is the badge + keycard you give a contractor. It defines what they're allowed to touch. Less is always more.**

---

## The contractor arrives at your office

A contractor shows up to fix your servers. What do you give them?

| Office world | K8s securityContext |
|---|---|
| Visitor badge, not an employee keycard | `runAsNonRoot: true` — not running as the all-powerful root |
| Access only to the server room, not HR files | `readOnlyRootFilesystem: true` — can't write anywhere unexpected |
| Can't call a locksmith (no tool escalation) | `allowPrivilegeEscalation: false` — can't gain more privileges |
| Can plug in ethernet cables but not cut power | Specific Linux capabilities only: drop ALL, add what's needed |
| Logged by building security | `seccompProfile: RuntimeDefault` — syscall filter |

You wouldn't hand a contractor the CEO's badge and say "figure it out." Same logic: don't give a container root + all capabilities and hope for the best.

---

## Pod level vs container level

```yaml
spec:
  securityContext:           # pod-level — applies to all containers
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000            # volume ownership

  containers:
  - name: app
    securityContext:         # container-level — overrides pod-level for this container
      runAsNonRoot: true
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: RuntimeDefault
```

Container-level overrides pod-level when both are set on the same field.

---

## The minimal production securityContext for a stateless web service

```yaml
containers:
- name: api
  image: my-api:1.0
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001           # non-zero, non-root UID
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    seccompProfile:
      type: RuntimeDefault
  volumeMounts:
  - name: tmp
    mountPath: /tmp            # writable scratch space
  - name: cache
    mountPath: /app/cache      # if the app needs a writable cache dir

volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir: {}
```

The `readOnlyRootFilesystem: true` + emptyDir pattern is the key: the root filesystem is immutable (no attacker can write malware into `/usr/bin`), but the app gets writable directories exactly where it needs them.

---

## The intuition cheat sheet

| Setting | What it does | Default (if not set) |
|---|---|---|
| `runAsNonRoot: true` | Refuses to start if image runs as UID 0 | Not enforced |
| `runAsUser: N` | Forces container to run as UID N | Whatever the image specifies |
| `readOnlyRootFilesystem: true` | Makes `/` read-only | Writable (dangerous) |
| `allowPrivilegeEscalation: false` | Blocks `setuid` binaries and `sudo`-style escalation | Allowed |
| `capabilities.drop: ["ALL"]` | Removes all Linux capabilities | Inherits a default set |
| `seccompProfile.type: RuntimeDefault` | Applies the container runtime's default syscall filter | Unconfined (all syscalls allowed) |
| `fsGroup: N` | Sets GID ownership on mounted volumes | No GID set |

---

## Self-test (one question — the killer one)

Out loud:

> **"Walk me through every securityContext setting you'd insist on for a stateless web service in production."**

**Reference answer (intuitive version):**

"`runAsNonRoot: true` and `runAsUser: 10001` — the process runs as a non-root UID, so even if the container is compromised, it's not root on the node. `readOnlyRootFilesystem: true` — an attacker can't write binaries or scripts anywhere persistent; combined with emptyDir volumes for writable scratch space. `allowPrivilegeEscalation: false` — blocks setuid binaries and sudo. `capabilities: { drop: [ALL] }` — strips all Linux capabilities so the process can't do anything privileged (raw sockets, mounting filesystems, killing other processes). `seccompProfile: { type: RuntimeDefault }` — adds a syscall filter so the process can't call unexpected kernel APIs. Together these are the restricted PSS level. I'd enforce them with a Kyverno ClusterPolicy so no deployment can skip them."

---

## Further reading

- [securityContext docs](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Linux capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html)

---

## Next: the deep-dive

When the contractor analogy feels obvious, jump to [`security-context.md`](./deep-dive.md). The deep-dive covers:

- Every field with exact behavioral detail
- Pod-level vs container-level precedence rules
- The `fsGroup` + volume ownership story
- Seccomp and AppArmor
- Capabilities: what `NET_BIND_SERVICE` and `SYS_PTRACE` actually do
- Kyverno enforcement
- 4 self-test drills + 4-dimensions framing
