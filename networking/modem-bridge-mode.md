
# ISP Modem to MikroTik: Bridge Mode Migration

**Goal:** Put the ISP modem into bridge mode so the MikroTik L009 owns the public IP directly, enabling WireGuard VPN and port forwarding without double-NAT.

## 1. Physical Topology
- **Modem:** Huawei HG8145X6 (ISP-provided fiber gateway)
- **Router:** MikroTik L009
- **Connection:** Modem **LAN 1** → MikroTik **Ether1**

---

## 2. Modem Configuration

1. Access the installer menu: `http://192.168.1.1/instalador`
2. Navigate to **Advanced Configuration → WAN**.
3. Edit the existing Internet profile (usually `VID_10`).
4. Change **WAN Mode** from `Route WAN` to **`Bridge WAN`**.
5. **Binding:** Check `LAN1` only — this passes the signal to the MikroTik.
6. **SSID Binding:** Uncheck all SSID boxes — modem Wi-Fi should not be tied to the bridge.
7. **Apply** and wait for reboot.

---

## 3. MikroTik Configuration

### A. Disable Old DHCP Client
Prevents the MikroTik from attempting to use the modem's internal network:
```routeros
/ip dhcp-client disable [find interface=ether1]
```

### B. Configure PPPoE Tunnel
```routeros
/interface pppoe-client add name=pppoe-isp \
    interface=ether1 \
    user="<pppoe_username>@<isp.com>" \
    password="<pppoe_password>" \
    add-default-route=yes \
    use-peer-dns=yes \
    disabled=no
```

### C. Configure NAT
```routeros
# Remove old NAT rules
/ip firewall nat remove [find]

# Add masquerade for the PPPoE tunnel
/ip firewall nat add chain=srcnat out-interface=pppoe-isp action=masquerade
```

### D. MTU/MSS Optimization
PPPoE adds headers that increase packet size. Clamping MSS prevents fragmentation and page-loading failures:
```routeros
/interface pppoe-client set pppoe-isp max-mtu=1492 max-mru=1492

/ip firewall mangle add chain=forward protocol=tcp tcp-flags=syn \
    action=change-mss new-mss=clamp-to-pmtu passthrough=yes
```

---

## 4. Verification

| Command | Purpose |
| :--- | :--- |
| `/interface pppoe-client monitor pppoe-isp` | Check status is `connected` and view public IP |
| `/ip route print` | Ensure single `0.0.0.0/0` route via `pppoe-isp` |
| `/ping 8.8.8.8` | Verify raw internet connectivity |
| `/put [:resolve google.com]` | Verify DNS resolution |

---

## 5. Persistence
```routeros
/system backup save name=bridge-mode-config
```

---

## Key Benefits
1. **Direct Public IP:** MikroTik owns the fixed IP — WireGuard and port forwarding work cleanly.
2. **Hardware Offloading:** The MikroTik ARM processor handles PPPoE encryption more efficiently than the modem CPU.
3. **No Double NAT:** Eliminates the latency penalty of two NAT layers for servers and remote access.
