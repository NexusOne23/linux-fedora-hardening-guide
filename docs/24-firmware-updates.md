# ⚙️ Firmware Updates (fwupd)

> Keep firmware current using fwupd and the Linux Vendor Firmware Service (LVFS).
> Applies to: Fedora Workstation 43+ | All hardware with LVFS support
> Reboot required after firmware updates.

---

## Overview

Firmware runs below the OS — in your UEFI/BIOS, NVMe controller, embedded controller, Thunderbolt chip, and more. Firmware vulnerabilities bypass all OS-level hardening because they execute before the kernel loads.

### Why

| Firmware type | What it does | Risk if outdated |
|--------------|-------------|-----------------|
| System BIOS/UEFI | Initializes hardware, loads bootloader | Secure Boot bypasses, boot-level malware |
| CPU microcode | Patches CPU behavior at boot | Spectre/Meltdown variants, side-channel attacks |
| NVMe/SSD controller | Manages disk I/O | Data corruption, encryption bypass |
| Intel ME firmware | Management Engine operations | Remote exploitation (even with MEI blocked) |
| UEFI dbx | Secure Boot forbidden signatures | Revoked bootloaders still trusted |
| Embedded controller | Keyboard, fan, power management | Hardware-level persistence |

### What you get

| Component | Detail |
|-----------|--------|
| Tool | `fwupd` (pre-installed on Fedora) |
| Source | LVFS (Linux Vendor Firmware Service) — signed firmware from vendors |
| Activation | D-Bus on-demand — not a permanent service |
| Update method | Manual via CLI (`fwupdmgr`) |
| Verification | Signature-checked by fwupd before flashing |

---

## 1. How fwupd Works

### Architecture

```
fwupdmgr (CLI)  →  fwupd (D-Bus daemon)  →  LVFS (online repo)
                                            ↓
                                    Download signed firmware
                                            ↓
                                    Verify signature
                                            ↓
                                    Flash via UEFI Capsule / USB / NVMe
                                            ↓
                                    Reboot to apply
```

### D-Bus activation (on-demand)

```bash
systemctl is-enabled fwupd
# Expected: static
```

`static` means fwupd is **not a permanent service**. It starts only when requested via D-Bus (e.g., when you run `fwupdmgr`) and stops after inactivity. Minimal attack surface.

### LVFS trust model

| Aspect | Detail |
|--------|--------|
| Who provides firmware | Hardware vendors (Lenovo, Dell, HP, Intel, etc.) upload to LVFS |
| Signing | Firmware is signed by the vendor; fwupd verifies the signature |
| Transport | Downloaded over HTTPS from `fwupd.org` |
| Open source | LVFS is a Linux Foundation project — fully open |

> **Supply chain**: fwupd never flashes unsigned firmware. The signature chain is: vendor signs firmware → uploads to LVFS → fwupd downloads → verifies signature → flashes. A compromised LVFS mirror cannot inject malicious firmware.

---

## 2. Checking Your Firmware

### List managed devices

```bash
fwupdmgr get-devices
```

This shows every device fwupd knows about — BIOS, embedded controller, NVMe, TPM, Intel ME, USB peripherals, etc. Not all devices support firmware updates via LVFS.

### Check for available updates

```bash
# Refresh metadata from LVFS:
fwupdmgr refresh

# Check for updates:
fwupdmgr get-updates
```

If all firmware is current, you'll see:
```
Devices with the latest available firmware version:
 • (list of devices)
```

### View update history

```bash
fwupdmgr get-history
```

Shows past firmware updates with dates, versions, and success/failure status.

---

## 3. Installing Firmware Updates

### How

```bash
# Install all available updates:
fwupdmgr update

# Reboot to apply (UEFI Capsule updates are applied during reboot):
sudo reboot
```

### What happens during update

1. fwupd downloads the firmware from LVFS
2. Verifies the vendor signature
3. Stages the update (writes to EFI System Partition)
4. On reboot: UEFI firmware applies the staged update
5. System boots with updated firmware

### Decision

```
Firmware update strategy
├── Manual CLI updates (recommended):
│   ├── You control when updates happen
│   ├── Firmware updates require reboot — schedule when convenient
│   ├── Run periodically: fwupdmgr refresh && fwupdmgr get-updates
│   └── Include in your update routine (→ Doc 25)
│
├── Automatic updates via GNOME Software:
│   ├── gnome-software.service checks for firmware updates
│   ├── If masked (→ Doc 08): no automatic firmware checks
│   ├── Re-enable if you want automatic notifications
│   └── Trade-off: convenience vs control
│
└── Ignore firmware updates (not recommended):
    ├── Firmware vulnerabilities bypass all OS hardening
    ├── UEFI dbx updates are critical for Secure Boot integrity
    └── CPU microcode patches fix hardware-level vulnerabilities
```

---

## 4. UEFI dbx (Secure Boot Revocation)

### What

The UEFI dbx (Forbidden Signature Database) is a list of revoked Secure Boot signatures. When a bootloader is found to have a security vulnerability, its signature is added to dbx — preventing it from being trusted by Secure Boot.

### Why it matters

Without dbx updates, an attacker could use a known-vulnerable bootloader to bypass Secure Boot and load unsigned code.

### How to update

dbx updates are delivered through fwupd like any other firmware update:

```bash
fwupdmgr refresh && fwupdmgr get-updates
# If a dbx update is available:
fwupdmgr update
```

> **Always apply dbx updates promptly** — they close Secure Boot bypass vulnerabilities.

---

## 5. CPU Microcode

### Important distinction

CPU microcode is **not** updated via fwupd. It's delivered through the kernel package:

| Distribution | Microcode package | Updated via |
|-------------|------------------|-------------|
| Fedora (Intel) | `microcode_ctl` | `dnf upgrade` |
| Fedora (AMD) | `linux-firmware` | `dnf upgrade` |

The kernel loads microcode at early boot — before any userspace runs. This patches CPU-level vulnerabilities (Spectre, Meltdown, MMIO stale data, etc.).

### Verify

```bash
# Current microcode version:
journalctl -b | grep microcode
# Expected: "microcode updated early" or similar

# Or:
cat /proc/cpuinfo | grep microcode
# Shows the loaded microcode revision
```

> **Keep your kernel updated**: Microcode patches arrive with kernel updates. `dnf upgrade` is the delivery mechanism — fwupd cannot help here.

---

## 6. Host Security ID (HSI)

fwupd includes a system security assessment called HSI (Host Security ID). This scores your system's firmware security:

```bash
fwupdmgr security
```

### HSI levels

| Level | Meaning |
|-------|---------|
| HSI:0 | No firmware security features |
| HSI:1 | Basic security (UEFI Secure Boot, TPM) |
| HSI:2 | Extended security (IOMMU, BootGuard verified) |
| HSI:3 | Maximum hardware security |
| `!` suffix | Runtime issues detected (e.g., kernel tainted, suspend-to-RAM) |

### Known false negatives

If you've blocked Intel MEI modules (→ Doc 15, Doc 21), fwupd cannot communicate with the Management Engine. This causes:

| HSI check | Reported | Actual | Cause |
|-----------|----------|--------|-------|
| BootGuard | "Not supported" | May be fused and verified | MEI blocked — fwupd can't query ME |
| IOMMU | "Not found" | May be active | Missing IOMMU boot parameters (→ Doc 01) or MEI blocked |

These are **false negatives** — the features may work, but fwupd can't detect them. For IOMMU, the most common cause is missing boot parameters (→ Doc 01 for full IOMMU setup), not MEI blocking. Verify independently:

```bash
# IOMMU active:
dmesg | grep -i "IOMMU enabled\|DMAR"
# Expected: "DMAR: IOMMU enabled" or similar

# BootGuard (if MEI temporarily loaded):
# Requires temporarily unblocking MEI — see Doc 15 for procedure
```

---

## 7. Complete Verification

```bash
# 1. fwupd installed:
rpm -q fwupd
# Expected: fwupd package listed

# 2. Firmware current:
fwupdmgr refresh && fwupdmgr get-updates
# Expected: all devices at latest version

# 3. Update history:
fwupdmgr get-history
# Expected: past updates with "Success" status

# 4. No automatic updates (if gnome-software is masked):
systemctl --user is-enabled gnome-software.service 2>/dev/null
# Expected: masked (or not found)

# 5. HSI score:
fwupdmgr security
# Expected: HSI:1 or higher

# 6. Microcode loaded:
journalctl -b | grep -i microcode
# Expected: microcode update message
```

---

## Important Notes

- **D-Bus activation is correct**: `systemctl is-enabled fwupd` showing `static` is normal — fwupd starts on demand, not at boot. Don't try to enable it.
- **Intel ME firmware updates**: fwupd can update ME firmware via UEFI Capsule even with MEI modules blocked (→ Doc 15). The update happens at UEFI level, not through MEI. The OS-level MEI block is unaffected.
- **BIOS SPI lock**: The BIOS flash is typically hardware-locked (SPI write protection). Firmware updates use UEFI Capsule (signed, vendor-provided) — not direct SPI write. "Device firmware has been locked" in fwupd is expected.
- **Not all hardware supports LVFS**: Some NVMe drives, GPUs, and peripherals are not on LVFS. For those, check the vendor's website for firmware updates (often bootable ISO images).
- **VPN and firmware downloads**: fwupd downloads firmware over your active network connection. If you use a VPN, downloads go through the VPN tunnel — your ISP doesn't see what firmware you're downloading.
- **Reboot timing**: UEFI Capsule updates are applied during reboot, before the OS loads. Don't interrupt the reboot process — a failed firmware flash can brick the device.
- **Include in update routine**: Run `fwupdmgr refresh && fwupdmgr get-updates` as part of your regular update process (→ Doc 25).

---

*Previous: [23 — NetworkManager Hardening](23-networkmanager-hardening.md)*
*Next: [25 — Update Process](25-update-process.md) — Systematic update workflow with integrity verification*
