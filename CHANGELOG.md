# Changelog

All notable changes to this guide will be documented in this file.

---

## [1.0] — 2026-04-07

### Initial Release — 27 Documents

| # | Document | Scope |
|---|----------|-------|
| 00 | Overview | Architecture, requirements, implementation order |
| 01 | Kernel Boot Parameters | GRUB hardening, Secure Boot, lockdown, IOMMU |
| 02 | Sysctl Hardening | 64 kernel/network parameters |
| 03 | Firewall | firewalld DROP default, custom zones/policies |
| 04 | ARP Hardening | Static ARP, nftables Layer 2 filtering |
| 05 | LAN Isolation | MAC randomization, LLDP off, multicast suppression |
| 06 | VPN Killswitch | Quadruple killswitch (nftables + policy routing + firewall) |
| 07 | IPv6 Hardening | Selective disable, RA/DAD defense-in-depth |
| 08 | Service Minimization | 53 system + 18 user units + 2 D-Bus overrides |
| 09 | SSH Hardening | Key-only auth, service disabled by default |
| 10 | PAM & Login Security | Yescrypt, umask 027, core dumps 5x disabled, SUID audit |
| 11 | DNS & NTP | DNSSEC, DoH (Quad9), chrony NTS (4 servers) |
| 12 | SELinux & auditd | Enforcing + 38 immutable audit rules + desktop notifications |
| 13 | AIDE | Daily file integrity monitoring + notifications |
| 14 | USBGuard | Block-by-default, hash-based device identification |
| 15 | Intel ME Mitigation | 9-layer mitigation (MEI blocked, HECI off, NIC separation, IOMMU) |
| 16 | Firefox Hardening | arkenfox, FPP, CRLite, DoH, uBlock Origin |
| 17 | Desktop Stack | GNOME/Wayland hardening, autoclose-xwayland, privacy gsettings |
| 18 | Flatpak Sandboxing | Permission audit, filesystem/device/network restrictions |
| 19 | GPU & Secure Boot | Proprietary drivers with MOK signing, lockdown integrity |
| 20 | Snapshots | Snapper: root + home, timeline + pre/post, rollback procedures |
| 21 | Kernel Module Blacklisting | 46 modules across 12 categories |
| 22 | LUKS Encryption | LUKS2 AES-XTS-512, Argon2id, mount hardening, zram swap |
| 23 | NetworkManager Hardening | Static IP, MAC stable, connectivity check off |
| 24 | Firmware Updates | fwupd, LVFS, UEFI dbx, HSI |
| 25 | Update Process | Structured workflow: snapshot → dnf → flatpak → firmware → AIDE |
| 26 | Package Removal | Attack surface reduction, safe dependency handling |

