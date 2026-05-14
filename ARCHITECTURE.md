
# Architecture Overview

## Design Philosophy
This lab is designed to mirror a production-grade enterprise environment at home lab scale. Every architectural decision maps to a real-world pattern: VLANs replace VPC subnets, Nginx replaces an Application Load Balancer, and Pi-hole replaces Route 53.

See [`aws-mapping.md`](aws-mapping.md) for the explicit AWS SAA-C03 conceptual mapping.

---

## Network Topology

### IP Addressing Scheme
| Segment | Range | Purpose |
| :--- | :--- | :--- |
| **VLAN 10 (Trusted)** | `192.168.10.0/24` | Servers, VMs, infrastructure |
| **VLAN 20 (Lab)** | `192.168.20.0/24` | Test workloads, dev VMs |
| **VLAN 30 (IoT)** | `192.168.30.0/24` | Restricted IoT devices |
| **WireGuard VPN** | `10.0.0.0/24` | Secure remote access tunnel |

### Key Hosts
| Role | Hostname | IP |
| :--- | :--- | :--- |
| Fedora Hypervisor | `fedora-host` | `192.168.10.107` |
| Pi-hole DNS | `pihole-server` | `192.168.10.53` |
| Nginx Gateway | `gateway-server` | `192.168.10.210` |
| Monitoring VM | `monitor-server` | `192.168.10.200` |
| MikroTik (VPN) | `mikrotik` | `10.0.0.1` |
| Fedora Host (VPN) | `fedora-host` | `10.0.0.2` |

---

## Service Architecture

### Traffic Flow (External → Service)
```
Remote Client (WireGuard)
    └─→ MikroTik Hub (10.0.0.1)
            └─→ Nginx Gateway (192.168.10.210:80)
                    ├─→ grafana.lab  → Monitor VM (192.168.10.200:3000)
                    ├─→ prometheus.lab → Monitor VM (192.168.10.200:9090)
                    └─→ uptime.lab   → Fedora Host (192.168.10.107:3001)
```

### Security Perimeter
- **WireGuard** provides encrypted remote access; no direct port exposure.
- **Nginx** acts as the single ingress point (Layer 7 gateway).
- **Firewalld Rich Rules** restrict backend services to accept traffic only from the Gateway VM.
- **Pi-hole** provides DNS-level ad/malware blocking for all VLANs.
- **SELinux** enforces Mandatory Access Control on the Fedora host and VMs.

---

## Automation Layer

All infrastructure is defined as code using **Ansible**. Playbooks cover the full lifecycle:

| Playbook | Purpose |
| :--- | :--- |
| `deploy-vm.yml` | Provision a generic KVM VM via Cloud-Init |
| `deploy-lvm-vm.yml` | Provision a VM with an attached data disk (for K3s/storage) |
| `deploy-pihole.yml` | Deploy Pi-hole VM + Docker container |
| `deploy-plg-stack.yml` | Deploy Prometheus, Loki, Grafana, and Promtail agents |
| `deploy-monitoring.yml` | Deploy Uptime Kuma with bridge/SELinux fixes |
| `deploy-reverse-proxy.yml` | Configure Nginx gateway and harden backend firewalls |

---

## Observability Stack

| Component | Role | Port |
| :--- | :--- | :--- |
| **Prometheus** | Metrics scraper (pull-based) | `9090` |
| **Loki** | Log aggregation | `3100` |
| **Grafana** | Visualization dashboard | `3000` |
| **Promtail** | Log shipper agent (per VM) | `9080` |
| **node-exporter** | System metrics agent (per VM) | `9100` |
| **Uptime Kuma** | Availability / TCP health checks | `3001` |
