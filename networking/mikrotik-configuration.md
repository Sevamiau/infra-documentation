
# MikroTik RouterOS Configuration Reference

A command reference for initial setup and ongoing maintenance of the MikroTik router.

---

## 1. Security & Identity

```routeros
/system identity set name="<router_name>"
/user set admin password="<your_password>"
```

---

## 2. System Updates

### Update RouterOS
```routeros
/system package update check-for-updates
/system package update install
```

### Update RouterBOARD Firmware
Run **after** the OS update reboot:
```routeros
/system routerboard upgrade
/system reboot
```

---

## 3. Internet (WAN) & Firewall NAT

### Get IP from Modem (DHCP mode)
```routeros
/ip dhcp-client add interface=ether1 disabled=no
```

### NAT Masquerade (Hide Internal Traffic Behind WAN IP)
```routeros
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

---

## 4. Bridge (Local Network)

### Create Bridge
```routeros
/interface bridge add name=bridge
```

### Add Ethernet Ports and Wi-Fi to Bridge
```routeros
/interface bridge port add bridge=bridge interface=ether2
/interface bridge port add bridge=bridge interface=wifi1
```

---

## 5. Wi-Fi (AX / Wi-Fi 6) Configuration

### Set Country and SSID
```routeros
/interface wifi set wifi1 configuration.ssid="<YOUR_SSID>" configuration.country=<YOUR_COUNTRY>
```

### Set WPA2 Security
```routeros
/interface wifi security set [find] authentication-types=wpa2-psk passphrase="<your_password>"
```

---

## 6. DNS Configuration

### Set Upstream DNS Servers
```routeros
/ip dns set servers=192.168.10.53,8.8.8.8 allow-remote-requests=yes
```

### Add Local Domain Names (Reverse Proxy)
```routeros
/ip dns static add name=grafana.lab address=192.168.10.210
/ip dns static add name=prometheus.lab address=192.168.10.210
/ip dns static add name=uptime.lab address=192.168.10.210
```

### Push Pi-hole to All DHCP Clients
```routeros
/ip dhcp-server network set [find] dns-server=192.168.10.53
```

---

## 7. Maintenance Commands

### View Active DHCP Leases
```routeros
/ip dhcp-server lease print
```

### View Connected Wi-Fi Clients
```routeros
/interface wifi registration-table print
```

### Create Backup
```routeros
/system backup save name=<backup_name>
/export file=<backup_name>
```
