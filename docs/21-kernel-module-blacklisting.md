# 🧩 Kernel Module Blacklisting

> Block unnecessary and dangerous kernel modules to reduce attack surface.
> Applies to: Fedora Workstation 43+ | All hardware
> Reboot required after changes (initramfs rebuild needed).

---

## Overview

The Linux kernel can load hundreds of modules for hardware support, filesystems, and network protocols. Most of these are never needed on a desktop workstation — but each loaded module is additional attack surface with potential privilege escalation vulnerabilities.

### Why

Kernel modules run in **Ring 0** — a vulnerability in any loaded module gives an attacker full kernel access. Historical examples:

| CVE | Module | Impact |
|-----|--------|--------|
| CVE-2017-6074 | `dccp` | Use-after-free → root privilege escalation |
| CVE-2017-2636 | `n-hdlc` | Race condition → root privilege escalation |
| CVE-2021-43267 | `tipc` | Heap overflow → remote code execution |
| CVE-2019-18683 | `vivid` | Race condition → root privilege escalation |
| CVE-2010-3904 | `rds` | Privilege escalation → root |

These modules were loaded by default on most distributions. Blacklisting prevents them from loading even if an attacker triggers auto-loading via a crafted packet or device.

### What you get

| Category | Modules | Attack surface removed |
|----------|---------|----------------------|
| Legacy network protocols | 6 | Amateur radio, AppleTalk, Oracle cluster |
| Obsolete network protocols | 2 | ATM, LLC/SNAP — dead protocols with no desktop use |
| Tunneling protocols | 3 | L2TP (if not used by your VPN) |
| Bluetooth | 6 | Wireless attack surface (if no BT hardware) |
| Intel ME interface | 5 | OS ↔ Management Engine communication |
| Rare filesystems | 8 | Parser bugs in unused filesystem drivers |
| Network filesystems | 5 | CIFS/NFS/GFS2 — remote filesystem attack surface |
| USB mass storage | 1 | Defense-in-depth with USBGuard |
| Dangerous network protocols | 5 | CVE-rich protocols never needed on desktop |
| DMA attack vectors | 3 | Physical DMA attacks via FireWire and Thunderbolt |
| Test drivers | 1 | Virtual video driver with CVE history |
| Binary format handler | 1 | binfmt_misc interpreter registration attack surface |

---

## 1. How Module Blacklisting Works

### Two methods

| Method | Protection | Bypassable? |
|--------|-----------|-------------|
| `blacklist <module>` | Prevents auto-loading by udev/modprobe | Yes — `modprobe <module>` as root still works |
| `install <module> /bin/false` | Prevents **all** loading | No — even manual `modprobe` fails |

### When to use which

```
Module blocking strategy
├── blacklist (sufficient for most modules):
│   ├── Prevents kernel from auto-loading the module
│   ├── An attacker with root could still modprobe it
│   ├── But: attacker with root already has full control
│   └── Use for: all general-purpose blacklisting
│
└── install /bin/false (maximum protection):
    ├── Module cannot be loaded under any circumstances
    ├── Even root cannot modprobe it
    ├── Use for: modules that enable communication with
    │   independent hardware (Intel ME) that operates
    │   outside OS control
    └── Overkill for most modules — reserve for special cases
```

### Where to configure

| File | Purpose |
|------|---------|
| `/etc/modprobe.d/99-security-blacklist.conf` | Your custom blacklist (all categories) |
| `/etc/modprobe.d/blacklist-mei-unused.conf` | Intel MEI `install /bin/false` overrides (→ Doc 15) |
| `/etc/modprobe.d/blacklist-*.conf` | Fedora-shipped blacklists (nouveau, floppy, etc.) |

> **Note**: Custom files in `/etc/modprobe.d/` are not overwritten by `dnf upgrade`. Fedora's own blacklist files may be updated.

---

## 2. Module Categories and Decision Trees

### 2.1 Legacy Network Protocols (6 modules)

```
Legacy network protocols: appletalk, ax25, netrom, rose, batman-adv, rds
├── Blacklist ALL (recommended):
│   ├── appletalk — obsolete Apple protocol, CVE history
│   ├── ax25, netrom, rose — amateur radio protocols
│   ├── batman-adv — mesh networking (Fedora already blacklists this)
│   └── rds — Oracle cluster protocol, CVE-2010-3904
│
├── Keep if:
│   ├── ax25/netrom/rose — you operate amateur radio with Linux
│   ├── batman-adv — you run a mesh network (rare)
│   └── rds — you run Oracle RAC clusters (not on a desktop)
│
└── What breaks: nothing on a standard desktop
```

### 2.2 Tunneling Protocols (3 modules)

```
L2TP modules: l2tp_eth, l2tp_netlink, l2tp_ppp
├── Blacklist if:  Your VPN uses WireGuard or OpenVPN (not L2TP)
├── Keep if:       Your VPN provider uses L2TP/IPsec
└── What breaks:   L2TP-based VPN connections
    Check: ask your VPN provider which protocol they use.
    Most modern VPN providers use WireGuard or OpenVPN.
```

### 2.3 Bluetooth (6 modules)

```
Bluetooth: bluetooth, btusb, btrtl, btbcm, btintel, btmtk
├── Blacklist ALL if:
│   ├── Desktop without Bluetooth hardware
│   ├── Laptop where you never use Bluetooth
│   └── Bluetooth already disabled in BIOS/UEFI
│
├── Keep ALL if:
│   ├── You use Bluetooth headphones, keyboard, or mouse
│   ├── You use Bluetooth for file transfer
│   └── You plan to use Bluetooth in the future
│
├── What breaks: ALL Bluetooth functionality
│   ├── No Bluetooth devices can connect
│   ├── GNOME Bluetooth settings panel shows nothing
│   └── Bluetooth service (already masked in Doc 08) has no driver
│
└── Note: blacklisting is defense-in-depth with service masking (→ Doc 08)
    Both layers together ensure Bluetooth is fully disabled.
```

### 2.4 Intel ME Interface (5 modules) — Intel systems only

```
Intel MEI: mei, mei_me, mei_wdt, mei_hdcp, mei_pxp
├── Blacklist + install /bin/false (recommended for Intel systems):
│   ├── Cuts OS ↔ Management Engine communication
│   ├── Both blacklist AND install /bin/false for maximum protection
│   └── Full details: → Doc 15 (Intel ME Mitigation)
│
├── Skip if:
│   ├── AMD system (no Intel ME — AMD has PSP but different modules)
│   ├── You need Intel AMT for remote management
│   └── You need fwupd to detect BootGuard/IOMMU via MEI (→ Doc 15)
│
└── What breaks:
    ├── fwupd HSI report shows false negatives (BootGuard "Not supported")
    ├── Intel AMT remote management (if you use it)
    └── mei_hdcp/mei_pxp: zero impact if you use a discrete NVIDIA/AMD GPU
```

### 2.5 Rare Filesystems (8 modules)

```
Filesystems: floppy, cramfs, freevxfs, jffs2, hfs, hfsplus, udf, squashfs
├── Blacklist ALL (recommended):
│   ├── floppy — no floppy drives exist anymore, CVE history
│   ├── cramfs — compressed ROM filesystem (embedded devices)
│   ├── freevxfs — Veritas proprietary filesystem
│   ├── jffs2 — journaling flash filesystem (embedded devices)
│   ├── hfs/hfsplus — Apple filesystems (legacy)
│   ├── udf — Blu-ray/DVD format
│   └── squashfs — compressed read-only FS (used by Snap, AppImage)
│
├── Keep selectively:
│   ├── udf — if you have an optical drive and read Blu-ray/DVD discs
│   ├── squashfs — if you use AppImages (they mount as squashfs)
│   ├── hfsplus — if you access macOS-formatted drives
│   └── floppy — if you actually have a floppy drive (unlikely in 2025+)
│
└── What breaks:
    ├── squashfs blocked → AppImages won't mount, Snap won't work
    │   (Fedora doesn't use Snap — but check if you use AppImages)
    ├── udf blocked → cannot read Blu-ray/DVD data discs
    ├── hfsplus blocked → cannot mount macOS external drives
    └── Others: nothing on a standard desktop
```

### 2.6 USB Mass Storage (1 module)

```
USB storage: usb_storage
├── Blacklist (recommended if you use USBGuard):
│   ├── Defense-in-depth with USBGuard (→ Doc 14)
│   ├── Modern USB drives use UAS (USB Attached SCSI) — not usb_storage
│   └── UAS is faster and separate from usb_storage
│
├── Keep if:
│   ├── You use very old USB drives that don't support UAS
│   ├── You don't use USBGuard
│   └── You're unsure — test your USB drives first
│
├── What breaks:
│   ├── USB drives using the legacy BOT (Bulk-Only Transport) protocol
│   ├── Modern USB 3.0+ drives are unaffected (they use UAS)
│   └── Test: plug in your USB drive after blacklisting — if it mounts, UAS is working
│
└── How to check which protocol your drive uses:
    dmesg | grep -E "uas|usb.storage" (after plugging in the drive)
```

### 2.7 Dangerous Network Protocols (5 modules)

```
Network protocols: dccp, sctp, tipc, n-hdlc, can
├── Blacklist ALL (strongly recommended):
│   ├── dccp — CVE-2017-6074 (use-after-free → root)
│   ├── sctp — telecom protocol, extensive CVE history
│   ├── tipc — cluster IPC, CVE-2021-43267 (remote code exec)
│   ├── n-hdlc — serial protocol, CVE-2017-2636
│   └── can — automotive/industrial bus protocol
│
├── Keep if:
│   ├── sctp — you develop or test telecom (SIP/Diameter) applications
│   ├── can — you work with automotive/industrial hardware via Linux
│   └── Others: no legitimate desktop use case exists
│
└── What breaks: nothing on a standard desktop.
    These protocols are never used in normal networking.
```

### 2.8 Obsolete Network Protocols (2 modules)

```
Obsolete protocols: atm, psnap
├── Blacklist ALL (recommended):
│   ├── atm — Asynchronous Transfer Mode, obsolete WAN protocol
│   └── psnap — LLC/SNAP protocol, legacy Layer 2
│
├── Keep if:
│   └── Never — no modern network uses ATM or SNAP
│
└── What breaks: nothing on a standard desktop
```

### 2.9 Network Filesystems (5 modules)

```
Network filesystems: cifs, nfs, nfsv3, nfsv4, gfs2
├── Blacklist ALL (recommended):
│   ├── cifs — SMB/Windows network shares client, extensive CVE history
│   ├── nfs, nfsv3, nfsv4 — NFS client, not needed on standalone desktop
│   └── gfs2 — Red Hat cluster filesystem (server/HPC only)
│
├── Keep if:
│   ├── cifs — you mount Windows/Samba network shares
│   ├── nfs/nfsv3/nfsv4 — you mount NFS exports from a NAS or server
│   └── gfs2 — never on a desktop (cluster filesystem)
│
└── What breaks:
    ├── cifs blocked → cannot mount SMB/CIFS shares (\\server\share)
    ├── nfs blocked → cannot mount NFS exports (server:/export)
    └── gfs2 blocked → nothing on desktop
```

### 2.10 DMA Attack Vectors (3 modules)

```
FireWire + Thunderbolt: firewire-core, firewire-ohci, thunderbolt
├── Blacklist ALL (recommended):
│   ├── FireWire allows Direct Memory Access (DMA) by external devices
│   ├── Thunderbolt allows DMA over USB-C (same attack class as FireWire)
│   ├── Physical attack: attacker connects device → reads RAM
│   ├── Can extract: encryption keys, passwords, session tokens
│   └── IOMMU mitigates DMA attacks but blocking the module is defense-in-depth
│
├── Keep if:
│   ├── firewire — you use FireWire audio interfaces (rare in 2025+)
│   ├── thunderbolt — you use a Thunderbolt dock, eGPU, or Thunderbolt storage
│   └── Note: USB-C without Thunderbolt is NOT affected (USB uses xhci_hcd)
│
└── What breaks:
    ├── firewire blocked → all FireWire device connectivity
    └── thunderbolt blocked → Thunderbolt docks, eGPUs, Thunderbolt displays
        (USB-C charging and USB-C data still work — only Thunderbolt protocol affected)
```

### 2.11 Test Drivers (1 module)

```
Test driver: vivid
├── Blacklist (recommended):
│   ├── Virtual video test driver — only useful for kernel developers
│   ├── CVE-2019-18683 (race condition → privilege escalation)
│   └── No legitimate use on a production system
│
├── Keep if: you develop or test V4L2 video drivers
│
└── What breaks: nothing — only affects kernel video driver testing
```

### 2.12 Binary Format Misc (1 module)

```
Binary format handler: binfmt_misc
├── Blacklist + install /bin/false (recommended):
│   ├── Allows registering arbitrary binary format interpreters via
│   │   /proc/sys/fs/binfmt_misc/register
│   ├── Used by Wine (run Windows .exe) and QEMU user-mode (run foreign arch binaries)
│   ├── An attacker with root could register a malicious interpreter for
│   │   persistence or privilege escalation
│   └── On a desktop without Wine or QEMU user-mode: pure attack surface
│
├── Keep if:
│   ├── You use Wine to run Windows applications
│   ├── You use QEMU user-mode emulation (e.g., running ARM binaries on x86)
│   └── You use systemd-nspawn containers with foreign-architecture images
│
└── What breaks:
    ├── Wine cannot register its PE format handler (Wine won't auto-execute .exe files)
    ├── QEMU user-mode cannot register its binary format handlers
    └── Nothing else — binfmt_misc is not used by standard desktop applications
```

---

## 3. Configuration Files

### Main blacklist: `/etc/modprobe.d/99-security-blacklist.conf`

```bash
# === Legacy network protocols ===
blacklist appletalk
blacklist ax25
blacklist netrom
blacklist rose
blacklist batman-adv
blacklist rds

# === Tunneling protocols (if your VPN doesn't use L2TP) ===
blacklist l2tp_eth
blacklist l2tp_netlink
blacklist l2tp_ppp

# === Bluetooth (if no Bluetooth hardware/need) ===
blacklist bluetooth
blacklist btusb
blacklist btrtl
blacklist btbcm
blacklist btintel
blacklist btmtk

# === Intel MEI — Intel systems only (see also Doc 15) ===
blacklist mei
blacklist mei_me
blacklist mei_wdt
blacklist mei_hdcp
blacklist mei_pxp

# === Floppy ===
blacklist floppy

# === Rare/unused filesystems ===
blacklist cramfs
blacklist freevxfs
blacklist jffs2
blacklist hfs
blacklist hfsplus
blacklist udf
blacklist squashfs

# === USB mass storage (defense-in-depth with USBGuard) ===
blacklist usb_storage

# === Dangerous network protocols ===
blacklist dccp
blacklist sctp
blacklist tipc
blacklist n-hdlc
blacklist can

# === Obsolete network protocols ===
blacklist atm
blacklist psnap

# === Network filesystems (no desktop use case) ===
blacklist cifs
blacklist nfs
blacklist nfsv3
blacklist nfsv4
blacklist gfs2

# === DMA attack vectors ===
blacklist firewire-core
blacklist firewire-ohci
blacklist thunderbolt

# === Test drivers ===
blacklist vivid

# === Binary format handler (if no Wine/QEMU user-mode) ===
blacklist binfmt_misc
```

### binfmt_misc override: `/etc/modprobe.d/blacklist-binfmt.conf`

> **Skip if you use Wine or QEMU user-mode emulation.**

```bash
blacklist binfmt_misc
install binfmt_misc /bin/false
```

### Intel MEI overrides: `/etc/modprobe.d/blacklist-mei-unused.conf`

> **Intel systems only** — skip this file on AMD systems.

```bash
# Intel ME modules — install /bin/false prevents ALL loading
install mei /bin/false
install mei_me /bin/false
install mei_hdcp /bin/false
install mei_pxp /bin/false
install mei_wdt /bin/false
```

### How to create these files

```bash
# Create main blacklist:
sudo tee /etc/modprobe.d/99-security-blacklist.conf << 'EOF'
# (paste the content from above)
EOF

# Create MEI overrides (Intel only):
sudo tee /etc/modprobe.d/blacklist-mei-unused.conf << 'EOF'
# (paste the content from above)
EOF

# Rebuild initramfs (required — blacklists must be in early boot):
sudo dracut --force

# Reboot:
sudo reboot
```

> **Why dracut**: Module blacklists in `/etc/modprobe.d/` are read by the kernel during boot. The initramfs must include these files so they take effect before the kernel tries to auto-load modules. Without `dracut --force`, changes only apply after the initramfs stage (too late for some modules).

---

## 4. Customizing Your Blacklist

### Remove modules you need

If a module from the blacklist is required on your system, simply remove or comment out that line:

```bash
# Example: keep squashfs for AppImages
# blacklist squashfs    ← commented out
```

Then rebuild initramfs and reboot:

```bash
sudo dracut --force && sudo reboot
```

### Add additional modules

If you identify modules loaded on your system that you don't need:

```bash
# List all currently loaded modules:
lsmod | wc -l

# Check what a specific module does:
modinfo <module-name>

# Add to blacklist if not needed:
echo "blacklist <module-name>" | sudo tee -a /etc/modprobe.d/99-security-blacklist.conf
sudo dracut --force
```

### Undo all blacklisting

```bash
# Remove custom blacklist files:
sudo rm /etc/modprobe.d/99-security-blacklist.conf
sudo rm /etc/modprobe.d/blacklist-mei-unused.conf

# Rebuild initramfs:
sudo dracut --force

# Reboot — all modules will auto-load again:
sudo reboot
```

---

## 5. Complete Verification

```bash
# 1. No blacklisted modules loaded:
lsmod | grep -E "appletalk|ax25|bluetooth|mei|firewire|dccp|sctp|cramfs|vivid|atm|psnap|cifs|nfs|gfs2|thunderbolt"
# Expected: no output

# 2. install /bin/false active (Intel MEI):
modprobe -n -v mei 2>&1
# Expected: "install /bin/false"

# 3. Blacklist file exists:
ls -la /etc/modprobe.d/99-security-blacklist.conf
# Expected: file exists with your blacklist content

# 4. Blacklist is in initramfs:
lsinitrd /boot/initramfs-$(uname -r).img | grep 99-security
# Expected: etc/modprobe.d/99-security-blacklist.conf listed

# 5. Count loaded modules (fewer = better):
lsmod | wc -l
# Compare before/after blacklisting — should be noticeably fewer
```

---

## Important Notes

- **dracut --force after every change**: Without rebuilding initramfs, blacklist changes don't apply at early boot. Some modules load before the root filesystem is mounted — they must be blocked in initramfs.
- **Fedora-shipped blacklists**: Fedora already blacklists some modules (nouveau when NVIDIA installed, floppy, batman-adv). Your `99-` prefix ensures your file is loaded after Fedora's defaults.
- **squashfs and AppImages**: If you use AppImages, don't blacklist `squashfs` — AppImages are squashfs archives that need this module to mount.
- **usb_storage vs UAS**: Modern USB 3.0+ drives use UAS (USB Attached SCSI), not `usb_storage`. Blacklisting `usb_storage` typically doesn't affect modern USB drives. Test before committing.
- **module.sig_enforce=1**: If you set this boot parameter (→ Doc 01), only signed modules can load. This is a separate, complementary protection to blacklisting.
- **Intel MEI on AMD systems**: AMD systems don't have Intel ME. Skip the MEI section entirely. AMD has PSP (Platform Security Processor) but it uses different kernel interfaces.
- **Bluetooth on laptops**: Many laptops have built-in Bluetooth. Don't blacklist Bluetooth modules on a laptop unless you're certain you'll never need them.

---

*Previous: [20 — Snapshots](20-snapshots.md)*
*Next: [22 — LUKS Encryption](22-luks-encryption.md) — Full disk encryption with LUKS2 and Argon2id*
