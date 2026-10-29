# AWS Environment (planned)

The short-lived AWS sandbox. Everything except the Terraform state bootstrap is created on demand and destroyed after each session. Labels `TB-n` mark trust boundaries, defined in [trust-boundaries.md](../trust-boundaries.md).

```mermaid
flowchart TB
  ghaws["GitHub Actions on main"] -->|"OIDC to STS, short-lived role (TB-2)"| iam
  admin["ops-01, admin egress address only"] -->|"restricted EKS API endpoint (TB-6)"| eks

  subgraph acct["Dedicated AWS sandbox account"]
    subgraph persistent["Persistent bootstrap"]
      state["S3 Terraform state: versioned, encrypted, locked"]
    end

    iam["IAM roles: separate plan and apply"]
    kms["KMS customer-managed key"]
    sm["Secrets Manager"]
    ecr["ECR repository"]
    ct["CloudTrail audit log"]

    subgraph vpc["VPC 10.80.0.0/16, two availability zones"]
      subgraph priv["Private subnets"]
        eks["EKS control plane and managed node group"]
        pod["Demo API pods"]
      end
      pub["Public subnets: NAT only if required"]
    end
  end

  iam -->|"Terraform (TB-3)"| state
  iam -->|"Terraform provisions"| vpc
  eks --> pod
  pod -->|"EKS Pod Identity (TB-5)"| sm
  sm -->|"decrypt"| kms
  ecr -->|"signed images by digest"| pod
  priv -->|"outbound only"| pub
  iam -.->|"API calls recorded"| ct
```

Teardown (TB-8) destroys everything inside the account except the state bootstrap, then audits for anything left behind.
