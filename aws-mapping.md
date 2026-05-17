
# Mapping On-Premise Infrastructure to AWS Architecture (SAA-C03)

This document explains how the local home lab infrastructure implements architectural patterns from the **AWS Certified Solutions Architect – Associate (SAA-C03)** curriculum.

## 1. Compute & Virtualization
| AWS Concept | Local Implementation | Description |
| :--- | :--- | :--- |
| **Amazon EC2** | **QEMU/KVM Virtual Machines** | Provisioning Linux instances via Ansible and Cloud-Init to simulate on-demand compute resources. |
| **Instance Metadata** | **Cloud-Init Config** | Using Cloud-Init to bootstrap users, SSH keys, and network settings upon "launch." |

## 2. Networking & Content Delivery
| AWS Concept | Local Implementation | Description |
| :--- | :--- | :--- |
| **Application Load Balancer (ALB)** | **Nginx Reverse Proxy** | Managing Layer 7 traffic routing and SSL termination. |
| **Target Groups** | **Nginx `upstream` Blocks** | Defining a group of backend VMs that receive traffic from the proxy. |
| **VPC / Subnets** | **MikroTik VLANs** | Network isolation between "Public" (DMZ), "Private" (Internal Services), and "Management" segments. |
| **Security Groups** | **MikroTik Firewall Rules** | Stateful packet inspection to allow only specific ports (80, 443, 22) to specific targets. |
| **Route 53** | **Pi-hole (Local DNS)** | Internal service discovery and local domain resolution. |

---

## 3. Deep Dive: Load Balancing & High Availability

### Target Group Simulation
In AWS SAA-C03, a **Target Group** routes requests to registered backend instances. This is replicated using the Nginx `upstream` module:

```nginx
# Replicating an AWS Target Group
upstream my_web_target_group {
    server 192.168.10.221;  # Target 01
    server 192.168.10.222;  # Target 02
}
```

### Health Checks (Passive vs. Active)
AWS ALBs use **Active Health Checks** (polling a path like `/health`). Using Nginx Open Source, **Passive Health Checks** are implemented instead:

- **`max_fails`:** Replicates the AWS "Unhealthy Threshold." If a VM fails to respond, it is marked unavailable.
- **`fail_timeout`:** Replicates the interval before Nginx re-includes the target in the pool.

```nginx
server 192.168.10.221 max_fails=3 fail_timeout=30s;
```

---

## 4. Disaster Recovery & Storage
| AWS Concept | Local Implementation | Description |
| :--- | :--- | :--- |
| **Amazon S3** | **Minio (Planned)** | Object storage simulation for application data backups. |
| **EBS Snapshots** | **QCOW2 Snapshots** | Point-in-time snapshots of VM disks before major updates. |
| **AWS Backup** | **Borg/Rsync Scripts** | Automated scripts to move configuration data to off-site storage. |

---

