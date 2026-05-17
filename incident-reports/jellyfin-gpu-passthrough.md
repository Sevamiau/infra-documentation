
# Tech Note: High-Performance Media Passthrough (Virtio-FS) for KVM

## 1. The Challenge
Running a media server (Jellyfin) inside a Fedora VM for isolation, while keeping the actual media files on the Physical Host's disks. Standard network shares (NFS/SMB) introduce latency and overhead unsuitable for high-bitrate 4K video streaming.

---

## 2. The Solution: Virtio-FS
Virtio-FS allows a VM to access the host's filesystem at near-native speeds by sharing memory between the host and guest kernels — no network protocol overhead.

---

## 3. Step 1: Physical Host Configuration
Virtio-FS requires "Shared Memory Backing" in the VM definition.

1. **Edit the VM XML:** `virsh edit jellyfin-server`
2. **Add Memory Backing:** Inside the `<domain>` tag:
    ```xml
    <memoryBacking>
      <access mode='shared'/>
    </memoryBacking>
    ```
3. **Define the Filesystem Device:**
    ```xml
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs'/>
      <source dir='/path/to/your/media'/>
      <target dir='my_media'/>
    </filesystem>
    ```

---

## 4. Step 2: Guest VM Configuration
```bash
sudo mount -t virtiofs my_media ~/jellyfin/media

my_media    /home/labuser/jellyfin/media    virtiofs    defaults    0    0
```

---

## 5. Step 3: SELinux & Docker Volume Labeling
Since both the Host and Guest are **Fedora**, SELinux is active on both. By default, SELinux prevents a Docker container from reading a Virtio-FS mount, even if the VM can see it.

**The Fix:** Use the `:z` SELinux relabeling flag on the media volume mount:
```bash
docker run -d \
  --name jellyfin \
  --net=host \
  -v /home/labuser/jellyfin/config:/config \
  -v /home/labuser/jellyfin/cache:/cache \
  -v /home/labuser/jellyfin/media:/media:z \
  jellyfin/jellyfin
```

The `:z` flag tells Docker to automatically relabel the files with an SELinux context the container can read.

---

## 6. Step 4: CPU Passthrough ("Error 139")
Jellyfin was crashing on startup with `Error 139` (Segmentation Fault).

- **Cause:** The default QEMU virtual CPU model doesn't support advanced instructions (AVX/AES) required by modern media transcoding libraries.
- **Fix:** Set the VM CPU model to `host-passthrough`, which tells the VM to use the exact CPU model and feature set of the physical processor.

---

## 7. Architecture Benefits
1. **Isolation:** If the media server is ever compromised, the attacker is contained in a VM with no direct access to the physical host.
2. **Snapshots:** Full VM state can be snapshotted before risky updates — instant rollback in seconds.
3. **Performance:** Virtio-FS is fast enough to handle multiple concurrent 4K streams without the CPU overhead of network protocols.

---
