
# Self-Hosted Infrastructure Lab

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Fedora](https://img.shields.io/badge/Fedora-51A2DA?style=flat&logo=fedora&logoColor=white)
![Kubernetes](https://img.shields.io/badge/K3s-FFC61C?style=flat&logo=kubernetes&logoColor=black)
![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat&logo=wireguard&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)

## About This Project

This repository documents a production-grade home lab built to practice real-world Infrastructure and DevOps engineering. Every component — from VM provisioning to firewall hardening — is defined as code and reflects patterns used in professional environments.

The lab simulates an enterprise network: a physical Fedora hypervisor runs QEMU/KVM virtual machines, a MikroTik router enforces VLAN segmentation and WireGuard VPN access, Ansible automates the full deployment lifecycle, and a PLG stack (Prometheus, Loki, Grafana) provides end-to-end observability. Incident reports document real problems encountered and resolved during the build.

---

## Core Components
- **Hypervisor:** Fedora Server running QEMU/KVM VMs.
- **Automation:** Ansible playbooks for VM provisioning and service deployment.
- **Networking:** MikroTik RouterOS with VLAN segmentation and WireGuard VPN for secure remote access.
- **Observability:** PLG Stack (Promtail, Loki, Grafana) and Uptime Kuma for proactive monitoring.
- **Hardware Integration:** CyberPower UPS monitoring for graceful shutdown/power management.

## Tech Stack
- **OS:** Fedora Server (host and VMs)
- **Networking:** MikroTik (VLANs, Bridge, Firewall), WireGuard, RouterOS
- **DevOps:** Ansible, Docker, Cloud-Init
- **Monitoring:** Grafana, Loki, Promtail, Uptime Kuma
- **Security:** Pi-hole (DNS Filtering), Nginx Reverse Proxy

## Repository Structure

```
.
├── infrastructure/        # VM provisioning, storage, K3s, reverse proxy
├── networking/            # WireGuard VPN, MikroTik, jump-host architecture
├── observability/         # PLG stack (Prometheus/Loki/Grafana), Uptime Kuma
├── automation/ansible/    # Playbooks, group_vars, and inventory for all deployments
└── incident-reports/      # Real-world debugging and resolution write-ups
```

## Network Architecture

```mermaid
graph TD
    subgraph Remote_Users [Remote Access]
        Laptop[Laptop - 10.0.0.3]
        Workstation[Workstation - 10.0.0.4]
    end

    Internet((Internet))

    subgraph Office_Gateway [MikroTik Router - Fixed Public IP]
        WG[WireGuard Hub]
        FW{Firewall Rules}
        VLAN10[VLAN 10: Trusted]
        VLAN20[VLAN 20: Lab]
        VLAN30[VLAN 30: IoT]
    end

    subgraph Virtualization_Host [Fedora Server]
        Host[Fedora Host - 192.168.10.107]
        BR10[Bridge br10]
        Pihole[Pi-hole VM - 192.168.10.53]
    end

    Laptop -- "WireGuard Tunnel" --> Internet
    Workstation -- "WireGuard Tunnel" --> Internet
    Internet --> WG
    WG --> FW

    FW --> VLAN10
    FW --> VLAN20
    FW --> VLAN30

    VLAN10 -- "Trunk Port" --> BR10
    BR10 --> Host
    BR10 --> Pihole

    VLAN20 -. "DNS Traffic (Port 53)" .-> Pihole
    VLAN30 -. "DNS Traffic (Port 53)" .-> Pihole
    Remote_Users -. "Encrypted DNS" .-> Pihole

    style Pihole fill:#f96,stroke:#333,stroke-width:2px
    style WG fill:#bbf,stroke:#333,stroke-width:2px
    style FW fill:#f66,stroke:#333,stroke-width:2px
```
