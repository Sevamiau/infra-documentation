
# Project Roadmap

## Phase 1: Foundation & Networking (Completed)
- [x] Initial Fedora Server deployment.
- [x] MikroTik RouterOS integration (VLANs, Bridge, Firewall).
- [x] WireGuard VPN setup for secure remote management.
- [x] Static IP schema design for lab devices.

## Phase 2: Automation & Virtualization (Completed)
- [x] QEMU/KVM setup on Fedora.
- [x] Ansible playbook development for "One-Touch" VM creation.
- [x] Docker containerization for core services (Pi-hole, Uptime Kuma).
- [x] UPS (CyberPower) integration with automated shutdown scripts.

## Phase 3: Observability & Persistence (In Progress)
- [ ] **PLG Stack Optimization:** Fine-tuning Loki log retention policies.
- [ ] **Automated Backups:** Implementing off-site backups for VM images and Docker volumes to S3 or NAS.
- [ ] **SSL Management:** Implementing Let's Encrypt with wildcard certificates and automated renewal.

## Phase 4: Security Hardening & Scaling (Planned)
- [x] **Network Segregation:** Moving IoT devices to a dedicated, restricted VLAN.
- [ ] **CI/CD Integration:** Automatically trigger Ansible playbooks when this GitHub repo is updated.
- [ ] **Intrusion Detection:** Deploying CrowdSec or Fail2Ban across the Fedora host and VMs.
- [ ] **Kubernetes Migration:** Transitioning Docker Compose services to a lightweight K3s cluster.
