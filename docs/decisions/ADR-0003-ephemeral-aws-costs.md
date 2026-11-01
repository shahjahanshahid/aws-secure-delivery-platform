# ADR-0003: Ephemeral AWS environment with a small persistent foundation

- **Status:** Accepted
- **Date:** 2026-11-01
- **Milestones affected:** M5, M6, M9

## Context

The AWS side of this project has to show real EKS, IAM, KMS and Terraform work,
but the budget is under $50 per month of AWS usage, calculated before credits.
An EKS cluster with workers and a NAT gateway left running costs about $5.70 a
day, or roughly $170 a month. Running it full time is not an option.

The local platform (K3s on Proxmox, CI on GitHub-hosted runners, monitoring on
the ops host) costs nothing in AWS and covers most day-to-day work.

## Decision

The AWS environment is **ephemeral**. It is created for a working session of a
few hours, verified, and destroyed by the same code every time. Only a small
**persistent foundation** stays between sessions: the Terraform state bucket,
budget alerts, the GitHub OIDC provider and CI roles, and human admin access.

Ownership, lifecycle classes, state design and teardown order are defined in
[AWS resource lifecycle](../architecture/aws-resource-lifecycle.md).

## Cost

Estimates from US East (Ohio) list prices, to be re-priced before the first
apply in M5 and replaced with billed amounts after the first sessions:

- A 6-hour session (EKS control plane, two small workers, one NAT gateway, no
  load balancer) is about **$2**.
- Six to eight sessions a month is about **$12 to $25**.
- The persistent foundation costs cents per month.

The risk is not session cost but a **forgotten environment**, which crosses
$50 in under nine days. A second risk is a cluster on a Kubernetes version
outside standard support, where the control-plane fee is six times higher.

## Guardrails

- AWS Budgets alerts at $5, $15, $30 and $40. A forgotten session crosses $5
  within a day.
- At $35 projected for the month, stop and review before any new session.
- `aws-audit` runs after every session and queries AWS directly, not Terraform
  state.
- Preflight refuses the wrong account or region and an EKS version outside
  standard support.
- No load balancer, interface endpoints or second NAT gateway unless a specific
  demo needs one, and then only for that session.

Budgets are alerts, not caps. Billing data lags by hours, so teardown and audit
are the real controls.

## Alternatives considered

| Option | Why not |
| --- | --- |
| EKS running full time | About $170 a month, over three times the limit |
| Local only, no AWS | Loses the cloud IAM, KMS, OIDC and EKS evidence the project exists to show |
| Self-managed Kubernetes on EC2 instead of EKS | Cheaper, but skips managed EKS and Pod Identity, which is what employers use |
| Rely on credits | Credits expire; the design must be affordable without them |

## Consequences

- Every AWS change must be reproducible from code; nothing fixed by hand in the
  console survives a rebuild.
- Each session pays a startup cost of cluster creation time (roughly 15 to 20
  minutes).
- The account plan is chosen at signup in M5. A Free plan account closes itself
  after six months or when credits run out, which would delete the persistent
  state, so a Paid plan is expected.
- One NAT gateway in one Availability Zone is a lab tradeoff, not a resilient
  design.

## Validation

- M6: two clean create, verify, destroy cycles from code, each followed by an
  audit showing only persistent and expected residual resources.
- M6 and M9: actual billed cost per session recorded and compared with the
  estimate above.
