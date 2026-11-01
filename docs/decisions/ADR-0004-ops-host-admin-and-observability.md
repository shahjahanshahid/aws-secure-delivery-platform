# ADR-0004: Dedicated ops host for administration and monitoring

- **Status:** Accepted
- **Date:** 2026-10-16
- **Milestones affected:** M0, M1, M8, M9

## Context

The project needs a Linux machine to run the toolchain (Git, Terraform,
kubectl, Helm, Ansible) and a place to run monitoring (Prometheus, Grafana,
Loki). Milestones 8 and 9 deliberately break the cluster and power off nodes.
If monitoring runs on the cluster or the hypervisor, the dashboards go dark at
the moment they are needed.

A spare laptop was available.

## Decision

The laptop becomes `ops-01`, a bare-metal Ubuntu Server host that is both the
admin workstation and the monitoring host. It is outside the cluster and
outside Proxmox, so it is a separate failure domain. The Windows laptop is used
only as a terminal that connects to `ops-01` over SSH.

Because `ops-01` holds credentials (SSH keys, kubeconfig, cloud sessions), it
is hardened first: full-disk encryption (LUKS2), Secure Boot, key-only SSH and
current patches.

## Alternatives considered

- **WSL2 on the Windows laptop.** No new hardware, but monitoring would stop
  whenever the laptop sleeps, and Windows and Linux tooling would be mixed on
  one machine.
- **Admin VM on Proxmox.** Easy to rebuild, but it shares the hypervisor's
  failure domain and uses RAM the cluster needs.
- **Use the laptop as a K3s worker.** Adds capacity, but leaves no host outside
  the cluster for administration and monitoring.

## Consequences

- Dashboards and alerts stay available while the cluster or Proxmox is broken
  on purpose.
- The Proxmox host's RAM goes entirely to the K3s nodes.
- **Lab tradeoff:** administration and monitoring share one host. A production
  setup would separate them: short-lived admin access from a hardened
  workstation, and monitoring in its own account with high availability and a
  dead man's switch alert. Losing `ops-01` loses both functions until it is
  rebuilt.
- `ops-01` is a laptop on the home network and needs a fixed address and lid
  handling (Milestone 1).

## Validation

- Milestone 0: disk encryption, Secure Boot and key-only SSH verified with
  positive and negative tests (done).
- Milestone 8: dashboards keep showing data while the cluster is down.
