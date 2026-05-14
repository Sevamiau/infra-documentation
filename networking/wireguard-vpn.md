
# WireGuard VPN: Remote Access Documentation

## 1. Network Architecture

| Device | VPN IP | Role |
| :--- | :--- | :--- |
| MikroTik Router | `10.0.0.1` | VPN Hub |
| Fedora Server | `10.0.0.2` | Jump host / target |
| Laptop | `10.0.0.3` | Remote client |
| Workstation | `10.0.0.4` | Remote client |

- **VPN Hub:** MikroTik (Fixed Public IP)
- **Internal VPN Network:** `10.0.0.0/24`
- **DNS (Pi-hole):** `192.168.10.53` (accessible via VPN)

---

## 2. Client Configuration
The configuration is stored in `/etc/wireguard/wg0.conf`.

**Key Parameters:**
- **Endpoint:** `<YOUR_PUBLIC_IP>:51280` — the public entry point.
- **AllowedIPs:** `10.0.0.0/24, 192.168.10.0/24` — routes both VPN and Trusted VLAN traffic through the tunnel.
- **DNS:** `192.168.10.53` — forces the client to use Pi-hole even when remote.

---

## 3. Daily Usage

```bash
# Start the VPN
sudo wg-quick up wg0

# Stop the VPN
sudo wg-quick down wg0

# Check connection status
sudo wg show
```

**Success indicator:** `latest handshake` timestamp is present and `transfer` bytes are incrementing.

---

## 4. Accessing Services via VPN

| Service | Address |
| :--- | :--- |
| Fedora SSH | `ssh labuser@10.0.0.2` |
| MikroTik Admin | WinBox → `10.0.0.1` |
| Pi-hole Web | `http://192.168.10.53/admin` |
| Grafana | `http://grafana.lab` |

---

## 5. Adding a New Device (Spoke)

### Phase 1: Generate Identity
```bash
# Linux
wg genkey | tee privatekey | wg pubkey > publickey
```
For Windows/Android/iOS: use the WireGuard app → **Add Tunnel → Create from scratch**.

### Phase 2: Register on MikroTik
Assign the next available IP in the `10.0.0.0/24` range (e.g., `10.0.0.6`) and add the peer:
```mikrotik
/interface wireguard peers
add interface=wg-vpn public-key="<NEW_DEVICE_PUBLIC_KEY>" allowed-address=10.0.0.6/32 comment="Device Name"
```

### Phase 3: Client Config Template
```ini
[Interface]
PrivateKey = <NEW_DEVICE_PRIVATE_KEY>
Address = 10.0.0.6/24
DNS = 192.168.10.53
MTU = 1420

[Peer]
PublicKey = <MIKROTIK_WG_PUBLIC_KEY>
Endpoint = <YOUR_PUBLIC_IP>:51280
AllowedIPs = 10.0.0.0/24, 192.168.10.0/24
PersistentKeepalive = 25
```

### Phase 4: Firewall Trust (Fedora Host)
Since the full `10.0.0.0/24` range is already in the trusted zone, new devices are usually authorized automatically. For a service with restricted access:
```bash
sudo firewall-cmd --permanent --zone=trusted --add-source=10.0.0.6/32
sudo firewall-cmd --reload
```

### Phase 5: Verification Checklist
1. Run `sudo wg show` — confirm `Latest Handshake` is present.
2. Ping `10.0.0.1` (MikroTik) and `10.0.0.2` (Fedora Host).
3. Open `http://grafana.lab` — if it loads, DNS via Pi-hole is working.

---

## 6. Troubleshooting

| Symptom | Likely Cause | Resolution |
| :--- | :--- | :--- |
| Handshake works, can't ping Fedora | WireGuard not running on server | `sudo systemctl status wg-quick@wg0` |
| Handshake fails entirely | Public IP changed or firewall issue | Verify UDP `51280` is open on MikroTik input chain |
| `.lab` domains don't resolve | VPN DNS priority conflict | Add entries to `/etc/hosts` on the client |

---

## Pro-Tip: Auto-Start on Reboot
```bash
sudo systemctl enable wg-quick@wg0
```
