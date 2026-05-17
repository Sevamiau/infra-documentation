
# Self-Hosted Infrastructure Lab

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Fedora](https://img.shields.io/badge/Fedora-51A2DA?style=flat&logo=fedora&logoColor=white)
![Kubernetes](https://img.shields.io/badge/K3s-FFC61C?style=flat&logo=kubernetes&logoColor=black)
![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=flat&logo=wireguard&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)

## About This Project

This repository documents a live, self-hosted infrastructure environment running on bare-metal hardware. It is built around security-first principles. Every layer, from network segmentation to service exposure, is deliberately designed and version-controlled.

A physical Fedora hypervisor runs QEMU/KVM virtual machines — each service in its own dedicated VM for isolation. A MikroTik router enforces VLAN segmentation, and WireGuard VPN provides secure remote access. Ansible automates the full deployment lifecycle from VM provisioning to service configuration.

Services include an Nginx reverse proxy gateway, Pi-hole for network-wide DNS filtering, Jellyfin as a self-hosted media server, and a K3s cluster for container orchestration. The Grafana Stack (Alloy, Loki, Grafana) alongside Uptime Kuma provide end-to-end observability. Incident reports document real operational problems encountered and resolved.

> **Mirror documentation.** This repository is a mirror of the configuration and documentation running on the live server. It reflects the actual state of the infrastructure as closely as possible, with configs, playbooks, and write-ups kept in sync with what is deployed in production.

> **Actively maintained.** This lab is continuously evolving — ongoing work focuses on security hardening, service availability.

## How to Navigate This Repo

Each top-level directory is self-contained and focused on a specific concern:

- Start with **`infrastructure/`** for an overview of how VMs are provisioned and how storage is organised.
- Read **`networking/`** to understand the VLAN layout, WireGuard setup, and jump-host design.
- See **`observability/`** for how logs and metrics are collected and visualised with the Grafana Stack (Alloy, Loki, Grafana).
- Browse **`automation/ansible/`** to explore the playbooks and inventory that tie everything together.
- Check **`incident-reports/`** for detailed write-ups of real operational problems encountered and how they were diagnosed and resolved.

---

## Stack & Components
- **Virtualisation:** QEMU/KVM on Fedora Server — each service runs in a dedicated VM for isolation
- **Automation:** Ansible for full lifecycle management (provisioning, config, deployment), Cloud-Init for VM bootstrap
- **Networking:** MikroTik RouterOS (VLANs, firewall), WireGuard VPN, Pi-hole for DNS filtering
- **Services:** Nginx reverse proxy, Jellyfin media server, K3s cluster — containerised with Docker
- **Observability:** Grafana Stack (Alloy, Loki, Grafana) and Uptime Kuma
- **Hardware:** CyberPower UPS monitoring for graceful shutdown

## Repository Structure

```
.
├── infrastructure/        # VM provisioning, storage, K3s, reverse proxy
├── networking/            # WireGuard VPN, MikroTik, jump-host architecture
├── observability/         # Grafana Stack (Alloy, Loki, Grafana), Uptime Kuma
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
        Modem[[Movistar Modem Mode Bridge]]
    end

    subgraph Storage [Storage]
        NAS[(20 TB NAS)]
    end

    %% Router Layer
    subgraph Router [Gateway: Mikrotik L009]
        L009[L009 Routing Logic]
        WG{{"WireGuard VPN Admin Tunnel"}}
    end

    %% Server Layer
    subgraph Server [Fedora Server]
        PhysNIC[Physical NIC]
        Bridge["br0 (VLAN-Aware Bridge)"]
        
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
    RemoteAdmin -.->|Encrypted Tunnel| WG
    
    %% The Trunk Link
    L009 ---|Trunk Link Tagged| PhysNIC
    PhysNIC --- Bridge
    
    %% VPN Landing Path
    WG ==>|Direct Admin Access| VLAN10

    %% Bridge Distribution
    Bridge --- VLAN10
    Bridge --- VLAN20
    Bridge --- VLAN30

    %% Storage Path
    Server -.->|NFS/SMB| NAS

    %% Styling
    style Modem fill:#fce4ec,stroke:#880e4f
    style RemoteAdmin fill:#2ecc71,stroke:#333
    style L009 fill:#bbf,stroke:#333
    style WG fill:#3498db,color:#fff
    style Bridge fill:#dfd,stroke:#333,stroke-width:4px
    style VLAN10 fill:#e8f5e9,stroke:#2e7d32
    style VLAN20 fill:#fff3e0,stroke:#ef6c00
    style VLAN30 fill:#ffebee,stroke:#c62828
```

## Services Architecture

```mermaid
graph TB
    subgraph Host [Fedora Server Layer]
        direction TB
        
        Bridge{{"br0 (Virtual Switch)"}}

        subgraph KVM [QEMU/KVM Hypervisor Layer]
            direction LR
            GW[gateway-server Nginx]
            PI[pihole-server DNS]
            MON[monitor-server Grafana]
            JF[jellyfin-server Media]
            K3S[k3s-test Cluster]
        end

        Bridge <--> GW
        Bridge <--> PI
        Bridge <--> MON
        Bridge <--> JF
        Bridge <--> K3S
    end

    GW -->|Proxy Pass| MON
    GW -->|Proxy Pass| JF
    GW -->|Proxy Pass| K3S

    style Host fill:#f5f5f5,stroke:#333
    style Bridge fill:#e1f5fe,stroke:#01579b
    style GW fill:#fff9c4,stroke:#fbc02d
    style PI fill:#ffebee,stroke:#c62828
    style KVM fill:#f3e5f5,stroke:#7b1fa2
```
