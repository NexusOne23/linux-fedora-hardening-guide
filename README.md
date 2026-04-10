# 🛡️ Linux Hardening Guide for Fedora Workstation

> Comprehensive Linux hardening guide for Fedora Workstation — 27 documents, tested on real hardware, covering every security layer from UEFI Secure Boot to browser fingerprinting.

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-blue.svg)](LICENSE)
[![Fedora 43+](https://img.shields.io/badge/Fedora-43%2B-blue.svg)](https://fedoraproject.org/)
[![Audit Score](https://img.shields.io/badge/Audit-119%2F119-brightgreen.svg)](#audit-results)
[![Version](https://img.shields.io/badge/Version-1.0-blue.svg)](CHANGELOG.md)

---

## 📋 What This Is

A complete, tested, and verified hardening guide for **Fedora Workstation**. Every configuration has been implemented on a real system, audited, and documented with reproduction steps and verification commands.

This is a **maximum-reference guide** — it documents everything that CAN be hardened, not everything you MUST harden. Every section includes decision trees that help you choose what applies to your system, hardware, and threat model.

### Audit Results

- **119/119** security checks passed (custom audit v3.1)
- **98% FORTRESS** rating (253 pass, 0 fail, 4 warn — all false positives) — audited with [NoID Privacy for Linux](https://github.com/NexusOne23/noid-privacy-linux) (our own open-source audit tool)
- Exceeds CIS RHEL 9 v2.0.0 and ANSSI-BP-028 v2.0 High Level

---

## 👤 Who This Is For

- Linux users who want to **understand** what they're hardening and **why**
- Privacy-conscious individuals running Fedora as a daily driver
- Security enthusiasts who want defense-in-depth beyond basic guides
- System administrators looking for a Fedora-specific hardening reference
- Journalists, activists, and researchers with elevated threat models

**Prerequisites:**
- Fedora Workstation (43+) installed with full disk encryption (LUKS2)
- Comfortable with the terminal and `sudo`
- Willingness to read before copy-pasting commands

> **This guide assumes a single-user desktop workstation.** Server hardening has different requirements and tradeoffs.

---

## 🎚️ Choose Your Depth

You don't need to implement everything. Pick a level that matches your needs:

| Level | Documents | Time | What you get |
|-------|-----------|------|-------------|
| **Essential** | 01-03, 08-09, 12 | ~2 hours | Kernel hardening, firewall, service reduction, SSH, SELinux |
| **Recommended** | Essential + 04-07, 10-11, 21-22 | ~4 hours | + network isolation, PAM, DNS/NTP, encryption, module blacklisting |
| **Maximum** | All 27 documents | ~8 hours | Full defense-in-depth across every layer |

Every document is independently actionable. Start with Essential, add more layers as you see fit. The decision trees in each document tell you when to apply a setting and when to skip it.

---

## 🎯 Threat Model

**This guide protects against:**

- 🌐 Remote attackers (network-based intrusion, scanning, exploitation)
- 📡 Local network threats (ARP spoofing, rogue DHCP/DNS, LAN surveillance)
- 💀 Malware and drive-by exploits (browser 0-days, malicious downloads)
- 👁️ Privacy erosion (tracking, fingerprinting, telemetry, DNS leaks)
- 🔌 Physical access (cold boot, DMA attacks, USB attacks, boot manipulation)
- 📦 Supply chain attacks (compromised packages, firmware, updates)

**This guide does NOT protect against:**

- A targeted nation-state adversary with unlimited resources (use [Qubes OS](https://www.qubes-os.org/))
- Hardware implants at manufacturing level
- Rubber-hose cryptanalysis

---

## 🏗️ Defense Layers

```
┌─────────────────────────────────────────────────────┐
│              🔐 UEFI Secure Boot                     │
│          (Shim → GRUB → Kernel → Modules)            │
├─────────────────────────────────────────────────────┤
│              🔒 LUKS2 (AES-XTS-512)                  │
│           Argon2id, Full Disk Encryption              │
├─────────────────────────────────────────────────────┤
│        ⚙️ Kernel Hardening (Boot Params + 64 sysctls) │
│   lockdown=integrity, pti=on, 46 modules blacklisted │
├─────────────────────────────────────────────────────┤
│           🛡️ SELinux Enforcing + auditd               │
│        Targeted Policy, 38 audit rules, -e 2         │
├─────────────────────────────────────────────────────┤
│           🔥 Firewall (4 nftables tables)             │
│    Default DROP, ARP hardening, VPN killswitch        │
├─────────────────────────────────────────────────────┤
│              🌐 Network Isolation                     │
│    MAC random, LLDP off, IGMP off, broadcast DROP     │
├─────────────────────────────────────────────────────┤
│              🔗 VPN (WireGuard)                       │
│     Quadruple killswitch, DNS over tunnel             │
├─────────────────────────────────────────────────────┤
│              ⛔ Service Minimization                   │
│       53+18 masked units, D-Bus overrides             │
├─────────────────────────────────────────────────────┤
│           🔑 Access Control (PAM/SSH/USBGuard)        │
│   Key-only SSH, Yescrypt, USB whitelist, umask 027    │
├─────────────────────────────────────────────────────┤
│           📊 Integrity Monitoring                     │
│             AIDE + auditd (daily)                     │
├─────────────────────────────────────────────────────┤
│              🖥️ Desktop Hardening                     │
│   Wayland, autoclose-xwayland, lockscreen privacy     │
├─────────────────────────────────────────────────────┤
│              🦊 Browser Hardening                     │
│  arkenfox, uBlock Origin, WebRTC off, ECH, DoH       │
├─────────────────────────────────────────────────────┤
│              📸 Snapshots (Snapper)                   │
│      Btrfs timeline + DNF pre/post rollback           │
└─────────────────────────────────────────────────────┘
```

---

## 📚 Documents

### ⚙️ Kernel & System

| # | Document | Topics |
|---|----------|--------|
| 01 | [Kernel Boot Parameters](docs/01-kernel-boot-params.md) | slab_nomerge, pti, lockdown, vsyscall=none, debugfs=off, tsx=off, module.sig_enforce, IOMMU |
| 02 | [Sysctl Hardening](docs/02-sysctl-hardening.md) | 64 sysctl parameters: kptr_restrict, ptrace_scope, ASLR, ICMP, ARP, io_uring, mmap_rnd_bits, user namespaces |
| 21 | [Kernel Module Blacklisting](docs/21-kernel-module-blacklisting.md) | 46 blacklisted modules (legacy protocols, Bluetooth, MEI, filesystems, network FS, FireWire, Thunderbolt, binfmt_misc) |
| 22 | [LUKS Encryption](docs/22-luks-encryption.md) | LUKS2 AES-XTS-512, Argon2id, Btrfs layout, noexec tmpfs, zram swap, header backup |

### 🌐 Network & Firewall

| # | Document | Topics |
|---|----------|--------|
| 03 | [Firewall](docs/03-firewall.md) | firewalld zones (drop, strict-wan), policies (block-lan-out, allow-host-ipv6) |
| 04 | [ARP Hardening](docs/04-nftables-arp-hardening.md) | Static ARP, nftables arp table, NM dispatcher, Layer 2 protection |
| 05 | [LAN Isolation](docs/05-lan-isolation.md) | MAC randomization, LLDP off, multicast/broadcast DROP, IGMP suppression |
| 06 | [VPN Killswitch](docs/06-vpn-killswitch.md) | Quadruple killswitch: nftables IPv4/IPv6 + policy routing + firewall policy |
| 07 | [IPv6 Hardening](docs/07-ipv6-hardening.md) | Selective IPv6 disable, RA/DAD/autoconf hardening, redirect blocking |
| 23 | [NetworkManager Hardening](docs/23-networkmanager-hardening.md) | Static IP, MAC stable, LLDP off, connectivity check off |

### 🔑 Services & Authentication

| # | Document | Topics |
|---|----------|--------|
| 08 | [Service Minimization](docs/08-service-minimization.md) | 53 masked system units + 18 user units + 2 D-Bus overrides |
| 09 | [SSH Hardening](docs/09-ssh-hardening.md) | Key-only, root denied, MaxAuthTries 3, service disabled |
| 10 | [PAM & Login Security](docs/10-pam-login-security.md) | nullok removed, Yescrypt, umask 027, SUID audit, core dumps 5x disabled |
| 11 | [DNS & NTP](docs/11-dns-ntp.md) | DNSSEC, DoH, chrony NTS (4 servers), LLMNR/mDNS off |

### 📊 Monitoring & Integrity

| # | Document | Topics |
|---|----------|--------|
| 12 | [SELinux & auditd](docs/12-selinux-auditd.md) | SELinux enforcing + hardened booleans, 38 audit rules, immutable (-e 2) |
| 13 | [AIDE](docs/13-aide.md) | Daily file integrity monitoring, systemd timer |

### 🔩 Hardware & Firmware

| # | Document | Topics |
|---|----------|--------|
| 14 | [USBGuard](docs/14-usbguard.md) | ImplicitPolicyTarget=block, hash-based device identification |
| 15 | [Intel ME Mitigation](docs/15-intel-me-mitigation.md) | Multi-layer mitigation: MEI blocked, HECI disabled, NIC separation, IOMMU isolation |
| 19 | [GPU & Secure Boot](docs/19-gpu-secureboot.md) | Proprietary GPU drivers (NVIDIA/AMD) with MOK signing |
| 24 | [Firmware Updates](docs/24-firmware-updates.md) | fwupd, LVFS, UEFI dbx, manual update policy |

### 🖥️ Desktop & Applications

| # | Document | Topics |
|---|----------|--------|
| 16 | [Firefox Hardening](docs/16-firefox-browser.md) | arkenfox, FPP, CRLite, DoH, uBlock Origin, WebRTC off, ECH |
| 17 | [Desktop Stack](docs/17-desktop-stack.md) | GNOME/Wayland, autoclose-xwayland, masked user services, D-Bus overrides |
| 18 | [Flatpak Sandboxing](docs/18-flatpak-sandboxing.md) | Minimal Flatpak apps, permission lockdown |
| 20 | [Snapshots](docs/20-snapshots.md) | Snapper: root + home, timeline + DNF pre/post, retention policy |

### 🔧 Maintenance

| # | Document | Topics |
|---|----------|--------|
| 25 | [Update Process](docs/25-update-process.md) | Systematic update workflow: Snapper bracket, akmods, Flatpak, AIDE rebuild |
| 26 | [Package Removal](docs/26-package-removal.md) | Attack surface reduction through package cleanup |

---

## 🚀 Quick Start

1. **Read the [Overview](docs/00-overview.md) first** — understand the architecture before changing anything
2. **Choose your depth level** — Essential, Recommended, or Maximum (see table above)
3. **Follow the recommended order:** LUKS (22) → Kernel (01-02) → Network (03-07) → Services (08) → Auth (09-11) → Monitoring (12-13) → Hardware (14-15) → Desktop (16-18) → GPU (19) → Snapshots (20) → Modules (21) → NM (23) → Firmware (24)
4. **Reboot required after:** kernel parameters (01), module blacklisting (21), auditd -e 2 (12)
5. **Rebuild AIDE database** after every change
6. **All `sudo` commands are manual** — this guide is not an automated script

> **Tip:** An AI coding assistant with terminal access (such as [Claude Code](https://claude.ai/code)) can read this guide and run the verification commands against your system — significantly speeding up implementation and catching misconfigurations.

---

## ⚠️ Accepted Risks

Every hardened system has conscious tradeoffs:

| Risk | Detail | Mitigation |
|------|--------|------------|
| GPU proprietary drivers | Closed-source userspace + firmware | Open kernel module, MOK-signed |
| Intel ME | Autonomous processor in PCH with DMA access | Multi-layer mitigation (doc 15) |
| Browser 0-days | Every website is an attack surface | arkenfox + Seccomp sandbox + uBlock Origin |
| Desktop stack | GNOME/Wayland/PipeWire/D-Bus inherently broader than server | Maximum service reduction |
| User namespaces | Most common building block in kernel exploits | Limited to 256 (not disabled) — Flatpak + Firefox sandbox require them |
| Per-app egress | No per-process outbound filtering on Linux | VPN tunnel + nftables DROP default |

---

## 💡 What Makes This Guide Different

| Feature | This Guide | Most Other Guides |
|---------|-----------|-------------------|
| **Tested** | Every command verified on real hardware | Theoretical or copy-pasted |
| **Depth** | 27 documents, 13 layers | Single page or checklist |
| **Fedora-specific** | Correct packages, paths, SELinux contexts | Distro-agnostic (often wrong) |
| **Decision trees** | Every section helps you decide: apply or skip | "Do this" with no context |
| **Verification** | Every section has verification commands | "Trust me, it works" |
| **Tradeoffs** | Documents what was NOT done and why | Silent about limitations |
| **ARP/Layer 2** | Full nftables arp hardening with static ARP + MAC filter | Not covered anywhere |
| **LAN Isolation** | Layer 2-7 complete invisibility | Basic firewall rules |

---

## 🔍 Comparison with Other Guides

| Guide | Scope | Difference |
|-------|-------|------------|
| **CIS RHEL Benchmark** | Compliance checklist | We go far beyond CIS in network isolation, ME mitigation, browser hardening |
| **ANSSI-BP-028** | Government recommendations | We cover desktop-specific topics ANSSI ignores (browser, Flatpak, GPU) |
| **PrivSec.dev** | Desktop overview | We are deeper in every area (4 nftables tables vs basic firewalld) |
| **secureblue** | Automated Fedora Atomic | We explain *why*, they automate *what* — complementary, not competing |
| **Madaidan's Guide** | General Linux | We are Fedora-specific, more current, and cover more layers |

---

## 🤝 Contributing

Contributions are welcome! Please:

- Test changes on a real Fedora system before submitting
- Include verification commands for any new configuration
- Document tradeoffs and what breaks if applicable
- One topic per pull request

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

---

## 📜 License

This work is licensed under [Creative Commons Attribution-ShareAlike 4.0 International](LICENSE).

You are free to share and adapt this material for any purpose, including commercial use, as long as you give appropriate credit and distribute your contributions under the same license.

---

<p align="center">
  <i>"You don't do security. Security needs to live rent free in your mind at all times."</i><br>
  — <a href="https://www.youtube.com/watch?v=40SnEd1RWUU">Kai Lentit</a>
</p>
