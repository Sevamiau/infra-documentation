
# VM Provisioning: Fedora KVM Host & Pi-hole

## 1. System Architecture
This setup uses a physical machine running **Fedora Server** as the hypervisor. Automation deploys a **Fedora Cloud VM** on this host, which runs **Pi-hole** inside a **Docker** container.

- **Physical Server:** `192.168.10.107` (OS: Fedora Server)
- **Virtual Machine:** Internal NAT (`192.168.122.0/24`)
- **Application:** Pi-hole (Docker Container)

---

## 2. Deployment Command
To deploy or update the entire stack from a management workstation:

```bash
ansible-playbook -i inventory.ini deploy-pihole.yml -K
```

- `-i inventory.ini`: Uses the local inventory file.
- `-K`: Prompts for the `sudo` password of the physical Fedora Server.

---

## 3. Remote Access (SSH)
Because the VM lives on a private internal network (`192.168.122.0/24`) inside the physical server, a **ProxyJump** is required to reach it from a remote client.

### Connection Command
```bash
ssh -J <admin_user>@192.168.10.107 labuser@<VM_IP>
```

---

## 4. Web Dashboard Access
To access the Pi-hole web interface from a remote browser:

1. **Create an SSH Tunnel:**
    ```bash
    ssh -L 8080:<VM_IP>:80 <admin_user>@192.168.10.107
    ```
2. **Open Browser:** Visit `http://localhost:8080/admin`

---

## 5. Maintenance Reference

### Reset Pi-hole Password
If the Web UI password is lost, run this **inside the VM**:
```bash
sudo docker exec -it pihole /bin/bash
pihole setpassword
```

### VM Management (On Fedora Server)
- **List VMs:** `sudo virsh list --all`
- **Start VM:** `sudo virsh start pihole-server`
- **Stop VM:** `sudo virsh destroy pihole-server`
- **Emergency Console:** `sudo virsh console pihole-server` (Exit: `Ctrl + ]`)

### Container Management (Inside VM)
- **Logs:** `cd /opt/pihole && sudo docker compose logs -f`
- **Restart:** `sudo docker restart pihole`

---

## 6. Full Cleanup (Before Re-deploying)
If you need to wipe the VM and start fresh, run these on the **Physical Server**:
```bash
sudo virsh destroy pihole-server && sudo virsh undefine pihole-server
sudo rm /var/lib/libvirt/images/pihole-server*
```
