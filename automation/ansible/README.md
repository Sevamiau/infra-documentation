
# Ansible Automation

All infrastructure is managed as code. Playbooks cover the full VM lifecycle — from provisioning and service deployment to firewall hardening.

## Setup

A pinned Python version is used intentionally to avoid compatibility breakage from future interpreter updates.

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Copy the example inventory and fill in your IPs:
```bash
cp inventory.example.ini inventory.ini
```

## Playbooks

| Playbook | Purpose |
| :--- | :--- |
| `deploy-vm.yml` | Provision a generic KVM VM via Cloud-Init |
| `deploy-lvm-vm.yml` | Provision a VM with an attached data disk (K3s/storage workloads) |
| `deploy-pihole.yml` | Deploy Pi-hole VM with Docker |
| `deploy-plg-stack.yml` | Deploy Prometheus, Loki, Grafana + Promtail agents on all targets |
| `deploy-monitoring.yml` | Deploy Uptime Kuma with bridge networking and SELinux fixes |
| `deploy-reverse-proxy.yml` | Configure Nginx gateway and harden backend firewall rules |

## Usage

```bash
# Deploy a VM (prompts for sudo password)
ansible-playbook -i inventory.ini playbooks/deploy-vm.yml -K

# Deploy the full observability stack
ansible-playbook -i inventory.ini playbooks/deploy-plg-stack.yml -K

# Configure Nginx reverse proxy and harden backends
ansible-playbook -i inventory.ini playbooks/deploy-reverse-proxy.yml -K
```

## Variable Management
Sensitive variables (passwords, keys) should be stored in an **Ansible Vault** file rather than hardcoded in playbooks:

```bash
ansible-vault create group_vars/all/vault.yml
```

Example vault content:
```yaml
vault_admin_password: "<your_password>"
vault_pihole_webpassword: "<your_pihole_password>"
vault_grafana_admin_password: "<your_grafana_password>"
ssh_public_key: "<your_ssh_public_key>"
```
