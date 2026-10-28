# Networking

This document describes the lab and cloud networks by role. Real host addresses are deliberately left out of this public repository; a private, untracked inventory maps each hostname to its address.

## Lab network

All lab hosts sit on one flat home network: a private `/24` inside `192.168.0.0/16`, served by a consumer mesh router that provides DHCP and DNS.

| Host | Network attachment | Addressing |
| --- | --- | --- |
| `ops-01` | Wired Ethernet | DHCP reservation |
| `pve-01` | Wired Ethernet, Linux bridge `vmbr0` | Static management address |
| `k8s-control-01` | Virtual NIC on `vmbr0` | DHCP reservation |
| `k8s-worker-01` | Virtual NIC on `vmbr0` | DHCP reservation |
| `k8s-worker-02` | Virtual NIC on `vmbr0` | DHCP reservation |

Every lab host gets a stable address so SSH configuration, the Ansible inventory, and monitoring targets never change underneath the automation.

Nothing in the lab is reachable from the internet. There are no router port forwards, and no management interface is published.

## Address plan and collision check

AWS and Kubernetes ranges are chosen so no two networks overlap, even though the lab and the VPC are never connected by a VPN.

| Network | CIDR | Notes |
| --- | --- | --- |
| Home LAN | A `/24` inside `192.168.0.0/16` | Exact range kept private |
| K3s pod network | `10.42.0.0/16` | K3s default |
| K3s service network | `10.43.0.0/16` | K3s default |
| AWS VPC (planned) | `10.80.0.0/16` | Two availability zones, public and private subnets |

Result: no overlaps. Recheck this table before any range changes.

## Allowed traffic

Host firewalls on every Linux host allow only the flows below. Everything else inbound is denied. Rules are managed by Ansible so they are reviewable and repeatable.

| From | To | Port | Purpose | Introduced |
| --- | --- | --- | --- | --- |
| Admin terminal | `ops-01` | 22/tcp | SSH (keys only) | Done |
| `ops-01` | All lab hosts | 22/tcp | SSH and Ansible (keys only) | M1 |
| `ops-01`, admin terminal | `pve-01` | 8006/tcp | Proxmox web UI | M1 |
| `ops-01` | `k8s-control-01` | 6443/tcp | Kubernetes API (kubectl, Helm) | M2 |
| K3s agents | `k8s-control-01` | 6443/tcp | K3s supervisor and API server | M2 |
| All K3s nodes | All K3s nodes | 8472/udp | Flannel VXLAN pod networking | M2 |
| All K3s nodes | All K3s nodes | 10250/tcp | Kubelet API and metrics | M2 |
| `ops-01` | Lab hosts | 9100/tcp | Prometheus scraping node metrics | M8 |
| Lab hosts | `ops-01` | 3100/tcp | Log shipping to Loki | M8 |
| Admin terminal | `ops-01` | 3000/tcp | Grafana dashboards | M8 |

Monitoring ports are the tools' defaults and are confirmed when the stack is built in M8.

Outbound, lab hosts reach the internet over HTTPS for OS packages, GitHub, and container registries.

## Kubernetes network policy

Inside the cluster, NetworkPolicies default to deny and allow only the traffic each workload needs. This compensates for the flat lab network, where segmentation is not available. Policies are added with the workload in M2 and tested with allowed and denied connections in M7.

## AWS network (planned)

- One VPC, `10.80.0.0/16`, across two availability zones.
- Public subnets hold only what must face the internet. EKS worker nodes run in private subnets.
- Outbound access for private subnets is decided in M5 against cost: a single NAT gateway if required, documented as a single-zone dependency.
- The EKS API endpoint is restricted to the admin egress address. It is never open to `0.0.0.0/0`.
- No load balancer runs by default. Demo endpoints are reached with `kubectl port-forward`.

## Lab tradeoff

A flat network means a compromised lab host can reach other devices on the home network. Mitigations are host firewalls, NetworkPolicies, key-only SSH, and no internet exposure. Network segmentation with a router VM or VLANs is a planned enhancement and will get its own decision record.
