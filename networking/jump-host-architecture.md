
# Jump Host Architecture: Fedora Server as Bastion

**Date:** April 2024
**Status:** Successfully Implemented
**Architecture:** Laptop (Controller) → Fedora Host (Bastion/Jump) → MikroTik (Target)

---

## 1. The Challenge
The initial setup required each management machine to be whitelisted directly on the MikroTik router. This created a security risk (broad access) and a management headache (tracking multiple laptop IPs). The goal was to centralize trust on the **Fedora Host**, so the MikroTik only needs to trust one static server.

---

## 2. The Hurdles & Solutions

### A. Firewall Conflict ("Connection Timed Out")
- **Issue:** After removing the laptop's whitelist, the Fedora Server (`10.0.0.2`) couldn't reach the MikroTik.
- **Cause:** The MikroTik's `input` chain was dropping packets from the VPN subnet.
- **Resolution:** Added a high-priority firewall rule to allow the Fedora Server:
    ```routeros
    /ip firewall filter add chain=input src-address=10.0.0.2 action=accept place-before=0
    ```

### B. Missing SSH Identity ("No Identities")
- **Issue:** The Fedora Server couldn't execute Ansible tasks because it had no SSH key of its own.
- **Resolution:** Generated an Ed25519 key pair on the Fedora Host:
    ```bash
    ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
    ```

### C. MikroTik SSH Key Import
- **Issue:** Standard `ssh-copy-id` failed because MikroTik uses a proprietary key database rather than `authorized_keys` text files.
- **Resolution:** Manual key import via the MikroTik CLI:
    1. `scp` the public key to the router.
    2. Import using the MikroTik command:
        ```routeros
        /user ssh-keys import public-key-file=mykey.pub user=admin
        ```

### D. Incorrect Sudo Password
- **Issue:** Ansible failed at the "become" stage.
- **Resolution:** Clarified that the `-K` flag requires the **target host user's** sudo password, not the local workstation password.

---

## 3. Final Architecture

| Role | Device | Responsibility |
| :--- | :--- | :--- |
| **Controller** | Laptop | Holds playbooks and inventory. No direct MikroTik access. |
| **Bastion/Jump Host** | Fedora Server (`10.0.0.2`) | Only device whitelisted to reach MikroTik's management port (22). Hosts VMs. |
| **Target** | MikroTik | Receives configuration commands only from the Fedora Server. |

---

## 4. Lessons Learned
1. **Bastion Logic:** When running commands on a remote router, always execute from the server closest to that router using `hosts: servers`.
2. **MikroTik Keys:** Never use `ssh-copy-id` on a router — always use the `/user ssh-keys import` workflow.
3. **Non-Interactive SSH:** Before running an Ansible playbook that uses SSH as a sub-tool, test with `-o BatchMode=yes` on the jump host to confirm no password prompts will interrupt automation.
