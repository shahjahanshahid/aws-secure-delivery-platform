# Platform Overview

## Purpose

This repository builds a small but complete secure delivery platform. A Python API moves from a pull request through automated security gates, into a signed container image, and onto Kubernetes through GitOps: first on a local lab cluster, then on a short-lived AWS environment created and destroyed with Terraform.

The application is deliberately minimal. It exists so the delivery pipeline, infrastructure, and security controls have something real to build, scan, deploy, observe, and break.

## Goals

The platform demonstrates, with evidence:

- Linux administration and networking on real hosts, automated with Ansible.
- Containers built to a hardened baseline: minimal base image, non-root user, pinned dependencies.
- Kubernetes operations on a multi-node K3s cluster, packaged with Helm.
- A CI pipeline in GitHub Actions with layered security gates: SAST, dependency scanning, secret scanning, IaC scanning, container scanning, and DAST.
- Supply-chain integrity: SBOM generation, image signing, and digest-pinned deployments.
- GitOps delivery with Argo CD, where Git is the only path to change a running environment.
- AWS infrastructure as code with Terraform: VPC, IAM, KMS, ECR, and EKS, with remote state and short-lived CI credentials through OIDC.
- Workload identity and secrets on AWS without long-lived keys.
- Observability, alerting, and documented incident response.
- Reproducible, cost-safe teardown with an audit of anything left behind.

## Success criteria

Each criterion is met only when linked evidence exists in `docs/evidence/`.

| Area | Done when |
| --- | --- |
| Repository controls | `main` rejects direct pushes; every change is a pull request that must pass required checks |
| Lab automation | A second Ansible run reports no unintended changes on every host |
| Workload | The API runs on K3s with probes, resource limits, and a non-root security context |
| CI security gates | A deliberately vulnerable change fails each relevant gate; the fixed change passes |
| Supply chain | An unsigned or tampered image is rejected at deploy time |
| GitOps | A change made outside Git is reverted; a bad release is recovered through Git |
| AWS foundation | Terraform plans and applies through a protected workflow; an unauthorized branch cannot assume the deploy role |
| AWS lifecycle | Two clean runs create, verify, and destroy the AWS environment with no orphaned billable resources |
| Workload identity | A pod can read its own secret and is denied access to another |
| Observability | Dashboards stay available while the cluster is deliberately broken; an alert fires and its runbook finds the cause |
| Cost | AWS spend stays under the budget in `docs/cost/budget.md` |

## Non-goals

- A feature-rich application or frontend.
- A continuously running AWS environment, a production SLA, or multi-region availability.
- Running several tools that do the same job.
- Service mesh, Vault, Crossplane, or custom Kubernetes operators.
- Real customer data or production secrets of any kind.
- Compliance certification claims.
- A permanently reachable public demo. Demonstrations are recorded and results published in sanitized form.
- Claiming the local K3s cluster is equivalent to EKS.

## Hardware and roles

| Host | Hardware | Role |
| --- | --- | --- |
| `ops-01` | Laptop, 6 cores / 8 threads, 16 GB RAM, 1 TB NVMe | Admin workstation (Git, Terraform, kubectl, Helm, Ansible) and the observability stack |
| `pve-01` | Desktop, 4 cores / 8 threads, 32 GB RAM, SSD storage | Proxmox VE hypervisor for the K3s virtual machines |
| `k8s-control-01` | VM: 2 vCPU, 4 GB | K3s server (control plane) |
| `k8s-worker-01` | VM: 2 vCPU, 6 GB | K3s agent |
| `k8s-worker-02` | VM: 2 vCPU, 6 GB | K3s agent, used for scheduling and node-failure tests |

`ops-01` is hardened as a credential-holding host: LUKS2 full-disk encryption, Secure Boot, and SSH key-only access.

Hostnames follow a `role-NN` convention so every host can be rebuilt from code under the same name.

## Environments

| Environment | Runs on | Lifecycle | Purpose |
| --- | --- | --- | --- |
| Local lab | Proxmox VMs and `ops-01` | Persistent | Day-to-day delivery, security testing, observability, fault injection |
| AWS sandbox | One dedicated AWS account, one region | Created on demand, destroyed after each session | EKS, IAM, KMS, and Secrets Manager integration |

Terraform remote state lives in a small persistent bootstrap stack that survives teardown. Everything else in AWS is ephemeral.

## Lab tradeoffs

This is a lab built to production-style practices, not a production system. Known simplifications:

- Admin tooling and monitoring share one host. In production they would be separate, with short-lived SSO access from a hardened workstation and monitoring in its own account with high availability.
- Lab hosts sit on a flat home network. Isolation relies on host firewalls and Kubernetes NetworkPolicies rather than network segmentation.
- The hypervisor has a single SSD. Recovery relies on rebuilding from code; backups for stateful components go to a separate disk.
- AWS uses a single account and, if outbound access requires it, a single NAT gateway.

Each tradeoff is recorded in an architecture decision record under `docs/decisions/`.

## Status

See the status table in the [README](../../README.md). Anything not marked implemented there is planned.
