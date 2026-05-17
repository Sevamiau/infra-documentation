# Incident Report: Phase 6 — Perimeter Hardening & Zero-Trust Ingress

**Architecture:** Nginx Gateway → Firewalld Rich Rules → Backend Services

---

## 1. The Challenge
The initial observability stack was functional but insecure. Management ports for Grafana (`3000`), Prometheus (`9090`), and Loki (`3100`) were open to the entire VPN subnet. Any VPN user could bypass the Nginx Gateway and hit management interfaces directly via the Monitor Server's IP.

---

## 2. Root Cause Analysis

1. **Standard Port Rules:** `firewalld` was configured with standard port rules (e.g., `3000/tcp`). A standard port rule is global — it allows traffic from any source IP, regardless of other restrictions.
2. **Rule Salad:** Overlapping `REJECT` and `ACCEPT` rules were conflicting. Because global port rules were open, more restrictive Rich Rules were being ignored by the kernel's rule processing order.

---

## 3. Resolution

### Step A: Reset to Default Deny
The firewall configuration on the Monitor Server was completely reset. All global port rules were removed to establish a **"Default Deny"** posture.

### Step B: Implement Source-Based Rich Rules
Transitioned to a **Source-Based Access Control** model. Instead of opening ports to the world, Rich Rules specifically allow only the Gateway VM's IP.

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.210" port port="3000" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.210" port port="9090" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.210" port port="3100" protocol="tcp" accept'
firewall-cmd --reload
```

### Step C: Nginx as Key-Holder
The Nginx Gateway handles all incoming user traffic on Port 80. Using **Host Headers** (`grafana.lab`), Nginx acts as the authorized intermediary that passes through the Monitor Server's firewall.

---

## 4. Tiered Verification

| Test Type | Source | Target | Command | Result |
| :--- | :--- | :--- | :--- | :--- |
| Negative (Direct) | Laptop | Monitor:3000 | `curl -I 192.168.10.200:3000` | Connection Refused ✅ |
| Positive (Proxy) | Laptop | `grafana.lab` | Browser | 200 OK ✅ |
| Internal (Trust) | Gateway | Monitor:3000 | `curl -I 192.168.10.200:3000` | 200 OK ✅ |

---

