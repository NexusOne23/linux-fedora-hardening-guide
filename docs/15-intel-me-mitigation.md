# 🔩 Intel ME Mitigation

> Multi-layer mitigation for Intel Management Engine: MEI blocked, HECI disabled, NIC separation, IOMMU isolation.
> Applies to: **Intel systems only** | Fedora Workstation 43+
> Skip this document entirely if you have an AMD processor.

---

## Overview

Intel Management Engine (ME) is an autonomous microprocessor inside the Platform Controller Hub (PCH) that runs independently of your OS. It has DMA access to your entire RAM and a dedicated hardware path to the Intel onboard NIC — all below the OS, below the kernel, below the firewall.

**ME cannot be fully disabled** (attempting to do so with `me_cleaner` risks bricking the board). But the **OS-side communication channels** can be completely severed:

| Layer | Mitigation | Type |
|-------|-----------|------|
| Kernel | 5 MEI modules blacklisted + `install /bin/false` | Software |
| Kernel | udev rule blocking KT/SOL (Keyboard/Text Redirection) | Software |
| PCI | HECI controller I/O, Memory, BusMaster deactivated | Hardware state |
| Boot | Blacklists integrated in initramfs (early boot) | Software |
| Boot | Secure Boot lockdown integrity (unsigned modules blocked) | Firmware |
| Network | AMT ports 16992–16995 not open | Network |
| IOMMU | VT-d active — HECI + KT/SOL isolated in IOMMU group | Hardware/OS |
| Physical | Separate PCIe NIC instead of Intel onboard (optional) | Hardware |
| UEFI | Intel NIC disabled in BIOS (if using separate NIC) | Firmware |

### Why this matters

ME can:
- Read your entire RAM (encryption keys, passwords, session tokens) via DMA
- Access the Intel onboard NIC independently of the OS (out-of-band networking)
- Provide AMT remote management (keyboard, screen, power control)

These mitigations sever ME's communication paths: it can still read RAM, but it cannot **send** the data anywhere.

---

## 1. Kernel Module Blacklisting (All Intel Systems)

### What

Block all 5 Intel MEI (Management Engine Interface) kernel modules from loading.

### Why

The MEI modules provide the OS-side communication channel to ME. Without them, no software on your system can talk to ME — and ME cannot use the OS as a relay.

### Configuration

**File: `/etc/modprobe.d/99-security-blacklist.conf`** (ME section)

```bash
# === Intel MEI (Management Engine Interface) ===
blacklist mei
blacklist mei_me
blacklist mei_wdt
blacklist mei_hdcp
blacklist mei_pxp
```

**File: `/etc/modprobe.d/blacklist-mei-unused.conf`**

```bash
# Prevent ALL loading — even manual modprobe by root
install mei /bin/false
install mei_me /bin/false
install mei_hdcp /bin/false
install mei_pxp /bin/false
install mei_wdt /bin/false
```

### Modules explained

| Module | Purpose | Why blocked |
|--------|---------|-------------|
| `mei` | MEI core driver — main OS ↔ ME communication | No reason for OS to talk to ME |
| `mei_me` | MEI hardware backend for PCH | Hardware interface to ME |
| `mei_wdt` | ME watchdog timer | Not needed |
| `mei_hdcp` | HDCP content protection via ME | Only relevant for Intel GPU with displays |
| `mei_pxp` | Protected media path via ME | Only relevant for Intel GPU with displays |

### Why BOTH `blacklist` AND `install /bin/false`?

| Method | Protection |
|--------|-----------|
| `blacklist` | Prevents auto-loading by udev/modprobe |
| `install /bin/false` | Prevents **any** loading — even manual `modprobe mei` by an attacker with root |

`blacklist` alone is not enough: an attacker with root could run `modprobe mei` and restore ME communication. `install /bin/false` makes this fail.

### Verify

```bash
$ lsmod | grep mei
# Expected: No output — no MEI module loaded

$ modprobe -n -v mei 2>&1
# Expected: "install /bin/false"
```

---

## 2. HECI PCI Device Deactivation

### What

The HECI (Host Embedded Controller Interface) is ME's PCI device. Without a loaded driver, it remains in a deactivated state with I/O, memory, and bus mastering disabled.

### How to find your HECI device

```bash
$ lspci -nn | grep -i "communication\|HECI\|MEI"
# Example output:
# 00:16.0 Communication controller [0780]: Intel Corporation ... HECI Controller [8086:xxxx]
# Note the PCI address (e.g., 00:16.0) and device ID (e.g., 8086:xxxx)
```

### Verify deactivation

```bash
$ lspci -vvs <YOUR-HECI-PCI-ADDR> | grep "Control:"
# Expected: I/O- Mem- BusMaster- (all flags show minus = disabled)
```

> **No manual action needed.** If MEI modules are blocked (Section 1), no driver binds to the HECI device, and it stays deactivated automatically. The PCI device exists on the bus but is functionally inert.

---

## 3. Intel NIC Separation (Optional — Maximum Mitigation)

### What

Intel ME has a **dedicated hardware path** to the Intel onboard NIC that bypasses the OS entirely. Through this path, ME can send and receive network traffic independently — no firewall, no kernel, no nftables can block it.

### Wired vs WiFi — different ME access

| | Wired Ethernet (I219/I225/I226) | WiFi (AX200/AX210/BE200) |
|---|---|---|
| ME hardware path | **Dedicated** (PCH-integrated) | None (software arbitration via `iwlmei` driver) |
| True out-of-band | **Yes** — works without OS, in all power states | No — requires host driver (iwlwifi) |
| Blocked by MEI module blacklist | No — hardware path bypasses OS | **Yes** — no iwlmei driver = no WiFi ME access |

**Key insight:** ME's dedicated NIC hardware path only applies to **wired Intel Ethernet**. WiFi ME access (AMT over WiFi) requires the `iwlmei` kernel driver — which is already blocked by the MEI module blacklisting in Section 1. Physical NIC separation is only necessary for the **wired** Intel NIC.

### Why this is the strongest single mitigation

Software mitigations (module blocking, firewall rules, UEFI settings) can potentially be circumvented by a sufficiently advanced ME exploit. A **physical** separation of the wired NIC cannot:

- **No cable in the Intel wired NIC port** = ME has no network, period
- **Separate PCIe NIC** (e.g., Realtek, Broadcom) = ME has no hardware path to the active NIC

### Decision

```
Intel NIC separation strategy
├── Maximum (separate NIC + air-gap):
│   ├── Install a separate PCIe Ethernet card (non-Intel)
│   ├── Disable Intel NIC in UEFI/BIOS
│   ├── Leave the Intel NIC port physically disconnected
│   └── Result: ME can read RAM but cannot exfiltrate data over the network
│
├── Medium (UEFI disable only):
│   ├── Disable Intel NIC in UEFI/BIOS (if you have a separate NIC)
│   ├── Use the separate NIC for all networking
│   └── Note: Some BIOS updates may re-enable the Intel NIC
│
├── Basic (software only — most common):
│   ├── Block MEI modules (Section 1) — cuts OS ↔ ME communication
│   ├── Rely on IOMMU (Section 5) to isolate ME's DMA
│   ├── Firewall blocks AMT ports (Section 6)
│   └── Note: ME's dedicated NIC path is NOT blocked at hardware level
│
└── Laptops:
    ├── WiFi ME access is already blocked by MEI module blacklisting (Section 1)
    ├── If laptop has Intel wired Ethernet: use a USB Ethernet adapter instead
    ├── If laptop has only WiFi (no wired Intel NIC): software mitigations are sufficient
    └── Physical NIC separation only matters for wired Intel Ethernet
```

> **Most users will use the "Basic" approach.** The NIC separation is for users with elevated threat models who can install a separate PCIe network card.

---

## 4. KT/SOL Device Blocking (Keyboard/Text Redirection)

### What

Intel ME has a **separate PCI device** for Keyboard/Text redirection (KT) and Serial over LAN (SOL). Through this, ME can:
- Inject remote keyboard input
- Provide a remote serial console
- Capture screen content as text

### How to find your KT/SOL device

```bash
$ lspci -nn | grep -i "serial\|KT\|SOL"
# Example output:
# 00:16.3 Serial controller [0700]: Intel Corporation ... KT Redirection [8086:xxxx]
# Note the PCI address and device ID
```

### udev Rule

**File: `/etc/udev/rules.d/99-disable-mei-kt.rules`**

```bash
# Intel ME KT/SOL (Keyboard/Text Redirection) — prevent any driver from binding
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="<YOUR-KT-DEVICE-ID>", ATTR{driver_override}="none"
```

> Replace `<YOUR-KT-DEVICE-ID>` with the 4-digit device ID from `lspci -nn` (e.g., `0x7aeb`). The vendor is always `0x8086` (Intel).

**Effect:** `driver_override=none` prevents **any** driver from binding to the KT/SOL PCI device. The device exists on the PCI bus but is functionless without a driver.

**Integrate into initramfs** so the rule applies at early boot:

```bash
$ sudo dracut --force
```

### Verify

```bash
$ lspci -ks <YOUR-KT-PCI-ADDR> | grep "driver"
# Expected: No "Kernel driver in use:" line — no driver bound
```

---

## 5. IOMMU Isolation

### What

With IOMMU active ([Doc 01](01-kernel-boot-params.md), Section 5), the HECI and KT/SOL devices are isolated in their own IOMMU group. Their DMA access is restricted to memory regions explicitly mapped by the kernel — which is nothing, since no driver is loaded.

### Verify

```bash
# Find IOMMU group for HECI:
$ find /sys/kernel/iommu_groups/*/devices/ -name "<YOUR-HECI-PCI-ADDR>" 2>/dev/null
# Expected: Shows the IOMMU group containing the HECI device

# List all devices in that group:
$ ls /sys/kernel/iommu_groups/<GROUP-NUMBER>/devices/
# Expected: HECI and possibly KT/SOL in the same group, isolated from other devices
```

> **No configuration needed.** IOMMU isolation is automatic when `intel_iommu=on iommu=pt` is set ([Doc 01](01-kernel-boot-params.md)).

---

## 6. AMT Port Verification

### What

Active Management Technology (AMT) uses ports 16992–16995 for remote management. These ports should never be open on a hardened system.

### Verify

```bash
$ sudo ss -tlnp | grep -E "1699[2-5]"
# Expected: No output — AMT ports not listening
```

> Even if AMT were active on the ME firmware level, it requires the Intel NIC for network access. With the NIC disabled or physically disconnected, AMT has no transport.

---

## 7. What ME Can Still Do (Accepted Risk)

ME runs **always** — these mitigations prevent OS communication and network access, not ME itself:

| Capability | Status after mitigation |
|-----------|----------------------|
| Execute ME firmware | Running (cannot be disabled without brick risk) |
| OS ↔ ME communication | **Blocked** (modules + HECI deactivated) |
| Network via Intel NIC | **Blocked** (NIC disabled in UEFI / physically disconnected) |
| DMA to main memory | Theoretically possible (hardware-level) |
| Receive firmware updates | **Blocked** (no MEI driver, no Intel NIC) |

**Accepted risk:** ME has hardware DMA access to all RAM (LUKS master key, passwords, tokens — everything in cleartext while the system runs). This is not mitigable in software. **However:** ME cannot exfiltrate the data without a network path. With IOMMU isolating its DMA scope and the Intel NIC disconnected, ME can read but cannot send.

> **`me_cleaner`** could further reduce ME firmware functionality, but carries **brick risk**. Not recommended without a hardware SPI flash programmer for recovery.

---

## 🔧 Applying Changes

### Step 1: Blacklist MEI modules

```bash
# Add to your security blacklist (or create the file):
$ sudo tee /etc/modprobe.d/blacklist-mei-unused.conf << 'MEI'
install mei /bin/false
install mei_me /bin/false
install mei_hdcp /bin/false
install mei_pxp /bin/false
install mei_wdt /bin/false
MEI
```

> If you have a separate `99-security-blacklist.conf` ([Doc 21](21-kernel-module-blacklisting.md)), add the 5 `blacklist` lines from Section 1 there too.

### Step 2: Block KT/SOL device

```bash
# Find your KT/SOL device ID:
$ lspci -nn | grep -i "serial.*KT\|serial.*Intel"
# Note the device ID (4 hex digits after 8086:)

$ sudo tee /etc/udev/rules.d/99-disable-mei-kt.rules << 'KT'
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="<YOUR-KT-DEVICE-ID>", ATTR{driver_override}="none"
KT
```

> Replace `<YOUR-KT-DEVICE-ID>` with your actual device ID (e.g., `0x7aeb`).

### Step 3: Rebuild initramfs

```bash
$ sudo dracut --force
```

### Step 4: (Optional) Disable Intel NIC in UEFI

If you have a separate non-Intel NIC:
- Enter UEFI setup (F1/F2/DEL at boot)
- Navigate to Devices/Network or similar
- Disable "Intel LAN Controller" or "Onboard Ethernet"
- Save and exit

### Step 5: Reboot

```bash
$ sudo reboot
```

---

## ✅ Complete Verification

```bash
# 1. No MEI modules loaded:
$ lsmod | grep mei
# Expected: No output

# 2. No MEI in dmesg:
$ dmesg | grep -i "mei\|heci"
# Expected: No output

# 3. install /bin/false active:
$ modprobe -n -v mei 2>&1
# Expected: "install /bin/false"

# 4. HECI device deactivated (find your address first):
$ lspci -nn | grep -i "communication\|HECI"
# Note the address, then:
$ lspci -vvs <YOUR-HECI-PCI-ADDR> | grep "Control:"
# Expected: I/O- Mem- BusMaster- (all minus = disabled)

# 5. KT/SOL device has no driver:
$ lspci -nn | grep -i "serial.*KT\|serial.*Intel"
# Note the address, then:
$ lspci -ks <YOUR-KT-PCI-ADDR> | grep "driver"
# Expected: No "Kernel driver in use:" line

# 6. AMT ports not open:
$ sudo ss -tlnp | grep -E "1699[2-5]"
# Expected: No output

# 7. IOMMU active (prerequisite):
$ dmesg | grep "DMAR: IOMMU enabled"
# Expected: IOMMU enabled
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| MEI modules blocked | No Intel iGPU content protection (HDCP/PXP) — zero impact with NVIDIA/AMD discrete GPU | ME cannot communicate with OS |
| `install /bin/false` | Cannot load MEI even intentionally | Attacker with root cannot restore ME communication |
| Intel NIC disabled | Must use separate NIC | ME's dedicated network path leads nowhere |
| KT/SOL blocked | No remote keyboard/serial via ME | Prevents ME-based remote input injection |
| IOMMU isolation | None (IOMMU has no performance cost in pt mode) | ME's DMA scope restricted |
| me_cleaner NOT used | ME firmware fully functional | No brick risk |
| fwupd HSI false negatives | BootGuard reported as "Not supported", IOMMU may show "Not found" | Expected — fwupd needs MEI to query these; kernel-level verification still works |

---

## Notes

- **This document applies to Intel systems only.** AMD has PSP (Platform Security Processor), which has a different architecture and mitigation approach.
- **ME runs always.** These mitigations don't disable ME — they sever its communication channels. ME still runs its firmware on the PCH.
- **PCI device IDs vary by Intel generation.** The HECI and KT/SOL device IDs change between chipset generations. Always use `lspci -nn` to find YOUR device IDs — don't copy values from examples.
- **BIOS updates may re-enable Intel NIC.** After firmware updates, verify that the Intel NIC is still disabled in UEFI.
- **`dracut --force`** is required after modprobe.d or udev rule changes to integrate them into initramfs for early boot enforcement.
- **fwupd HSI false negatives:** With MEI blocked, `fwupdmgr security` will report:
  - **BootGuard: "Not supported"** — false negative. fwupd queries BootGuard status via MEI. Without MEI, it can't check. BootGuard is a hardware fuse — it's either fused at the factory or not. Blocking MEI doesn't disable it.
  - **IOMMU: "Not found"** — may be a false negative on some systems. Verify IOMMU independently: `dmesg | grep "DMAR: IOMMU enabled"` and `ls /sys/kernel/iommu_groups/ | wc -l`. If these show IOMMU active, ignore fwupd's report.
  - **Overall HSI score drops** (e.g., HSI:2! → HSI:1!) because these checks fail. The actual security is unchanged — only the detection is blocked.
  - To temporarily verify full HSI: remove the `install /bin/false` rule, run `sudo modprobe mei_me`, run `fwupdmgr security`, then unload with `sudo modprobe -r mei_me` and restore the rule.

---

*Previous: [14 — USBGuard](14-usbguard.md)*
*Next: [16 — Firefox Hardening](16-firefox-browser.md) — arkenfox user.js, fingerprinting protection, content blocking*
