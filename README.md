# AWS Secure Delivery Platform

A reproducible, security-focused delivery platform: a small Python service built, scanned, signed and deployed through GitOps to a local Kubernetes lab and to short-lived AWS environments provisioned with Terraform.

## Status

**Milestone 0: architecture and planning. No infrastructure is deployed yet.**

Everything below describes the target design. Each capability is marked implemented only when linked evidence exists.

| Area | Status |
|---|---|
| Repository, branch protection, docs CI | In progress (M0) |
| Linux lab and automation (Ansible) | Planned (M1) |
| Containerized service on K3s with Helm | Planned (M2) |
| CI security gates (SAST, SCA, DAST, IaC, container, secrets) | Planned (M3) |
| GitOps delivery with Argo CD | Planned (M4) |
| AWS foundation with Terraform (VPC, IAM, KMS, OIDC) | Planned (M5) |
| Ephemeral EKS provision, deploy, verify, teardown | Planned (M6) |
| Kubernetes and AWS security hardening | Planned (M7) |
| Observability and incident response | Planned (M8) |
| Reliability and recovery testing | Planned (M9) |

## License

MIT. See [LICENSE](LICENSE).
