# Incident Report: SSH Identity Management & Docker Firewall Bypass

## 1. Incident 1: The "New Machine" Lockout

### Symptom
When moving management tasks from one laptop to another, Ansible returned `Permission denied (publickey,...)` and marked all VMs as `UNREACHABLE`, despite the playbook being syntactically correct.

### Root Cause
SSH authentication is based on a **Trust Handshake** — VMs are configured to trust a specific cryptographic identity (public key). When a new laptop attempted to connect, the VMs did not recognize the new identity and rejected the connection before a password could even be attempted.

### Resolution
A manual "identity handshake" was performed to add the new machine's public key to the fleet:

1. **Unlock Key:** Used `ssh-agent` and `ssh-add` to handle the private key passphrase interactively.
2. **Push Identity:** Used `ssh-copy-id labuser@<vm-ip>` to append the new public key to `~/.ssh/authorized_keys` on each target.
3. **Verification:** Confirmed manual SSH access succeeded before resuming Ansible automation.

---

## 2. Incident 2: The Docker Firewall Bypass

### Symptom
Security testing via `curl` from a non-whitelisted machine successfully reached Grafana (Port 3000) directly, **even though** `firewalld` rich rules were configured to only allow the Gateway IP.

### Root Cause
By default, Docker uses bridge networking and manipulates `iptables` directly. When a port is "published" (`-p 3000:3000`), Docker creates a high-priority NAT rule that intercepts traffic **before** it reaches `firewalld`'s filtering layer. This effectively creates a backdoor that bypasses the host's security policy.

### Resolution
The architecture was shifted from Docker Bridge Mode to **Host Networking Mode** for all monitoring containers.

**Change in Ansible playbook:**
```yaml
published_ports:
  - "3000:3000"

network_mode: host
```

**Effect:** Containers now share the Host's network stack directly. All traffic must pass through `firewalld` Rich Rules before reaching any container process.

---

## 3. Security Verification

Three tests confirmed the hardened environment:

| Test | Source | Target | Command | Result |
| :--- | :--- | :--- | :--- | :--- |
| Direct attack | Laptop | Monitor:3000 | `curl -I --connect-timeout 5 http://192.168.10.200:3000` | `Connection timed out` ✅ |
| Trusted path | Laptop | `grafana.lab` | Browser | `200 OK` ✅ |
| Port scan | Laptop | Monitor:3000 | `nmap -p 3000 192.168.10.200` | `State: Filtered` ✅ |

---
