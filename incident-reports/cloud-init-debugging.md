
# Incident Report: Cloud-Init & SSH Authentication Failure

**Environment:** MikroTik Gateway, Fedora 41 Host (KVM/Libvirt), Ansible Orchestration
**Status:** Resolved

---

## 1. Executive Summary
During the deployment of a Fedora Cloud 41 VM using Ansible, the system was successfully created and assigned an IP via MikroTik DHCP, but SSH access was denied (`Permission denied (publickey)`). The Cloud-Init configuration was not being processed due to SELinux restrictions on manual ISO files and a misunderstanding of the Cloud-Init execution lifecycle.

---

## 2. Symptoms
- **SSH Failure:** `ssh labuser@192.168.10.220` returned `Permission denied`.
- **Console Failure:** `virsh console new-server` showed a login prompt, but the user was not recognized.
- **MikroTik State:** IP was successfully bound in DHCP leases — network connectivity was functional.

---

## 3. Root Cause Analysis

### A. SELinux & Manual ISO Creation
The original deployment used `genisoimage` to create a `cidata.iso` for Cloud-Init. On Fedora hosts, Libvirt and QEMU run under strict SELinux policies. Manual ISO files created in `/tmp` lacked the `virt_content_t` security context, preventing the VM from reading the configuration data on boot.

### B. Cloud-Init Lifecycle (First Boot Only)
Cloud-Init runs **only on the first boot**. Once a VM disk (`.qcow2`) has been initialized, Cloud-Init marks the instance as "configured." Subsequent attempts to fix the config by updating the ISO and restarting the VM are ignored unless the disk is completely deleted and recreated.

### C. Incorrect SSH Key Flag
Initial connection attempts passed the `.pub` file extension to the `-i` flag (`ssh -i id_ed25519.pub`). SSH requires the **private key** for authentication; the public key file is rejected.

---

## 4. Resolution

### Step 1: Migrate to Native `--cloud-init`
The playbook was updated to use `virt-install --cloud-init` instead of manual ISO generation. This ensures:
- Automatic configuration volume creation.
- Correct SELinux labeling (`virt_content_t`) applied by the system.
- Proper injection of `user-data` and `meta-data`.

### Step 2: Console Fallback Password
A hashed password was added to the `user-data` configuration to allow emergency access via `virsh console` when SSH fails.

### Step 3: Clean State Protocol
A strict reset procedure was established to guarantee Cloud-Init always triggers on the next boot:
```bash
virsh destroy <vm>
virsh undefine <vm>
rm /var/lib/libvirt/images/<vm>.qcow2
```

---

## 5. Final Configuration Snippets

### Optimized Ansible Task
```yaml
- name: Define and Start VM (Native Cloud-Init)
  ansible.builtin.shell: >
    virt-install
    --name {{ vm_name }}
    --ram {{ vm_ram }}
    --vcpus {{ vm_cpus }}
    --os-variant fedora-unknown
    --disk path={{ libvirt_pool_dir }}/{{ vm_name }}.qcow2,format=qcow2
    --cloud-init user-data=/tmp/user-data
    --import --noautoconsole
    --network bridge=br10,model=virtio,mac={{ vm_mac }}
    --autostart
```

### Correct SSH Connection
```bash
# Remove stale host key fingerprint
ssh-keygen -R 192.168.10.220

# Connect with the private key (no .pub extension)
ssh -i ~/.ssh/id_ed25519 labuser@192.168.10.220
```

---

## 6. Lessons Learned
1. **Prefer Native Helpers:** Use `virt-install --cloud-init` on Fedora/RHEL to avoid SELinux permission pitfalls with manual ISO creation.
2. **First Boot is Final:** Always delete the existing `.qcow2` disk when testing Cloud-Init changes.
3. **Always Add Console Passwords:** Include a temporary hashed password in Cloud-Init for emergency `virsh console` debugging.
4. **Private vs Public Key:** Ensure the SSH client points to the private key (no extension), not the public key (`.pub`).
