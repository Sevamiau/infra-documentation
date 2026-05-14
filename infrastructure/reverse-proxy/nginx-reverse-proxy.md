
# Infrastructure Security: Nginx Reverse Proxy & Perimeter Hardening

**Stack:** Nginx, Firewalld (Rich Rules), SELinux, Pi-hole/Local DNS

---

## 1. The Strategy
To move from "Development" to "Production" security standards, a **Gateway Architecture** was implemented. The goal: hide all internal management ports (3000, 9090, 3100, 3001) and provide a single, clean entry point for all web-based services.

---

## 2. Implementation Summary

### A. Reverse Proxy
An **Nginx** server was deployed on a dedicated VM (`192.168.10.210`).

- **Virtual Hosting:** Configured Nginx to use **Host Headers**, allowing one IP to serve multiple domains (`grafana.lab`, `prometheus.lab`, `uptime.lab`).
- **SELinux Integration:** Enabled the `httpd_can_network_connect` boolean to allow the Nginx process to proxy traffic across the virtual network.

### B. Network Hardening
Once the proxy was stable, backend ports were locked down:

- **Rich Rules:** Applied **Firewalld Rich Rules** to the Monitor Server. Traffic on ports 3000, 9090, and 3100 is rejected unless the source IP is the Gateway VM (`192.168.10.210`).
- **Port Abstraction:** Successfully disabled public access to internal ports, reducing the attack surface of the monitoring stack.

---

## 3. Challenges & Resolutions

### I. The "VPN-DNS" Conflict
- **Issue:** After updating Pi-hole, the management laptop could not resolve `.lab` domains due to the VPN's DNS priority.
- **Resolution:** Implemented a local override in `/etc/hosts` on the management laptop, ensuring deterministic routing to the Gateway VM.

### II. WebSocket Instability
- **Issue:** Uptime Kuma dashboard would frequently disconnect.
- **Resolution:** Updated Nginx configuration with `Upgrade` and `Connection` headers to support **WebSockets**, ensuring real-time telemetry remained active.

---

## 4. Final Architecture View
```
User → http://grafana.lab (Port 80)
     → Gateway (192.168.10.210) → forwards → Monitor (192.168.10.200:3000)
                                 → Monitor verifies source IP → serves dashboard
```

---

