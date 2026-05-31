# AWS / EKS

Most senior SRE roles run on EKS. Strong AWS knowledge was confirmed in the rejection feedback — this area is mostly broadening, not closing gaps.

## Status: ✅ Strong overall; deepen EKS specifics

## Recommended order

### EKS internals
1. **EKS upgrade flow** — control plane → managed nodegroups, in-place vs blue-green
2. **Karpenter vs Cluster Autoscaler** — pros/cons, when each
3. **IRSA** — IAM Roles for Service Accounts; how the token exchange works end-to-end
4. **AWS VPC CNI** — prefix delegation for IP density, security groups for pods
5. **AWS Load Balancer Controller** — NLB vs ALB target types (ip vs instance)
6. **ExternalDNS** — Route 53 record management from K8s annotations

### Edge + security
7. **CloudFront + Shield Advanced + WAF** in front of EKS
8. **Secrets Manager** vs **Parameter Store** — when each
9. **VPC endpoints** for S3/ECR/DynamoDB (cost lever via NAT bytes)

### Observability + cost
10. **CloudWatch** vs **Prometheus/OTel** — cost trade-offs
11. **Cost Explorer** + tagging strategy
12. **OpenCost / Kubecost** for K8s-level attribution

## Files

(none yet — to be written)

## Why this matters

Strong AWS was confirmed in the rejection feedback. This area is about **deepening the EKS-specific patterns** to round out a senior profile, not closing critical gaps.

## Hands-on environment

- Free-tier EKS cluster (or local kind for non-cloud-specific exercises)
- `eksctl` for cluster lifecycle
- Karpenter Helm install for autoscaling experiments
