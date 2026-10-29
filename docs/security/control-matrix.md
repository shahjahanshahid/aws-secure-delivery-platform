# Control Matrix

Every security control in the platform, the tool that implements it, and its current state. A control is marked in place only when evidence exists; the evidence column links to it once published.

| Control | What it prevents or detects | Implementation | Threats | Milestone | Status |
| --- | --- | --- | --- | --- | --- |
| Protected `main` branch | Unreviewed or forced changes | GitHub ruleset stored as code, no bypass actors | T-03 | M0 | In place |
| Required status checks | Merging changes that fail checks | Ruleset requires `markdownlint` (more checks added in M3) | T-03 | M0 | In place |
| Read-only default workflow token | Workflows writing to the repository by default | Repository Actions setting; explicit per-job permissions | T-14 | M0 | In place |
| Action pinning | Tampered third-party actions | Actions referenced by full commit SHA | T-04 | M0 | In place |
| Secret deny rules | Committing state files, keys, or `.env` files | `.gitignore`, tested with decoy files | T-08 | M0 | In place |
| Secret scanning and push protection | Pushing recognizable secrets | GitHub secret scanning | T-08 | M0 | In place |
| Dependency and malware alerts | Known vulnerable or malicious packages | Dependabot alerts | — | M0 | In place |
| Private vulnerability reporting | Public disclosure of security issues | GitHub private vulnerability reporting, `SECURITY.md` | — | M0 | In place |
| Admin host hardening | Credential theft from a lost or stolen host | LUKS2 encryption, Secure Boot, key-only SSH on `ops-01` | T-02 | M0 | In place |
| Host firewalls | Unneeded inbound connections to lab hosts | nftables or ufw, managed by Ansible | T-02 | M1 | Planned |
| Container hardening | Privileged or root containers | Non-root image, dropped capabilities, read-only root filesystem | T-15 | M2 | Planned |
| SAST | Insecure code patterns | Semgrep | — | M3 | Planned |
| Dependency scanning (SCA) | Vulnerable Python dependencies | pip-audit | — | M3 | Planned |
| Secret scanning in CI | Secrets in code or history | Gitleaks | T-08 | M3 | Planned |
| IaC scanning | Insecure Terraform configuration | Checkov | T-16 | M3 | Planned |
| Container scanning | Vulnerable image contents | Trivy | — | M3 | Planned |
| DAST | Runtime API weaknesses | OWASP ZAP against disposable environments | T-12 | M3 | Planned |
| SBOM | Unknown image contents | Syft | T-05 | M3 | Planned |
| Image signing | Unsigned or modified images | Cosign | T-05 | M3 | Planned |
| GitOps reconciliation | Manual drift in the cluster | Argo CD with self-heal | T-06 | M4 | Planned |
| CI federation | Long-lived cloud keys in CI | GitHub OIDC to AWS STS, trust bound to immutable IDs | T-01, T-14 | M5 | Planned |
| State protection | State leakage or corruption | Encrypted, versioned, locked S3 state | T-09 | M5 | Planned |
| Least-privilege IAM | Excessive cloud permissions | Scoped IAM policies, separate plan and apply roles | T-16 | M5 | Planned |
| Cloud audit trail | Untraceable cloud changes | CloudTrail | T-07 | M5 | Planned |
| Teardown guard and audit | Wrong-account destroys and orphaned costs | Account and region preflight, post-teardown audit | T-13 | M6 | Planned |
| Workload identity | Shared or long-lived app credentials | EKS Pod Identity | T-10 | M7 | Planned |
| Secrets and encryption | Secret exposure at rest | Secrets Manager with a KMS customer-managed key | T-10 | M7 | Planned |
| Admission control | Unsafe or unsigned workloads | Kyverno policies, signature verification | T-05, T-15 | M7 | Planned |
| Network policy | Lateral movement in the cluster | Kubernetes NetworkPolicies, default deny | T-15 | M7 | Planned |
| Log redaction | Secrets or personal data in logs | Structured logging, redaction tests | T-11 | M8 | Planned |
| Alerting and runbooks | Undetected or slow-to-fix incidents | Prometheus alerts linked to tested runbooks | T-12 | M8 | Planned |

Evidence for each in-place control is collected in `docs/evidence/`.
