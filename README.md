# Proxmox Home Lab: HP ProDesk 400 G5 Media & Infrastructure

A high-efficiency Home Lab deployment focused on Proxmox VE optimization, ZFS storage management, and secure service delivery. This project documents the transition from bare-metal hardware to a fully virtualized environment hosting Immich, Jellyfin, and secure networking via Tailscale.

---

## 📊 Technical Specifications

| Component       | Details                                     |
| :-------------- | :------------------------------------------ |
| **Host** | HP ProDesk 400 G5                           |
| **CPU** | Intel Core (8th Gen)                    |
| **RAM** | 8GB DDR4 (Resource-optimized configuration) |
| **Boot/Apps** | NVMe SSD                                    |
| **Storage Pool**| 8TB HDD (ZFS Mirroring ready)               |
| **Networking** | Tailscale Mesh VPN / Subnet Routing         |

### Software Stack
* **Hypervisor:** Proxmox VE 9.1
* **Containers:** LXC & Docker-in-LXC
* **Media:** Immich (Photo Management), Jellyfin (Media Server)
* **Storage:** ZFS (LZ4 Compression enabled)
* **Security:** Fail2Ban, Unprivileged Containers, Keyctl

---

## 🛠️ System Hardening & BIOS Optimization

Before installing Proxmox, the hardware was sanitised to ensure stability:

* **Secure Erase:** Hardware-level NAND flush of the SSD. This destroys the partition map, ensuring Proxmox sees the drive as raw, unallocated space, preventing metadata conflicts during the ZFS/GPT initialization.
* **UEFI Implementation:** Disabled **Secure Boot** and **Legacy Support**. While Proxmox 9 supports Secure Boot, disabling it during the initial lab setup ensures the Linux kernel and non-signed drivers load without signature verification errors.
* **Fast Boot Disabled:** Ensures full initialization of USB controllers and UEFI devices during POST, preventing the "frozen keyboard" issue often encountered during OS installation.

---

## 🖥️ Storage & Virtualization Strategy

### ZFS Dataset Architecture
Instead of standard directories, I utilized **ZFS Datasets**.
* **Command:** `zfs create hdd-storage/media`
* **Benefit:** This treats the folder as a standalone file system. It allows for granular snapshots, individual quotas (e.g., stopping Media from filling the entire 8TB drive), and dataset-specific compression (LZ4).
<img width="996" height="515" alt="create-zfs" src="https://github.com/user-attachments/assets/06dc41fa-b7e0-4a13-a2db-2777e4e391b7" />

## 🐋 Docker-in-LXC Configuration
To run **Immich**, I opted for a Docker-enabled LXC. This provides near-native performance compared to a VM. 
**Critical Feature Flags Enabled:**
* **FUSE & Nesting:** Required for Docker to manage image layers efficiently.
* **Keyctl:** Mandatory for the Docker daemon to use the Linux kernel keyring for secret management.
<img width="685" height="709" alt="final-settings" src="https://github.com/user-attachments/assets/f01f2a08-0a68-4a8e-ab0e-30938b776640" />

---

## 🔒 Networking & Security
Tailscale Subnet Routing

Instead of opening ports (Port Forwarding), I deployed Tailscale and enabled IP Forwarding.

Subnet Router: Configured the ProDesk to advertise the 192.168.x.x/24 route.

Benefit: I can access the Proxmox GUI and Immich dashboard from anywhere in the world as if I were on my home Wi-Fi, using encrypted WireGuard tunnels.

**Hardening with Fail2Ban**

* Installed fail2ban on the host to monitor SSH and Web GUI login attempts.

    Optimization: Added specific local & tailnet management IPs to the ignoreip list to prevent accidental lockout while maintaining a zero-tolerance policy for external brute-force attempts.

---

## 🛡️ Technical Deep Dive: Troubleshooting & Root Cause Analysis

This section outlines the primary technical challenge encountered during the deployment: 

### Issue 1 
**UID Mapping in Unprivileged Containers.**

**The Problem**
After mounting the 8TB HDD to the Immich container, the services (PostgreSQL/Redis) failed to start with a `Permission Denied` error.
<img width="1267" height="155" alt="permission-denied" src="https://github.com/user-attachments/assets/360a49e8-75c9-477e-84e9-7720b011d00c" />

**The Root Cause: Sub-UID Mapping**
In Proxmox, an **Unprivileged Container** is a major security feature. It ensures that the root user inside the container is not the same as the root user on the host. 

By default, Proxmox maps the container's UIDs to a range starting at **100,000** on the host. 
* **Container Root (UID 0)** = **Host User (UID 100,000)**.
* Even though the container saw itself as `root`, the physical 8TB drive saw a random user (ID 100,000) trying to access its files.

**The Fix**
To resolve this without compromising the "Unprivileged" security boundary, I performed a recursive ownership change on the host level to match the mapped UID range:

```bash
# On the Proxmox Host Shell:
# Change ownership to the mapped LXC root (100000)
chown -R 100000:100000 /mnt/pve/hdd-storage/media

# Set permissions: Owner (rwx), Group/Others (rx)
chmod -R 755 /mnt/pve/hdd-storage/media
```
<img width="667" height="44" alt="chown-chmod" src="https://github.com/user-attachments/assets/6e10e677-d930-4c25-8576-7a1e78e87cc7" />

Why this matters: This fix maintains the security "sandbox." Even if an attacker gains root access inside the Immich container, they are still just a "nobody" (UID 100,000) on the actual host, preventing them from accessing host-level system files.

### Issue 2
**The "Locked Door" (Fail2Ban Misconfiguration)**

The Problem: While hardening the Proxmox Web GUI with Fail2Ban, I configured an overly restrictive jail policy. I inadvertently triggered a ban on my own management workstation, losing all remote access to the dashboard and SSH.

The Recovery:
Because remote access was revoked, I had to utilize physical console access.

    Connected a monitor and keyboard directly to the HP ProDesk.

    Logged into the local Proxmox host shell.

    Used fail2ban-client set pve-jail unbanip <My_IP> to restore immediate access.

    Modified the /etc/fail2ban/jail.local configuration.

**The Fix:**
I updated the ignoreip directive to include the IP's of devices in the tailscale network. This ensures that as long as I am connected via my secure VPN mesh, I am recognized as a "Known-Good" user. It also prevents any device which enters my network from automatically gaining access (though if they are inside the tailscale network I may have bigger issues.
Bash

Key Lesson Learned:
This highlighted the importance of Staging vs. Production. Moving forward, I implement a "Testing Window" for important configuration changes.

🚀 Roadmap

    [ ] Samba AD/DC: Transitioning to centralized identity management.

    [ ] Snapshot Automation: Automating ZFS snapshots to the 8TB drive before any major config changes.
    
    [ ] Hardware Expansion: Upgrading RAM to 16GB to support Windows Server VMs.
