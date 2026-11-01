# ADR-0001: Build and prove the platform locally before AWS

- **Status:** Accepted
- **Date:** 2026-10-15
- **Milestones affected:** M1 to M6

## Context

The project needs to show Linux, networking, Kubernetes, CI/CD, GitOps and
cloud security work. Most of that does not need a cloud provider, and every hour
spent learning on AWS costs money. Lab hardware already exists: a Proxmox host
for virtual machines and a dedicated ops host.

## Decision

Milestones 1 to 4 run entirely on the local lab: Linux hosts managed by
Ansible, a three-node K3s cluster on Proxmox, CI on GitHub-hosted runners, and
Argo CD deploying from Git. No AWS account exists until Milestone 5.

AWS reuses what was already proven locally. The same container image, Helm
chart and GitOps flow deploy to EKS, with AWS-specific values and identity
added on top.

## Alternatives considered

- **Cloud first.** Faster to reach EKS, but every mistake while relearning
  Kubernetes is billed, and the hourly cost discourages breaking things on
  purpose.
- **Single-node local cluster (kind, minikube).** Cheaper still, but no real
  scheduling across nodes, no node failure tests, and no Linux networking
  between hosts.
- **AWS only, skip local.** Loses the hands-on Linux and networking work and
  leaves nothing running when the AWS environment is torn down.

## Consequences

- A working, reviewable platform exists at the end of Milestone 4 with no AWS
  spend.
- CI and the local platform keep working when AWS is destroyed.
- K3s is not EKS. Managed control plane behavior, VPC networking and AWS
  identity are only proven in Milestones 5 to 7, and the docs say so.
- The two environments share charts and pipelines, so differences must stay in
  values files, not in forked code.

## Validation

- Milestone 4: a commit passes CI and reaches the K3s cluster through Argo CD.
- Milestone 6: the same chart and image deploy to EKS with only
  environment-specific values changed.
