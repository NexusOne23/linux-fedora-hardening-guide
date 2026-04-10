# 🛡️ Linux Hardening Guide for Fedora Workstation — Overview

> Complete documentation of all hardening measures
> 27 documents covering every security layer from UEFI to browser
> Target: Fedora Workstation 43+ | Intel or AMD desktop/workstation
> License: CC BY-SA 4.0

---

## 💻 System Requirements

| Component | Requirement |
|-----------|------------|
| **OS** | Fedora Workstation 43+ |
| **Boot** | UEFI with Secure Boot enabled |
| **Encryption** | LUKS2 full disk encryption (configured during installation) |
| **Filesystem** | Btrfs (Fedora default) with Snapper snapshots |
| **Network** | Ethernet recommended (WiFi has additional attack surface) |
| **VPN** | Optional but strongly recommended (WireGuard-based) |
| **SELinux** | Enforcing (Fedora default — do not disable) |

---

## 📚 Documents

### ⚙️ Kernel & System

| # | Document | Topics |
|---|----------|--------|
| 01 | [Kernel Boot Parameters](01-kernel-boot-params.md) | Boot-level hardening, Secure Boot, lockdown, CPU mitigations, IOMMU |
| 02 | [Sysctl Hardening](02-sysctl-hardening.md) | 64 kernel/network parameters: pointer restriction, ptrace, ASLR, ICMP, ARP, io_uring, user namespaces |
| 21 | [Kernel Module Blacklisting](21-kernel-module-blacklisting.md) | 46 blacklisted modules across 12 categories |
| 22 | [LUKS Encryption](22-luks-encryption.md) | LUKS2 AES-XTS-512, Argon2id, Btrfs layout, mount hardening, header backup, zram swap |

### 🌐 Network & Firewall

| # | Document | Topics |
|---|----------|--------|
| 03 | [Firewall](03-firewall.md) | firewalld with nftables backend, DROP default, custom zones and policies |
| 04 | [ARP Hardening](04-nftables-arp-hardening.md) | Layer 2 protection: static ARP, nftables arp table, dispatcher script |
| 05 | [LAN Isolation](05-lan-isolation.md) | Network invisibility: MAC randomization, LLDP off, multicast suppression |
| 06 | [VPN Killswitch](06-vpn-killswitch.md) | Quadruple killswitch ensuring zero traffic leaks |
| 07 | [IPv6 Hardening](07-ipv6-hardening.md) | Selective IPv6 disable with defense-in-depth |
| 23 | [NetworkManager Hardening](23-networkmanager-hardening.md) | Static IP, MAC randomization, connectivity check disabled |

### 🔑 Services & Authentication

| # | Document | Topics |
|---|----------|--------|
| 08 | [Service Minimization](08-service-minimization.md) | 73 disabled components (53 system + 18 user + 2 D-Bus) |
| 09 | [SSH Hardening](09-ssh-hardening.md) | Key-only authentication, service disabled by default |
| 10 | [PAM & Login Security](10-pam-login-security.md) | Yescrypt hashing, SUID audit, core dumps disabled, umask 027 |
| 11 | [DNS & NTP](11-dns-ntp.md) | DNSSEC, DNS-over-HTTPS, chrony with NTS (4 servers) |

### 📊 Monitoring & Integrity

| # | Document | Topics |
|---|----------|--------|
| 12 | [SELinux & auditd](12-selinux-auditd.md) | Enforcing SELinux + hardened booleans, 38 immutable audit rules |
| 13 | [AIDE](13-aide.md) | Daily file integrity monitoring |

### 🔩 Hardware & Firmware

| # | Document | Topics |
|---|----------|--------|
| 14 | [USBGuard](14-usbguard.md) | Block-by-default USB policy with hash-based identification |
| 15 | [Intel ME Mitigation](15-intel-me-mitigation.md) | Multi-layer mitigation for Intel Management Engine |
| 19 | [GPU & Secure Boot](19-gpu-secureboot.md) | Proprietary GPU drivers with MOK signing |
| 24 | [Firmware Updates](24-firmware-updates.md) | fwupd, LVFS, UEFI dbx management |

### 🖥️ Desktop & Applications

| # | Document | Topics |
|---|----------|--------|
| 16 | [Firefox Hardening](16-firefox-browser.md) | arkenfox user.js, fingerprinting protection, content blocking |
| 17 | [Desktop Stack](17-desktop-stack.md) | GNOME/Wayland hardening, service reduction, privacy settings |
| 18 | [Flatpak Sandboxing](18-flatpak-sandboxing.md) | Minimal Flatpak usage, permission management |
| 20 | [Snapshots](20-snapshots.md) | Btrfs snapshots for rollback capability |

### 🔧 Maintenance

| # | Document | Topics |
|---|----------|--------|
| 25 | [Update Process](25-update-process.md) | Systematic update workflow with integrity verification |
| 26 | [Package Removal](26-package-removal.md) | Attack surface reduction through package cleanup |

---

## 📋 Recommended Implementation Order

1. 🔒 **LUKS** (22) — Must be done during OS installation
2. ⚙️ **Kernel** (01, 02) — Boot parameters and sysctl
3. 🌐 **Network** (03-07) — Firewall, ARP, LAN isolation, VPN, IPv6
4. ⛔ **Services** (08) — Mask unnecessary services
5. 🔑 **Authentication** (09-11) — SSH, PAM, DNS/NTP
6. 📊 **Monitoring** (12-13) — SELinux audit rules, AIDE
7. 🔧 **Hardware** (14-15) — USBGuard, Intel ME
8. 🖥️ **Desktop** (16-18) — Browser, GNOME, Flatpak
9. 🎮 **GPU** (19) — Secure Boot with proprietary drivers
10. 📸 **Snapshots** (20) — Btrfs snapshot configuration
11. ⚙️ **Modules** (21) — Kernel module blacklisting
12. 🌐 **NetworkManager** (23) — Connection hardening
13. 🔧 **Firmware** (24) — Firmware update process

> **Reboot required after:** Kernel parameters (01), module blacklisting (21), auditd immutable mode (12)
>
> **Rebuild AIDE database after every change.**

---

## 📝 Conventions Used in This Guide

| Placeholder | Replace With | How to Find |
|-------------|-------------|-------------|
| `<YOUR-GATEWAY-IP>` | Your router's IP address | `ip route \| grep default` |
| `<YOUR-GATEWAY-MAC>` | Your router's MAC address | `ip neigh show` |
| `<YOUR-INTERFACE>` | Your network interface name | `ip link show` |
| `<YOUR-USERNAME>` | Your Linux username | `whoami` |
| `<YOUR-LUKS-UUID>` | Your LUKS partition UUID | `sudo blkid \| grep crypto_LUKS` |
| `<YOUR-VPN-SERVER-IP>` | Your VPN provider's server IP | VPN provider's documentation |

- Commands prefixed with `$` run as normal user, `#` as root
- Every section includes **Reproduction** steps and **Verification** commands

---

## ⚠️ Important Notes

- **All `sudo` commands must be executed manually** — this documentation is not an automated script
- **Test in a VM first** if you are unsure about any change
- **Create a Snapper snapshot** before making changes: `sudo snapper create -d "before hardening"`
- **AIDE database must be rebuilt** after system changes: `sudo aide --update`
- **auditd immutable mode (-e 2):** Rule changes require a reboot, not just a service restart
- Any local service you install should bind to `127.0.0.1` only, with telemetry and cloud features disabled

---

*Next: [01 — Kernel Boot Parameters](01-kernel-boot-params.md) — Boot-level hardening, Secure Boot, lockdown, CPU mitigations, IOMMU*
