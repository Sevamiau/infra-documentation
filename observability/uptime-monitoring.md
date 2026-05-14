
# Infrastructure Health & Alerting: Uptime Kuma

## 1. Goal
Implement a lightweight, agentless health check system that monitors the availability of every node in the fleet (MikroTik, Fedora Host, all VMs). The system provides a visual dashboard and sends instant push notifications if a service fails.

---

## 2. Architecture: Agentless Approach
**Uptime Kuma** runs on the **Fedora Host** and probes VMs from the outside — no agent required on targets.

- **Host:** Fedora 41
- **Container:** `louislam/uptime-kuma:1`
- **Storage:** Persistent data at `/opt/uptime-kuma` on the host.

---

## 3. Technical Hurdles & Resolutions

### A. SELinux Permission Conflict
- **Issue:** Container crashed with `Permission denied` when writing to the host directory.
- **Cause:** Fedora's SELinux policy blocks Docker containers from accessing host files unless they are specifically labeled.
- **Resolution:** Applied the `:Z` flag in the Ansible volume definition to trigger automatic SELinux relabeling:
    ```yaml
    volumes:
      - /opt/uptime-kuma:/app/data:Z
    ```

### B. Bridge Visibility Issue
- **Issue:** Uptime Kuma (in Docker) could reach the internet but not the KVM VMs on the same host bridge (`br10`).
- **Cause:** Linux bridge filtering (`bridge-nf-call-iptables`) was routing internal bridge traffic through iptables, where it was being dropped.
- **Resolution:** Tuned the kernel to allow bridge traffic to bypass iptables:
    ```bash
    sudo sysctl net.bridge.bridge-nf-call-iptables=0
    ```

### C. Host Firewall Access
- **Issue:** Dashboard was unreachable from the VPN on port 3001.
- **Resolution:** Opened the port permanently via Firewalld:
    ```bash
    sudo firewall-cmd --permanent --add-port=3001/tcp && sudo firewall-cmd --reload
    ```

---

## 4. Monitoring Configuration

### Service Check Strategy
**TCP Port Checks** are used rather than simple pings. This ensures that not only is the VM "on," but that the OS and management services are actively responsive.

- **Port 22 (SSH):** Monitored on all VMs.
- **Port 80/443:** Reserved for web applications.

### Alerting Pipeline
**Discord Webhooks** serve as the notification engine.

- **Trigger:** Service down for > 60 seconds.
- **Payload:** Discord message with server name and exact error (e.g., `Request Timeout`).

---

## 5. DevOps Value
- **Single-pane observability** across the entire fleet.
- **Decoupled resilience:** The monitoring system can report failures even if the entire VM network segment is down.
- **ChatOps pattern:** Discord alerts demonstrate integration of infrastructure monitoring with team communication tooling.
