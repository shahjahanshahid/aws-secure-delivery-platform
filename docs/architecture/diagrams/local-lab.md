# Local Lab

Physical hosts, virtual machines, and what manages each layer. Labels `TB-n` mark trust boundaries, defined in [trust-boundaries.md](../trust-boundaries.md).

```mermaid
flowchart TB
  term["Admin terminal"] -->|"SSH, keys only"| ops

  subgraph home["Home network (flat, no inbound internet access)"]
    subgraph opsbox["ops-01: encrypted laptop"]
      ops["Admin tools: Git, Terraform, kubectl, Helm, Ansible"]
      obs["Observability: Prometheus, Loki, Grafana"]
    end

    subgraph pve["pve-01: Proxmox VE"]
      subgraph k3s["K3s cluster (TB-6)"]
        cp["k8s-control-01: K3s server"]
        w1["k8s-worker-01: K3s agent"]
        w2["k8s-worker-02: K3s agent"]
        argo["Argo CD"]
        app["Demo API pods"]
      end
    end
  end

  ops -->|"Ansible over SSH"| pve
  ops -->|"Ansible over SSH"| cp
  ops -->|"Ansible over SSH"| w1
  ops -->|"Ansible over SSH"| w2
  ops -->|"kubectl and Helm, port 6443"| cp
  w1 -->|"join, port 6443"| cp
  w2 -->|"join, port 6443"| cp
  argo -->|"reconciles from Git (TB-4)"| app
  app -->|"metrics and logs (TB-7)"| obs
  cp -->|"node metrics"| obs
```

Ownership: Ansible configures every host; K3s runs the cluster services; Argo CD deploys workloads from Git.

Observability runs on `ops-01`, outside the cluster and the hypervisor, so dashboards keep working while the cluster is deliberately broken.
