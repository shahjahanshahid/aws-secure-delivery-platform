# ADR-0002: One owner per resource across Terraform, Argo CD and Ansible

- **Status:** Accepted
- **Date:** 2026-10-15
- **Milestones affected:** M1, M4, M5, M6

## Context

Three tools change infrastructure in this project. If two of them manage the
same object, each undoes the other's changes, and drift becomes impossible to
reason about.

## Decision

Each layer has exactly one owner:

- **Ansible** owns host state: users, SSH, firewalls, packages and updates on
  the ops host and the Proxmox guests.
- **Terraform** owns cloud infrastructure: VPC, IAM, KMS, ECR, the EKS cluster
  and node group, and the one-time install of Argo CD into a new cluster.
- **Argo CD** owns everything inside Kubernetes after that bootstrap:
  application workloads and platform add-ons, rendered from Helm charts and
  values in Git.

Terraform does not manage Kubernetes workloads after bootstrap. Argo CD does
not create cloud resources. Changes go through Git and a pull request; manual
`kubectl` or console changes are drift, not fixes.

## Alternatives considered

- **Terraform for everything, including workloads** (Helm or Kubernetes
  providers). One tool, but every application release becomes a Terraform
  apply with cloud credentials, and there is no continuous drift correction.
- **Argo CD for everything, including cloud resources** (for example through
  Crossplane). Adds a large component the project does not need; listed as a
  non-goal.
- **Manual `kubectl` and Helm.** Simple at first, with no audit trail or drift
  detection.

## Consequences

- Application releases need no cloud credentials; they are Git changes that
  Argo CD applies.
- The Argo CD bootstrap is the one handoff point and must be documented
  carefully, including how a cluster is rebuilt from scratch.
- Kubernetes controllers can still create AWS resources (load balancers,
  volumes). Those are deleted before Terraform destroys the cluster, as set out
  in the [AWS resource lifecycle](../architecture/aws-resource-lifecycle.md).

## Validation

- Milestone 4: a manual change to a deployed object is reverted by Argo CD.
- Milestone 6: after Argo CD syncs, `terraform plan` shows no changes.
- Milestone 1: a second Ansible run reports no changes.
