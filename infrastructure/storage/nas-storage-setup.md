
# Storage Server Setup

**Server:** Fedora | **Disks:** 2× 20TB NAS Drives | **Architecture:** LUKS → XFS → MergerFS → SnapRAID

## Design Philosophy
Most storage systems (like RAID) are rigid — adding a new disk often requires reformatting everything. This project uses the **"UnRAID" approach**: individual encrypted disks combined into a virtual pool that can grow one disk at a time, with no capacity lost to mirroring.

### The 4 Layers

1. **LUKS Encryption:** Every disk is encrypted at rest. Even if drives are physically stolen, they cannot be read without the keyfile stored on the OS drive.
2. **XFS Filesystem:** The enterprise-standard filesystem for Fedora/RHEL, optimized for large files and multi-terabyte volumes.
3. **MergerFS Pooling:** Merges multiple physical disks into a single `/storage` mountpoint — one virtual 20TB+ pool regardless of how many underlying disks exist.
4. **SnapRAID Parity:** One disk stores parity data to reconstruct any single failed data disk. Unlike RAID, only the changed data needs syncing.

---

## 1. Prerequisites
```bash
sudo dnf install cryptsetup mergerfs snapraid
```

---

## 2. Disk Encryption (LUKS)

### Create Keyfile
```bash
sudo mkdir -p /etc/luks-keys
sudo dd if=/dev/urandom bs=512 count=4 of=/etc/luks-keys/storage_key
sudo chmod 400 /etc/luks-keys/storage_key
```

### Format and Add Keys
```bash
sudo cryptsetup luksFormat /dev/sda
sudo cryptsetup luksFormat /dev/sdb

sudo cryptsetup luksAddKey /dev/sda /etc/luks-keys/storage_key
sudo cryptsetup luksAddKey /dev/sdb /etc/luks-keys/storage_key
```

### Configure Auto-Unlock (`/etc/crypttab`)
Get UUIDs via `lsblk -f`. Add to `/etc/crypttab`:
```text
data1    UUID=YOUR_SDA_UUID    /etc/luks-keys/storage_key    luks
parity1  UUID=YOUR_SDB_UUID    /etc/luks-keys/storage_key    luks
```

---

## 3. Filesystem & Mounting (XFS + MergerFS)

### Open and Format
```bash
sudo cryptsetup open /dev/sda data1 --key-file /etc/luks-keys/storage_key
sudo cryptsetup open /dev/sdb parity1 --key-file /etc/luks-keys/storage_key

sudo mkfs.xfs /dev/mapper/data1
sudo mkfs.xfs /dev/mapper/parity1
```

### Create Mount Points
```bash
sudo mkdir -p /mnt/disk1 /mnt/parity1 /storage
```

### Configure Mounts (`/etc/fstab`)
```text
# Physical Encrypted Drives
/dev/mapper/data1    /mnt/disk1     xfs    defaults,noatime    0 0
/dev/mapper/parity1  /mnt/parity1   xfs    defaults,noatime    0 0

# MergerFS Virtual Pool
/mnt/disk*    /storage    fuse.mergerfs    allow_other,use_ino,cache.files=off,dropcacheonclose=true,category.create=mfs,minfreespace=50G,fsname=mergerfs    0 0
```

---

## 4. Data Protection (SnapRAID)

### Configuration (`/etc/snapraid.conf`)
```text
parity /mnt/parity1/snapraid.parity

content /var/snapraid/snapraid.content
content /mnt/disk1/snapraid.content

data d1 /mnt/disk1/

exclude *.unrecoverable
exclude /tmp/
exclude /lost+found/
```

### Initialize
```bash
sudo mkdir -p /var/snapraid
sudo snapraid sync
```

### Automation (Nightly Sync via Cron)
```text
0 3 * * * /usr/bin/snapraid sync
```

---

## 5. Permissions & SELinux

### Ownership
```bash
sudo chown -R labuser:labuser /storage
sudo chmod -R 775 /storage
```

### SELinux Policy
Allow Docker and KVM VMs to access the MergerFS pool:
```bash
sudo setsebool -P virt_use_fusefs 1
sudo setsebool -P container_use_devices 1
sudo chcon -Rt svirt_sandbox_file_t /mnt/disk1
```

---

## Summary of Usage
- **Storage Path:** All application data lives in `/storage`.
- **Docker Volumes:** Map to subfolders in `/storage` (e.g., `/storage/movies`). Do **not** use the `:Z` SELinux relabeling flag on MergerFS mounts.
- **Expansion:** To add a third disk, encrypt it, mount to `/mnt/disk2`, and update `/etc/snapraid.conf`. MergerFS grows automatically.
