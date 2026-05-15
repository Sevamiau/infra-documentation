# Home Lab Observability Documentation: Loki & Alloy

## 1. Overview
This system provides centralized logging for the entire Fedora/QEMU infrastructure. It allows for real-time log analysis and historical searching across all Virtual Machines and the physical Host without needing to SSH into individual machines.

![Grafana test](grafana-test.png)

### Architecture
*   **Visualizer:** Grafana (Port 3000)
*   **Collector (The Bucket):** Grafana Loki (Port 3100)
*   **Agent (The Pusher):** Grafana Alloy (installed on every node)

---

## 2. Infrastructure Map
| Role | IP Address | OS | Agent |
| :--- | :--- | :--- | :--- |
| **Monitor Server** | `192.168.10.200` | Fedora (VM) | Loki (Docker), Alloy |
| **Fedora Host** | `192.168.10.107` | Fedora (Physical) | Alloy |
| **Gateway** | `192.168.10.210` | Fedora (VM) | Alloy |
| **Pi-hole** | `192.168.10.53` | Fedora/Debian (VM) | Alloy |
| **Jellyfin** | `192.168.10.219` | Fedora (VM) | Alloy |
| **K3s Node** | `192.168.10.60` | Fedora (VM) | Alloy |

---

## 3. Server Configuration (Monitor Server)
Loki runs as a Docker container on the Monitor Server.

*   **Port:** `3100`
*   **Firewall:** Must allow incoming TCP on `3100` from the `192.168.10.0/24` subnet.
    *   *Command:* `sudo firewall-cmd --permanent --add-port=3100/tcp && sudo firewall-cmd --reload`

---

## 4. Agent Configuration (Alloy)
**Grafana Alloy** is the modern replacement for Promtail. It is installed on every VM to scrape system logs.

### Installation (Fedora/RHEL)
```bash
sudo dnf install alloy -y
sudo usermod -aG systemd-journal alloy
```

### Configuration File (`/etc/alloy/config.alloy`)
Each node uses this standard template. The `host` label is customized per machine.

```river
// 1. Destination
loki.write "local_loki" {
  endpoint {
    url = "http://192.168.10.200:3100/loki/api/v1/push"
  }
}

// 2. Systemd Journal Scraper
loki.source.journal "read_journal" {
  forward_to = [loki.write.local_loki.receiver]
  labels     = { job = "systemd-journal", host = "INSERT_HOSTNAME_HERE" }
}

// 3. Optional: File Scraper (Example for Nginx or Pi-hole)
loki.source.file "app_logs" {
  targets = [
    { __address__ = "localhost", __path__ = "/var/log/nginx/*.log", job = "nginx", host = "INSERT_HOSTNAME_HERE" },
  ]
  forward_to = [loki.write.local_loki.receiver]
}
```

### Service Management
*   **Restart:** `sudo systemctl restart alloy`
*   **Status:** `sudo systemctl status alloy`
*   **Internal Debug UI:** `http://<VM_IP>:12345`

---

## 5. Usage in Grafana
Logs are queried using **LogQL** in the **Explore** tab.

### Basic Queries
*   **See all system logs:** `{job="systemd-journal"}`
*   **See logs for a specific VM:** `{host="gateway"}`
*   **Search for specific text:** `{host="pihole"} |= "error"`
*   **Monitor SSH attempts:** `{job="systemd-journal"} |= "ssh"`

---

## 6. Maintenance & Troubleshooting
1.  **"No Data" in Grafana:** 
    *   Check if Alloy is running: `systemctl status alloy`.
    *   Verify connectivity: `curl http://192.168.10.200:3100/ready` from the client VM.
2.  **Permission Denied:** Ensure the `alloy` user is in the `systemd-journal` and `adm` groups.
3.  **Loki Ingester Not Ready:** Restart the Loki container on the monitor server: `docker restart loki`.

---
**Date Documented:** May 9, 2026
**Status:** Operational (6/6 Nodes)
