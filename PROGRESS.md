# PROGRESS — Topic-by-topic checklist

> Updated at the end of each session. Tick `[x]` when fully understood (can teach it back out loud).
> `[~]` = partial coverage, needs more depth. `[ ]` = not yet started.

## 1. Linux fundamentals
- [x] cgroups v1 vs v2 — controller hierarchy, unified vs split
- [x] cgroups memory — `memory.max`, `memory.high`, `memory.low`, OOMKilled mechanics
- [x] cgroups CPU — `cpu.weight`, `cpu.max`, CFS throttling, request vs limit mapping
- [x] cgroups IO — `io.max`, EBS throttling implications
- [x] Namespaces — pid, net, mnt, uts, ipc, user, cgroup
- [x] How `runc` builds a container (namespaces + cgroups + capabilities + chroot)
- [x] iptables / nftables basics
- [x] conntrack (UDP DNS implications; intermittent timeouts)
- [x] systemd basics (services, units, journalctl)
- [x] File descriptors, ulimits, inodes
- [~] Process accounting and `/proc` exploration — touched in namespaces + ulimits docs

## 2. Kubernetes
### Scheduling
- [x] Taints + tolerations — effects: NoSchedule, PreferNoSchedule, NoExecute
- [x] Node affinity — required vs preferred
- [x] Pod affinity — when (co-location for cache locality)
- [x] Pod anti-affinity — when (HA spread)
- [x] `topologySpreadConstraints` — when over anti-affinity
- [x] priorityClass + preemption
- [x] The mental model: "can pod run here (taints) → does pod want here (affinity) → spread evenly (topology)"

### Resource management
- [x] requests vs limits — what each controls
- [x] QoS classes: Guaranteed / Burstable / BestEffort
- [x] Eviction order under node pressure
- [x] `kube-reserved`, `system-reserved`, `eviction-hard` kubelet flags
- [x] CPU throttling vs starvation
- [x] Memory limits vs cgroup `memory.max` mapping

### Pod lifecycle
- [x] Pod phases: Pending → Running → Succeeded/Failed
- [x] Init containers (sequential, before app)
- [x] Native sidecars (K8s 1.29+, `restartPolicy: Always` on init container)
- [x] Probes: liveness, readiness, startup
- [x] `terminationGracePeriodSeconds` + SIGTERM handling
- [x] PodDisruptionBudget

### Workload controllers
- [x] Deployment, ReplicaSet — rolling update mechanics
- [x] StatefulSet — ordered start/stop, stable network IDs, persistent storage
- [x] DaemonSet — node-level workloads
- [x] Job, CronJob — completion semantics, backoff
- [x] HorizontalPodAutoscaler — metric server, custom metrics

### Networking & services
- [x] Services: ClusterIP, NodePort, LoadBalancer, ExternalName, Headless
- [x] Endpoints vs EndpointSlices
- [x] kube-proxy modes: iptables, IPVS, eBPF
- [x] CNI choices and what they own
- [x] NetworkPolicy — ingress + egress rules
- [x] Ingress vs Gateway API
- [x] CoreDNS + NodeLocal DNS Cache

### Storage
- [x] CSI — what it is, how drivers work
- [x] StorageClass + dynamic provisioning
- [x] PVC / PV lifecycle, reclaim policies
- [x] Volume expansion
- [x] Snapshot CRDs

### Security
- [x] RBAC: Role / ClusterRole / RoleBinding / ClusterRoleBinding
- [x] ServiceAccounts + token projection
- [x] PodSecurityStandards (Privileged / Baseline / Restricted)
- [x] securityContext — runAsNonRoot, readOnlyRootFilesystem, capabilities
- [x] Image pull secrets + private registries

### Cluster ops
- [x] etcd basics — performance, backup/restore
- [x] Control plane components — kube-apiserver, scheduler, controller-manager
- [x] Upgrade flow: control plane → nodes
- [x] Kubeadm vs managed (EKS/GKE/AKS)

## 3. CI/CD + DevOps ecosystem
- [x] Multi-stage Docker builds + BuildKit
- [x] BuildKit cache mounts (`--mount=type=cache`)
- [x] Distroless / scratch base images
- [x] Image scanning: Trivy, Grype
- [x] SBOM: Syft
- [x] Image signing: Cosign / Sigstore (keyless via OIDC)
- [x] Admission policy enforcement: Kyverno / OPA Gatekeeper
- [x] ArgoCD: Application, AppProject, ApplicationSet
- [x] ArgoCD: sync waves, sync hooks (PreSync, Sync, PostSync, SyncFail)
- [x] ArgoCD: `ignoreDifferences`, Server-Side Apply
- [x] ArgoCD: app-of-apps pattern
- [x] ArgoCD Image Updater
- [x] Renovate (when over Dependabot)
- [x] Dependabot
- [x] GitHub Actions runners (self-hosted patterns)
- [x] Trunk-based development + immutable promotion across envs
- [x] Conftest / OPA in CI

## 4. Certificates
- [x] Let's Encrypt + ACME protocol
- [x] HTTP-01 vs DNS-01 vs TLS-ALPN-01 challenges
- [x] When DNS-01 is required (wildcard certs)
- [x] cert-manager: Issuer / ClusterIssuer / Certificate
- [x] Rate limits and staging environment
- [x] Vault as internal CA
- [x] Cert rotation patterns + monitoring expiry

## 5. Secrets management
- [x] Sealed Secrets — encrypted in Git, decrypted by controller
- [x] External Secrets Operator (ESO) — SecretStore, ExternalSecret CRDs
- [x] SOPS + age/KMS — file-level encryption
- [x] Vault as source of truth + ESO sync pattern
- [x] When to pick each

## 6. Architecture (senior mindset) — THE rejection gap
- [x] KISS, YAGNI, justified complexity
- [~] Ousterhout "A Philosophy of Software Design" (chapters 1-3) — concepts integrated; reading the book itself is parked
- [x] ADRs (Architecture Decision Records) — Context / Decision / Consequences
- [x] Runbooks as deliverables, not afterthoughts
- [x] Onboarding new engineers in <1 week as design constraint
- [x] Blameless postmortems (Google SRE book pattern)
- [x] The 5 whys for root cause
- [x] CI as architecture (immutable promotion, trunk-based)
- [x] **The 4 dimensions framework**: Tech + People + CI/CD + Operations
- [x] Practice: 3 design questions answered with all 4 dimensions

## 7. Observability
- [x] Prometheus: scrape config, targets, relabeling
- [x] Recording rules vs alerting rules
- [x] PromQL: rate, histogram_quantile, topk, sum by, irate
- [x] Cardinality discipline
- [x] Long-term storage: Thanos / Mimir / VictoriaMetrics
- [x] OpenTelemetry: collector architecture (receivers/processors/exporters)
- [x] Semantic conventions
- [x] Sampling: head-based vs tail-based
- [x] Grafana dashboards as code
- [x] Loki for logs (label cardinality discipline)
- [~] USE / RED methods — mentioned in slos.md context; not deep-dived
- [x] SLI / SLO / SLA design + error budgets
- [x] Alert design: avoiding fatigue, actionable signals — covered via multi-window multi-burn-rate

## 8. AWS / EKS
- [x] Strong on most AWS components (per interview feedback)
- [x] EKS upgrades: control plane + nodegroups, in-place vs blue-green
- [x] Karpenter vs Cluster Autoscaler — pros/cons
- [x] IRSA — how the token exchange works end-to-end
- [x] AWS VPC CNI: prefix delegation, security groups for pods
- [x] AWS Load Balancer Controller: NLB vs ALB, target types
- [x] ExternalDNS for Route 53
- [x] CloudFront + Shield + WAF in front of EKS
- [x] Secrets Manager vs Parameter Store
- [x] CloudWatch vs Prometheus/OTel — cost trade-offs

## 9. Terraform
- [x] Remote state: S3 backend + DynamoDB locking
- [x] Workspaces vs separate state per env (and why workspaces are usually wrong)
- [x] Module discipline: when to write vs use registry
- [~] `terraform_data`, `lifecycle` blocks, `moved` blocks — partially in modules + drift-detection
- [x] Drift detection: scheduled `plan` + alerting
- [x] Atlantis / Spacelift / Terragrunt — PR-based workflows
- [~] Testing: terratest, native `terraform test` (1.6+) — mentioned in modules, not deep
- [~] Multi-account AWS patterns — touched in pr-workflows, not deep

## 10. Networking (cross-cutting)
- [x] CNI internals: Cilium, Calico, AWS VPC CNI
- [x] NetworkPolicy: ingress + egress
- [x] kube-proxy modes
- [x] CoreDNS Corefile + plugins
- [x] NodeLocal DNS Cache
- [x] conntrack table limits + UDP DNS implications
- [ ] Gateway API
- [~] eBPF basics + Cilium replacing kube-proxy — covered in cni-comparison + kube-proxy-modes

## 11. Istio (mostly done) ✅
- [x] Architecture: istiod + Envoy, xDS
- [x] CRDs: Gateway, VS, DR, ServiceEntry, Sidecar, PeerAuth, ReqAuth, AuthZ
- [x] Resilience: timeouts, retries (safe retryOn), circuit breaking
- [x] mTLS: STRICT / PERMISSIVE / DISABLE, migration path
- [x] AuthorizationPolicy semantics (CUSTOM → DENY → ALLOW; default-deny)
- [x] JWT validation (RequestAuth + AuthZ)
- [x] TLS origination (ServiceEntry + DR with SNI)
- [x] Observability + Telemetry CR (cardinality control)
- [x] Upgrades: revision tags, canary CP
- [x] kube-proxy + CoreDNS + Istio coexistence
- [x] AWS NLB / ALB integration
- [x] Troubleshooting playbook (UH, UF, UC, NR debug flow)
- [x] 26 Q&A drills
- [x] External service YAML patterns (ServiceEntry + VS + DR)

## 12. Gravitee (mostly done) ✅
- [x] Category framing (APIM vs service mesh)
- [x] 5 core concepts: API, Plan, Application, Subscription, Policy
- [x] Policy chain (request → backend → response)
- [x] APIM vs AM vs Cockpit
- [x] Backing stores: MongoDB, Elasticsearch, Redis
- [x] Common policies: api-key, jwt, oauth2, rate-limit, transform
- [x] Comparison to Kong / Apigee
- [x] Honest "haven't used it" answer

## 13. Interview prep
- [x] STAR behavioral stories — 5 templates + "why leaving solo founding"
- [x] Troubleshooting drills — 5 end-to-end scenarios with TL;DR
- [x] Sharp questions to ask them — per interview stage
- [ ] Architecture design drills — 3 scenarios with reference answers
- [ ] Mock interview round 1 (technical + architecture + behavioral)
- [ ] Mock interview round 2
- [ ] Final review of weak areas
