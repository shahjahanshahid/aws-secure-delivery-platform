# Threat Model

This threat model uses STRIDE against the trust boundaries in [trust-boundaries.md](../architecture/trust-boundaries.md). It starts as a skeleton: threats are listed now, and each one gets a severity rating and test cases in the milestone where its component is built. Rating components that do not exist yet would be guesswork.

## Scope

In scope: the GitHub repository and its workflows, the local lab hosts and K3s cluster, the container supply chain, the AWS sandbox account, and the observability stack.

Out of scope: the home network's consumer devices, GitHub's and AWS's own infrastructure, and physical attacks beyond theft of a lab host.

## Assets

| Asset | Why it matters |
| --- | --- |
| Source and workflow definitions | Control what gets built and deployed |
| AWS roles and Terraform state | Control and describe the cloud environment |
| Container images and signatures | Determine what code actually runs |
| Cluster admin credentials on `ops-01` | Full control of the lab cluster |
| Application secrets | Grant access to AWS resources |
| Logs and audit trails | Needed to detect and investigate incidents |

## Rating scale

Applied per threat once its component exists.

| Rating | Likelihood | Impact |
| --- | --- | --- |
| High | Plausible with little effort or skill | Cloud account, cluster, or supply chain compromised |
| Medium | Needs a specific mistake or insider access | One workload or environment affected |
| Low | Needs several unlikely failures together | Limited, recoverable effect |

## Threats (STRIDE seed)

| ID | Category | Threat | Boundary | Planned mitigation | Rated in |
| --- | --- | --- | --- | --- | --- |
| T-01 | Spoofing | A workflow from another repository or branch assumes the AWS deploy role | TB-2 | OIDC trust bound to immutable repository and owner IDs; protected environments | M5 |
| T-02 | Spoofing | A stolen SSH key or kubeconfig is used to administer the lab | Lab host | Encrypted `ops-01`, per-device keys, key-only SSH, revocation runbook | M1 |
| T-03 | Tampering | Code is pushed to `main` without review or checks | TB-1 | Branch ruleset with no bypass; required status checks | M0 |
| T-04 | Tampering | A third-party action is replaced through a moved tag | TB-1 | Actions pinned to full commit SHAs; Dependabot updates reviewed by pull request | M3 |
| T-05 | Tampering | An unsigned or modified image is deployed | TB-4 | Cosign signing; signature verification at admission; digest pinning | M3, M7 |
| T-06 | Tampering | Manifests are changed directly in the cluster | TB-4 | Argo CD self-heal; RBAC limiting direct changes | M4 |
| T-07 | Repudiation | A cloud change cannot be traced to a person or workflow run | TB-2, TB-3 | CloudTrail; workflow run IDs in role session names | M5, M7 |
| T-08 | Information disclosure | Secrets are committed to the repository | TB-1 | `.gitignore` deny rules; GitHub secret scanning with push protection; Gitleaks in CI | M0, M3 |
| T-09 | Information disclosure | Terraform state exposes sensitive values | TB-3 | Encrypted, private, versioned bucket; least-privilege access to state | M5 |
| T-10 | Information disclosure | A workload reads another workload's secret | TB-5 | Per-workload Pod Identity roles; scoped Secrets Manager and KMS policies | M7 |
| T-11 | Information disclosure | Tokens or personal data are written to logs | TB-7 | Structured logging with redaction; injected-token test | M8 |
| T-12 | Denial of service | A DAST scan or faulty release takes the service down | TB-4 | Scans only against disposable environments; probes and rollback through Git | M3, M9 |
| T-13 | Denial of service | Forgotten AWS resources exhaust the budget | TB-8 | Teardown audit; budget alerts; lifecycle tags | M6 |
| T-14 | Elevation of privilege | A pull request job obtains write tokens or cloud credentials | TB-1 | Read-only default token; no OIDC on pull request workflows | M0, M3 |
| T-15 | Elevation of privilege | A pod runs privileged and escapes to the node | TB-6 | Non-root security context; Kyverno admission policy | M2, M7 |
| T-16 | Elevation of privilege | An IAM policy grants wildcard permissions | TB-2, TB-3 | Least-privilege policies; IaC scanning; denied-action tests | M5 |

## Change log

Ratings, test cases, and evidence links are added here as each milestone is completed.
