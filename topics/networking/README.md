# Networking (cross-cutting)

K8s networking is where many incidents start. CNI, NetworkPolicy, kube-proxy, DNS — each is a separate world.

## Recommended order

### CNI
1. **CNI plugin model** — what the CNI ABI actually does at pod creation
2. **AWS VPC CNI** — pods get VPC IPs; security implications
3. **Cilium** — eBPF-based, can replace kube-proxy
4. **Calico** — BGP option for on-prem

### Policy
5. **NetworkPolicy** — ingress + egress rules
6. Default-deny patterns
7. Cilium NetworkPolicy extensions (L7)

### Service routing
8. **kube-proxy modes**: iptables, IPVS, eBPF
9. Service → endpoint translation flow
10. Headless services and StatefulSet networking
11. **Gateway API** — replacement for Ingress

### DNS
12. **CoreDNS Corefile** + plugin model
13. **NodeLocal DNS Cache** — TCP path, bypass conntrack
14. **ndots** — query amplification
15. **conntrack** — UDP DNS implications (the classic intermittent-failure cause)

### eBPF
16. eBPF basics — what it actually is at the kernel level
17. How Cilium uses eBPF to replace kube-proxy + apply NetworkPolicy
18. Observability via eBPF (Hubble, Pixie, Parca)

## Files

(none yet — to be written)

## Why this matters

The intermittent-DNS scenario (already in [troubleshooting drills](../../interview-prep/troubleshooting-drills.md)) is a classic senior question. So is "explain how a packet flows from your app pod to another pod."

## Hands-on environment

- Local `kind` cluster (default CNI: kindnet)
- Optional: `kind` with Cilium for eBPF experiments
- `nsenter`, `ip netns`, `tcpdump` to poke at namespaces
- `dig`, `nslookup` from inside pods
