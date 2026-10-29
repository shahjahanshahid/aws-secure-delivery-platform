# Trust Boundaries

A trust boundary is any point where data or control passes between parties trusted to different degrees. Each boundary below lists the threat, the control that addresses it, and the test that proves the control works. Diagrams in [diagrams/](diagrams/) label these boundaries `TB-1` to `TB-8`.

A control is only marked in place when evidence exists. Everything else is planned for the milestone shown.

| ID | Boundary | Principal and path | Main threats | Control | Proof required | Milestone | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| TB-1 | Contributor to GitHub | Pull request code running in CI | Malicious workflow or code runs with privileged tokens; leaked credentials | Protected `main`; required checks; read-only default workflow token; actions pinned to commit SHAs; no cloud credentials on pull requests | A direct push to `main` is rejected; a pull request job cannot obtain a cloud role | M0, M3 | Partly in place |
| TB-2 | GitHub Actions to AWS | OIDC token exchanged for a short-lived role through STS | Overly broad trust lets another repository or branch assume the role | Trust policy bound to immutable repository and owner IDs; protected environments; separate plan and apply roles | An authorized run assumes the role; an unauthorized branch is denied | M5 | Planned |
| TB-3 | Terraform to AWS state | Provisioning role and the S3 state bucket | State leakage, corruption, concurrent writes, accidental deletion | Encrypted, versioned bucket with public access blocked; state locking; least-privilege access | A second concurrent apply is blocked by the lock; an unauthorized principal cannot read state | M5 | Planned |
| TB-4 | GitOps to Kubernetes | Argo CD applying desired state from Git | Unauthorized release; manifest tampering; drift | Protected branches; scoped Argo CD projects; RBAC; digest-pinned images | A change made outside Git is reverted; an unapproved application is denied | M4, M6 | Planned |
| TB-5 | Pod to AWS | EKS Pod Identity for a service account | Secret exfiltration; one workload using another's permissions | Least-privilege role per workload; Secrets Manager and KMS key policies scoped to that role | The pod reads its own secret; reading another secret is denied | M7 | Planned |
| TB-6 | Network to cluster | EKS API, K3s API, and application traffic | Public exposure of the API; lateral movement | EKS endpoint limited to the admin address; host firewalls; NetworkPolicies; no public dashboards | A connection from a non-allowed source or pod is refused | M2, M6, M7 | Planned |
| TB-7 | Workload to telemetry | Logs, metrics, and audit events sent to the observability stack | Secrets written to logs; log injection; unbounded retention | Structured logging with redaction; access-controlled dashboards; retention limits | An injected fake token does not appear in stored logs | M8 | Planned |
| TB-8 | Operator to teardown | Destroy script or protected workflow | Wrong account destroyed; state deleted; billable resources left behind | Account and region guard; manual confirmation; ordered cleanup; post-teardown audit | A run against the wrong account aborts; the audit finds no orphaned resources | M6, M9 | Planned |

## Lab host boundary

`ops-01` holds cluster and cloud credentials, so it is treated as a sensitive host: full-disk encryption, Secure Boot, and SSH key-only access are in place. Its loss would expose admin access, which is why cloud access uses short-lived sessions rather than stored keys.

## How this document is maintained

Severity, likelihood, and detailed test cases for each threat are added in `docs/security/threat-model.md` in the milestone where the component is built. When a control is proven, its status changes here and the evidence is linked.
