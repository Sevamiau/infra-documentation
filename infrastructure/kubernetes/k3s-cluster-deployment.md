
# Kubernetes (K3s) Cluster Deployment & Management

## 1. Overview
As the infrastructure grew, we begging to try services on a **Kubernetes (K3s)** environment. This new approach enables declarative infrastructure management, self-healing capabilities, and better resource orchestration within the virtualized environment.

## 2. Cluster Architecture
- **Distribution:** K3s (Lightweight Kubernetes by Rancher).
- **Deployment Target:** A dedicated Fedora Cloud VM running on the KVM virtualization host.
- **Network Integration:** Placed within **VLAN 10 (Trusted)** on bridge `br10`.
- **Storage:** Local Path Provisioner using **Persistent Volume Claims (PVC)** to ensure data persistence across pod restarts.

---

## 3. Implementation Process

### Installation & System Hardening
The cluster was initialized using the K3s installation script with the following key modifications:

- **Service Optimization:** Modified the `k3s.service` systemd unit with `--write-kubeconfig-mode 644` to allow non-root users to run `kubectl` without sudo.
- **Environment Configuration:** Configured the `KUBECONFIG` environment variable to point to the cluster authority file.

### Application Deployment: Dashboard
To centralize management of the hybrid infrastructure, a **Homepage Dashboard** was deployed using Kubernetes-native manifests:

- **Declarative Infrastructure:** YAML manifests define **Deployments**, **Services**, and **ConfigMaps**.
- **Storage Persistence:** A PVC maps configuration files (`services.yaml`, `widgets.yaml`) to the VM's underlying storage, preserving the dashboard layout across updates.
- **Self-Healing:** Replica configuration ensures the dashboard remains available even if a pod restarts.

---

## 4. Connectivity & Traffic Flow

Services are exposed to the wider network through a multi-tier approach:

1. **Internal K8s Service:** Applications listen on internal container ports.
2. **NodePort Exposure:** Services are mapped to high-range ports (e.g., `30080`) on the VM's IP.
3. **Reverse Proxy Integration:** The Nginx Reverse Proxy VM acts as the gateway, mapping `http://dashboard.lan` to the K8s NodePort.
4. **VPN Access:** The cluster lives in the Trusted VLAN and is fully accessible to WireGuard VPN users.

---

## 5. Challenges Overcome

### The DNS Bootstrap Problem
**Challenge:** During the initial K3s installation, the VM experienced "DNS blindness" because it could not resolve the installation script host while the Pi-hole VM was undergoing network changes.

**Resolution:** Performed a manual override of `/etc/resolv.conf` using a public upstream resolver (`1.1.1.1`) to bootstrap the installation, then configured NetworkManager permanently once the local DNS sinkhole was stable.

### Browser Cache & Service Identity
**Challenge:** Browser persistence caused the dashboard IP to continue serving the default Nginx welcome page even after the service was updated.

**Resolution:** Used `ss -tulpn` to verify process-level port binding and performed a targeted browser cache flush to force the new application state.
