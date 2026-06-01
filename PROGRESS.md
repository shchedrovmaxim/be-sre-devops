# PROGRESS — Topic-by-topic checklist

> Updated at the end of each session. Tick `[x]` when fully understood (can teach it back out loud).
> `[~]` = partial coverage, needs more depth. `[ ]` = not yet started.

## 1. Linux fundamentals
- [ ] cgroups v1 vs v2 — controller hierarchy, unified vs split
- [x] cgroups memory — `memory.max`, `memory.high`, `memory.low`, OOMKilled mechanics
- [ ] cgroups CPU — `cpu.weight`, `cpu.max`, CFS throttling, request vs limit mapping
- [ ] cgroups IO — `io.max`, EBS throttling implications
- [ ] Namespaces — pid, net, mnt, uts, ipc, user, cgroup
- [ ] How `runc` builds a container (namespaces + cgroups + capabilities + chroot)
- [ ] iptables / nftables basics
- [ ] conntrack (UDP DNS implications; intermittent timeouts)
- [ ] systemd basics (services, units, journalctl)
- [ ] File descriptors, ulimits, inodes
- [ ] Process accounting and `/proc` exploration

## 2. Kubernetes
### Scheduling
- [ ] Taints + tolerations — effects: NoSchedule, PreferNoSchedule, NoExecute
- [ ] Node affinity — required vs preferred
- [ ] Pod affinity — when (co-location for cache locality)
- [ ] Pod anti-affinity — when (HA spread)
- [ ] `topologySpreadConstraints` — when over anti-affinity
- [ ] priorityClass + preemption
- [ ] The mental model: "can pod run here (taints) → does pod want here (affinity) → spread evenly (topology)"

### Resource management
- [ ] requests vs limits — what each controls
- [ ] QoS classes: Guaranteed / Burstable / BestEffort
- [ ] Eviction order under node pressure
- [ ] `kube-reserved`, `system-reserved`, `eviction-hard` kubelet flags
- [ ] CPU throttling vs starvation
- [ ] Memory limits vs cgroup `memory.max` mapping

### Pod lifecycle
- [ ] Pod phases: Pending → Running → Succeeded/Failed
- [ ] Init containers (sequential, before app)
- [ ] Native sidecars (K8s 1.29+, `restartPolicy: Always` on init container)
- [ ] Probes: liveness, readiness, startup
- [ ] `terminationGracePeriodSeconds` + SIGTERM handling
- [ ] PodDisruptionBudget

### Workload controllers
- [ ] Deployment, ReplicaSet — rolling update mechanics
- [ ] StatefulSet — ordered start/stop, stable network IDs, persistent storage
- [ ] DaemonSet — node-level workloads
- [ ] Job, CronJob — completion semantics, backoff
- [ ] HorizontalPodAutoscaler — metric server, custom metrics

### Networking & services
- [ ] Services: ClusterIP, NodePort, LoadBalancer, ExternalName, Headless
- [ ] Endpoints vs EndpointSlices
- [ ] kube-proxy modes: iptables, IPVS, eBPF
- [ ] CNI choices and what they own
- [ ] NetworkPolicy — ingress + egress rules
- [ ] Ingress vs Gateway API
- [ ] CoreDNS + NodeLocal DNS Cache

### Storage
- [ ] CSI — what it is, how drivers work
- [ ] StorageClass + dynamic provisioning
- [ ] PVC / PV lifecycle, reclaim policies
- [ ] Volume expansion
- [ ] Snapshot CRDs

### Security
- [ ] RBAC: Role / ClusterRole / RoleBinding / ClusterRoleBinding
- [ ] ServiceAccounts + token projection
- [ ] PodSecurityStandards (Privileged / Baseline / Restricted)
- [ ] securityContext — runAsNonRoot, readOnlyRootFilesystem, capabilities
- [ ] Image pull secrets + private registries

### Cluster ops
- [ ] etcd basics — performance, backup/restore
- [ ] Control plane components — kube-apiserver, scheduler, controller-manager
- [ ] Upgrade flow: control plane → nodes
- [ ] Kubeadm vs managed (EKS/GKE/AKS)

## 3. CI/CD + DevOps ecosystem
- [ ] Multi-stage Docker builds + BuildKit
- [ ] BuildKit cache mounts (`--mount=type=cache`)
- [ ] Distroless / scratch base images
- [ ] Image scanning: Trivy, Grype
- [ ] SBOM: Syft
- [ ] Image signing: Cosign / Sigstore (keyless via OIDC)
- [ ] Admission policy enforcement: Kyverno / OPA Gatekeeper
- [ ] ArgoCD: Application, AppProject, ApplicationSet
- [ ] ArgoCD: sync waves, sync hooks (PreSync, Sync, PostSync, SyncFail)
- [ ] ArgoCD: `ignoreDifferences`, Server-Side Apply
- [ ] ArgoCD: app-of-apps pattern
- [ ] ArgoCD Image Updater
- [ ] Renovate (when over Dependabot)
- [ ] Dependabot
- [ ] GitHub Actions runners (self-hosted patterns)
- [ ] Trunk-based development + immutable promotion across envs
- [ ] Conftest / OPA in CI

## 4. Certificates
- [ ] Let's Encrypt + ACME protocol
- [ ] HTTP-01 vs DNS-01 vs TLS-ALPN-01 challenges
- [ ] When DNS-01 is required (wildcard certs)
- [ ] cert-manager: Issuer / ClusterIssuer / Certificate
- [ ] Rate limits and staging environment
- [ ] Vault as internal CA
- [ ] Cert rotation patterns + monitoring expiry

## 5. Secrets management
- [ ] Sealed Secrets — encrypted in Git, decrypted by controller
- [ ] External Secrets Operator (ESO) — SecretStore, ExternalSecret CRDs
- [ ] SOPS + age/KMS — file-level encryption
- [ ] Vault as source of truth + ESO sync pattern
- [ ] When to pick each

## 6. Architecture (senior mindset) — THE rejection gap
- [ ] KISS, YAGNI, justified complexity
- [ ] Ousterhout "A Philosophy of Software Design" (chapters 1-3)
- [ ] ADRs (Architecture Decision Records) — Context / Decision / Consequences
- [ ] Runbooks as deliverables, not afterthoughts
- [ ] Onboarding new engineers in <1 week as design constraint
- [ ] Blameless postmortems (Google SRE book pattern)
- [ ] The 5 whys for root cause
- [ ] CI as architecture (immutable promotion, trunk-based)
- [ ] **The 4 dimensions framework**: Tech + People + CI/CD + Operations
- [ ] Practice: 3 design questions answered with all 4 dimensions

## 7. Observability
- [ ] Prometheus: scrape config, targets, relabeling
- [ ] Recording rules vs alerting rules
- [ ] PromQL: rate, histogram_quantile, topk, sum by, irate
- [ ] Cardinality discipline
- [ ] Long-term storage: Thanos / Mimir / VictoriaMetrics
- [ ] OpenTelemetry: collector architecture (receivers/processors/exporters)
- [ ] Semantic conventions
- [ ] Sampling: head-based vs tail-based
- [ ] Grafana dashboards as code
- [ ] Loki for logs (label cardinality discipline)
- [ ] USE / RED methods
- [ ] SLI / SLO / SLA design + error budgets
- [ ] Alert design: avoiding fatigue, actionable signals

## 8. AWS / EKS
- [x] Strong on most AWS components (per interview feedback)
- [ ] EKS upgrades: control plane + nodegroups, in-place vs blue-green
- [ ] Karpenter vs Cluster Autoscaler — pros/cons
- [ ] IRSA — how the token exchange works end-to-end
- [ ] AWS VPC CNI: prefix delegation, security groups for pods
- [ ] AWS Load Balancer Controller: NLB vs ALB, target types
- [ ] ExternalDNS for Route 53
- [ ] CloudFront + Shield + WAF in front of EKS
- [ ] Secrets Manager vs Parameter Store
- [ ] CloudWatch vs Prometheus/OTel — cost trade-offs

## 9. Terraform
- [ ] Remote state: S3 backend + DynamoDB locking
- [ ] Workspaces vs separate state per env (and why workspaces are usually wrong)
- [ ] Module discipline: when to write vs use registry
- [ ] `terraform_data`, `lifecycle` blocks, `moved` blocks
- [ ] Drift detection: scheduled `plan` + alerting
- [ ] Atlantis / Spacelift / Terragrunt — PR-based workflows
- [ ] Testing: terratest, native `terraform test` (1.6+)
- [ ] Multi-account AWS patterns

## 10. Networking (cross-cutting)
- [ ] CNI internals: Cilium, Calico, AWS VPC CNI
- [ ] NetworkPolicy: ingress + egress
- [ ] kube-proxy modes
- [ ] CoreDNS Corefile + plugins
- [ ] NodeLocal DNS Cache
- [ ] conntrack table limits + UDP DNS implications
- [ ] Gateway API
- [ ] eBPF basics + Cilium replacing kube-proxy

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
