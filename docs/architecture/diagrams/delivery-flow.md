# Delivery Flow

How a change moves from a pull request to a running environment. Labels `TB-n` mark trust boundaries, defined in [trust-boundaries.md](../trust-boundaries.md).

```mermaid
flowchart LR
  dev["Engineer on ops-01"] -->|"pull request (TB-1)"| gh["GitHub repository"]
  gh --> pr["PR checks: tests, lint, SAST, SCA, secrets, IaC, container scan, DAST"]
  pr -->|"all required checks pass"| main["Protected main branch"]
  main --> rel["Release workflow: build, scan, SBOM, sign"]
  rel --> ghcr["GHCR (local images)"]
  rel -->|"OIDC to STS (TB-2)"| ecr["Amazon ECR (AWS images)"]
  rel -->|"reviewed PR pins image digest"| desired["GitOps desired state in Git"]
  desired -->|"TB-4"| argoL["Argo CD on K3s"]
  desired -->|"TB-4"| argoA["Argo CD on EKS"]
  ghcr --> argoL
  ecr --> argoA
  main -->|"protected workflow, OIDC (TB-2, TB-3)"| tf["Terraform plan and apply"]
  tf --> aws["AWS: VPC, IAM, KMS, ECR, EKS"]
```

Ownership: GitHub Actions builds and publishes; Terraform owns AWS infrastructure and the initial cluster bootstrap; Argo CD owns everything deployed into Kubernetes after that.

Pull request jobs never receive cloud credentials. Only workflows running from the protected `main` branch can request AWS roles.
