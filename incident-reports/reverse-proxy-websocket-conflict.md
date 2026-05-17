# Incident Report: Uptime Kuma Reverse Proxy Failure

**Systems:** Fedora Host, Nginx Gateway VM, Docker (Uptime Kuma)

---

## 1. Scenario

- **Physical Host:** `192.168.10.107` — Running Uptime Kuma in Docker on port `3001`.
- **Nginx VM (gateway-server):** `192.168.10.210` — Entry point for `uptime.lab`.
- **Goal:** Route traffic from the Gateway VM back to the Uptime Kuma container on the Host.

---

## 2. Issues 

1. **502 Bad Gateway** — Nginx was running but couldn't reach the upstream (the host).
2. **504 Gateway Timeout** — The connection was being silently dropped by the host firewall.
3. **WebSocket Disconnects** — Dashboard would frequently lose real-time connection after initial load.

---

## 3. Root Causes & Resolutions

### Host Firewall Blocking Upstream Traffic
- **Cause:** `firewalld` on the host was not configured to allow traffic on port `3001` from the local bridge network.
- **Fix (on Host):**
    ```bash
    sudo firewall-cmd --permanent --add-port=3001/tcp
    sudo firewall-cmd --reload
    ```

### WebSocket Headers Missing in Nginx
- **Cause:** Uptime Kuma uses WebSockets for real-time status updates. By default, Nginx does not forward the `Upgrade` and `Connection` headers required to establish a WebSocket connection, causing the dashboard to fall back to polling and frequently disconnect.
- **Fix (in Nginx config):**
    ```nginx
    server {
        listen 80;
        server_name uptime.lab;
        location / {
            proxy_pass http://192.168.10.107:3001;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            # Required for WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
    ```

---

