# AWS Resource Lifecycle and Teardown

> **Status: planned.** Nothing in this document is implemented yet. The
> lifecycle commands are built in Milestone 6. No AWS account exists.

Every AWS resource in this project has an owner, a lifecycle class and a cost
class. Anything that cannot be placed in this table does not get created.

## Lifecycle classes

| Class | Meaning | Created by | Destroyed by |
| --- | --- | --- | --- |
| **Persistent** | Survives between sessions; small and cheap | `infrastructure/terraform/bootstrap` | Manual, deliberate, never by automation |
| **Ephemeral** | Exists only during a working session | `infrastructure/terraform/environments/aws-lab` and Argo CD | `aws-down` at the end of every session |
| **Residual** | Left over after teardown by design or by AWS behavior | Side effect | Documented, tracked by `aws-audit`, expires or is removed manually |

## Resource inventory

### Persistent foundation

| Resource | Owner | Cost driver | Notes |
| --- | --- | --- | --- |
| S3 bucket for Terraform state | Bootstrap root | Storage and requests; cents per month | See state design below |
| AWS Budgets and alert subscriptions | Bootstrap root or manual account setup | Free | Thresholds in [ADR-0003](../decisions/ADR-0003-ephemeral-aws-costs.md) |
| GitHub OIDC identity provider and CI roles | Bootstrap root | Free | Trust bound to immutable GitHub owner and repository IDs |
| Human admin access (IAM Identity Center) | Manual account setup, documented | Free | Short-lived sessions from the ops host |
| ECR repository (optional) | Bootstrap root, only if justified in M5 | Storage per GB | Lifecycle policy keeps a small number of images |

### Ephemeral environment

| Resource | Owner | Cost driver |
| --- | --- | --- |
| VPC, subnets, route tables, internet gateway, security groups | `aws-lab` root | Free |
| NAT gateway and its Elastic IP | `aws-lab` root | Hourly + per GB + public IPv4 |
| EKS cluster | `aws-lab` root | Hourly control-plane fee |
| Managed node group (EC2 + EBS root volumes) | `aws-lab` root | Instance hours + volume storage |
| EKS add-ons, Pod Identity associations, IAM workload roles | `aws-lab` root | Free |
| KMS customer managed keys for the IAM/KMS demos | `aws-lab` root | $1 per key-month, prorated |
| Secrets Manager secrets for the demo workload | `aws-lab` root | Per secret-month, prorated |
| CloudWatch log groups, including the EKS control-plane log group | `aws-lab` root, created **before** the cluster with a retention period | Ingestion + storage |
| Argo CD, application workloads | Argo CD after bootstrap | Runs on the node group |
| Load balancers, persistent volumes created by Kubernetes | Kubernetes controllers (none by default) | Hourly / storage; can orphan |

The EKS control-plane log group is created by Terraform ahead of time. If EKS
creates it instead, it gets unlimited retention and Terraform does not own it.

## Terraform state design (persistent)

State lives apart from the environment it describes, so destroying the lab can
never destroy its own state.

| Property | Setting |
| --- | --- |
| Separate root | `bootstrap/` holds the bucket; `environments/aws-lab/` holds ephemeral resources and uses that bucket as its backend |
| Encryption | Server-side encryption on by default |
| Public access | All four Block Public Access settings on |
| Versioning | On, so a corrupted or deleted state file can be restored |
| Old versions | Lifecycle rule expires noncurrent versions after a fixed period to bound cost |
| Locking | S3-native lockfile (`use_lockfile = true`). DynamoDB-based locking is deprecated in current Terraform and is not used |
| Access | Only the plan/apply roles and the human admin can read or write state; TLS-only bucket policy |
| Bootstrap's own state | Stored locally on the ops host's encrypted disk and backed up; never committed |
| Deletion | `prevent_destroy` on the bucket; no automation is allowed to delete it |

Planned proofs (M5): a second concurrent `plan` fails on the lock, a principal
without access is denied, and a previous state version is restored.

## Required tags

Every taggable resource carries:

| Tag | Value |
| --- | --- |
| `Project` | `aws-secure-delivery` |
| `Environment` | `aws-lab` |
| `Owner` | `portfolio` |
| `Lifecycle` | `persistent` or `ephemeral` |
| `ManagedBy` | `terraform`, `argocd` or `manual` |
| `ExpiresAt` | UTC timestamp, ephemeral resources only |

Tags are applied through the AWS provider's `default_tags`, so a resource
cannot be created without them.

## Lifecycle commands (planned, M6)

Run from the ops host, or from a manually triggered, protected GitHub Actions
workflow. Nothing applies or destroys automatically.

| Command | What it does | Safety checks |
| --- | --- | --- |
| `make aws-preflight` | Confirms identity, account, region, budget status, tool versions and EKS version support | Aborts on an account not on the private allowlist, wrong region, expired session, or an EKS version outside standard support |
| `make aws-plan` | Terraform plan plus IaC scan and cost review | Plan files are short-lived and never published |
| `make aws-up` | Applies the reviewed plan | Requires explicit confirmation; rejects unexpected resources |
| `make aws-deploy` | Bootstraps Argo CD; Argo CD deploys the workload | Terraform and Argo CD do not manage the same objects |
| `make aws-verify` | Health, IAM/KMS allow and deny tests, network and smoke tests | Fails if a security control is missing |
| `make aws-down` | Ordered teardown of the ephemeral environment | Never touches the bootstrap root |
| `make aws-audit` | Independent inventory of what is left and what it costs | Reads AWS directly, not Terraform state |

The account allowlist lives in a gitignored file on the ops host. Account IDs
never appear in the public repository.

## Teardown order (`aws-down`)

Kubernetes creates some AWS resources itself. Those must go first, or they
block VPC deletion and keep billing.

1. **Preflight:** confirm account, region and that the target is `aws-lab`.
2. **Capture evidence** needed from the running environment.
3. **Delete Kubernetes objects that own AWS resources:** `Service` type
   `LoadBalancer`, `Ingress`, and `PersistentVolumeClaim` objects.
4. **Wait** for the controllers to remove the load balancers and volumes.
   Time out and stop, rather than continue, if they remain.
5. **Remove Argo CD** and platform add-ons as needed.
6. **`terraform destroy`** the `aws-lab` root.
7. **Run `aws-audit`** and record the result in the session cost report.

If a step fails, stop and follow the teardown runbook. Do not force-delete
resources out of order.

## Expected residuals

`aws-audit` reports these as known and expected, not as failures:

| Residual | Why it remains | Cost |
| --- | --- | --- |
| KMS keys pending deletion | AWS enforces a 7 to 30 day waiting period | None while pending; cancelling the deletion bills the key as if it had never been scheduled |
| Noncurrent state object versions | Versioning is on by design | Cents; bounded by the lifecycle rule |
| Persistent foundation | By design | Cents per month |

## Audit checklist (`aws-audit`)

An empty Terraform state does not prove nothing is left. The audit queries AWS
directly in the lab region, and for global services:

- EKS clusters and node groups
- EC2 instances, EBS volumes and snapshots
- NAT gateways, Elastic IPs and any other public IPv4 addresses
- Load balancers and target groups
- Network interfaces left in the VPC
- VPCs other than the default
- KMS keys and their state
- Secrets Manager secrets, including those scheduled for deletion
- CloudWatch log groups and their retention
- ECR repositories and image counts
- Any resource tagged `Project=aws-secure-delivery` with an `ExpiresAt` in the
  past
- Month-to-date cost and forecast

The result is a pass or fail report: pass means only persistent and expected
residual resources remain.

## Related

- [ADR-0003](../decisions/ADR-0003-ephemeral-aws-costs.md): why the AWS
  environment is ephemeral, cost target and guardrails
- [Trust boundaries](trust-boundaries.md): operator to teardown boundary
