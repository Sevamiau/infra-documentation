
# Self-Hosted Infrastructure Lab

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Fedora](https://img.shields.io/badge/Fedora-51A2DA?style=flat&logo=fedora&logoColor=white)
![Kubernetes](https://img.shields.io/badge/K3s-FFC61C?style=flat&logo=kubernetes&logoColor=black)
![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat&logo=wireguard&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)

## About This Project

This repository documents a production-grade self-hosted infrastructure environment built to apply and demonstrate real-world Infrastructure and DevOps engineering. Every component — from VM provisioning to firewall hardening — is defined as code and follows patterns used in professional production environments.

The infrastructure is architected as a private enterprise-grade network: a physical Fedora hypervisor runs QEMU/KVM virtual machines, a MikroTik router enforces VLAN segmentation and WireGuard VPN access, and Ansible automates the full deployment lifecycle. Each service runs in its own dedicated VM for isolation and security — including a gateway VM running Nginx as a reverse proxy. Docker contains the individual service workloads, Pi-hole handles network-wide DNS filtering and manages local DNS records for all internal services, and Jellyfin runs as a self-hosted media server. A PLG stack (Promtail, Loki, Grafana) alongside Uptime Kuma provide end-to-end observability. Incident reports document real operational problems encountered and resolved.

> **Mirror documentation.** This repository is a mirror of the configuration and documentation running on the live server. It reflects the actual state of the infrastructure as closely as possible, with configs, playbooks, and write-ups kept in sync with what is deployed in production.

> **Work in progress.** This lab is actively evolving. Improving security hardening and service availability are the top ongoing priorities — expect to see continued work in those areas across the codebase.

## How to Navigate This Repo

Each top-level directory is self-contained and focused on a specific concern:

- Start with **`infrastructure/`** for an overview of how VMs are provisioned and how storage is organised.
- Read **`networking/`** to understand the VLAN layout, WireGuard setup, and jump-host design.
- See **`observability/`** for how logs and metrics are collected and visualised with the PLG stack.
- Browse **`automation/ansible/`** to explore the playbooks and inventory that tie everything together.
- Check **`incident-reports/`** for detailed write-ups of real operational problems encountered and how they were diagnosed and resolved.

> **A note on storage:** Persistent data is backed by a 20 TB NAS, which provides storage for VM disk images, media libraries, and service data volumes. None of this is tracked in the repository. The documentation in `infrastructure/` describes the storage layout (LVM thin pools, qcow2 images, bind-mount paths, NAS mounts) so the setup can be reproduced, but the data itself lives outside version control.

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
    %% External & Entry
    subgraph ISP [Public Internet]
        RemoteAdmin((Remote Admin))
        Internet((Internet))
    end

    subgraph Entry [Entry Point]
        Modem[[Movistar Modem<br/>'Bridge Mode']]
    end

    subgraph Storage [Storage]
        NAS[(20 TB NAS)]
    end


    %% Router Layer
    subgraph Router [Gateway: Mikrotik L009]
        L009[L009 Routing Logic<br/>'Holds Fixed IP']
        WG{{"WireGuard VPN<br/>(Admin Tunnel)"}}
    end

    %% Storage Path
    Server -.->|NFS/SMB| NAS

    %% Server Layer
    subgraph Server [Fedora Server]
        PhysNIC[Physical NIC]
        Bridge["'the bridge'<br/>(VLAN Aware)"]
        
        subgraph VLAN10 [VLAN 10: Trusted]
            PH[Admin]
        end

        subgraph VLAN20 [VLAN 20: Lab]
            JF[Lab]
        end

        subgraph VLAN30 [VLAN 30: IoT]
            IoT[IoT Devices]
        end
    end

    %% Connections
    Internet --- Modem
    Modem --- L009
    RemoteAdmin -.->|"Encrypted Tunnel"| WG
    
    %% The Trunk Link
    L009 ---|"Trunk Link (Tagged 10,20,30)"| PhysNIC
    PhysNIC --- Bridge
    
    %% VPN Landing Path
    WG ==>|"Direct Admin Access"| VLAN10

    %% Bridge Distribution
    Bridge --- VLAN10
    Bridge --- VLAN20
    Bridge --- VLAN30

    %% Styling
    style Modem fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    style Movistar fill:#f9f,stroke:#333
    style RemoteAdmin fill:#2ecc71,stroke:#333
    style L009 fill:#bbf,stroke:#333
    style WG fill:#3498db,color:#fff,stroke-width:2px
    style Bridge fill:#dfd,stroke:#333,stroke-width:4px
    style VLAN10 fill:#e8f5e9,stroke:#2e7d32
    style VLAN20 fill:#fff3e0,stroke:#ef6c00
    style VLAN30 fill:#ffebee,stroke:#c62828
    style PhysNIC fill:#eee,stroke:#333
```
```

graph TB
    subgraph Host [Fedora Server Layer]
        direction TB
        
        %% The Bridge
        Bridge{{"the bridge<br/>(Virtual Switch)"}}

        subgraph KVM [QEMU/KVM Hypervisor Layer]
            direction LR
            
            %% The VMs
            GW[<b>1. gateway-server</b><br/>Nginx Reverse Proxy]
            PH[<b>2. pihole-server</b><br/>DNS & Filtering]
            MON[<b>3. monitor-server</b><br/>PLG Stack + Kuma]
            JF[<b>4. jellyfin-server</b><br/>Media Services]
            K3S[<b>5. k3s-test</b><br/>K3s Cluster]
        end

        %% Connections to Bridge
        Bridge <--> GW
        Bridge <--> PH
        Bridge <--> MON
        Bridge <--> JF
        Bridge <--> K3S
    end

    %% Traffic Flow
    GW -->|Proxy Pass| MON
    GW -->|Proxy Pass| JF
    GW -->|Proxy Pass| K3S

    %% Styling
    style Host fill:#f5f5f5,stroke:#333,stroke-width:2px
    style Bridge fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    style GW fill:#fff9c4,stroke:#fbc02d
    style PH fill:#ffebee,stroke:#c62828
    style KVM fill:#f3e5f5,stroke:#7b1fa2
