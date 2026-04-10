# ⚙️ Kernel Boot Parameters

> Boot-level hardening: memory protection, attack surface reduction, CPU mitigations, IOMMU, Secure Boot, and kernel lockdown.
> Applies to: Fedora Workstation 43+ | All x86_64 hardware
> Reboot required after changes.

---

## Overview

Kernel boot parameters are the **first line of defense** after the bootloader. They control how the kernel initializes memory, handles CPU vulnerabilities, restricts its own attack surface, and enforces module security — before any userspace process starts.

This document covers:
- **13 hardening parameters** added to GRUB (plus conditional GPU parameters)
- Kernel lockdown via Secure Boot
- Full LSM (Linux Security Module) stack verification
- CPU vulnerability mitigation status
- IOMMU for DMA protection

Changes are made in `/etc/default/grub` and applied with `grub2-mkconfig`.

> **Related:** Kernel module blacklisting (modprobe.d) is covered separately in [Doc 21 — Kernel Module Blacklisting](21-kernel-module-blacklisting.md). Runtime kernel parameters (sysctl) are covered in [Doc 02 — Sysctl Hardening](02-sysctl-hardening.md).

---

## 1. Memory Hardening

### What

Five parameters that harden kernel memory allocation against exploitation.

### Why

Kernel heap exploits (use-after-free, heap spraying, slab overflow) are among the most common privilege escalation vectors in Linux. These parameters make exploitation significantly harder by eliminating information leaks and adding randomization at the allocator level.

Without these parameters, freed memory retains its previous contents (data leaks), slab caches of similar size are merged (enabling cross-cache attacks), and allocation patterns are predictable (enabling heap spraying).

### Parameters

| Parameter | What it does | Threat mitigated |
|-----------|-------------|------------------|
| `init_on_alloc=1` | Zero-fills memory on allocation | Information leaks from uninitialized memory |
| `init_on_free=1` | Zero-fills memory on deallocation | Use-after-free data recovery |
| `slab_nomerge` | Prevents merging of same-size SLAB caches | Cross-cache slab overflow attacks |
| `page_alloc.shuffle=1` | Randomizes page allocator free lists | Heap spraying, predictable allocation |
| `randomize_kstack_offset=on` | Randomizes kernel stack offset per syscall | Stack-based kernel exploits |

### Decision

```
Memory hardening parameters
├── Set ALL if:    Desktop or server (recommended for everyone)
├── Skip if:       Embedded systems with extreme memory constraints
└── Performance:   ~1-2% overhead (init_on_alloc/free), negligible for others
```

> **Recommendation:** Set all five. The performance impact is minimal on modern hardware and the security benefit is substantial.

### How

Add to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`:

```
init_on_alloc=1 init_on_free=1 slab_nomerge page_alloc.shuffle=1 randomize_kstack_offset=on
```

### Verify

```bash
$ cat /proc/cmdline | tr ' ' '\n' | grep -E "init_on|slab|page_alloc|randomize_kstack"
init_on_alloc=1
init_on_free=1
slab_nomerge
page_alloc.shuffle=1
randomize_kstack_offset=on
```

### What breaks

Nothing. These parameters are safe for all workloads.

### Undo

Remove the parameters from `GRUB_CMDLINE_LINUX`, regenerate GRUB config, reboot.

---

## 2. Attack Surface Reduction

### What

Three parameters that disable unnecessary or dangerous kernel interfaces.

### Why

The kernel exposes several legacy and debug interfaces that serve no purpose on a hardened desktop but provide valuable primitives for attackers:

- **vsyscall**: A fixed-address memory page from the pre-vDSO era. Because its address is constant and predictable, it serves as a reliable ROP (Return-Oriented Programming) gadget for exploit chains.
- **debugfs**: Exposes internal kernel state through `/sys/kernel/debug`. Useful for kernel developers, dangerous on production systems — it can leak kernel addresses, timing information, and in some cases allow state manipulation.
- **PTI disabled**: Without Page Table Isolation, the kernel's page tables are mapped in userspace, enabling Meltdown-class attacks to read kernel memory.

### Parameters

| Parameter | What it does | Threat mitigated |
|-----------|-------------|------------------|
| `pti=on` | Enforces Page Table Isolation (separate kernel/user page tables) | Meltdown (CVE-2017-5754), kernel address leaks |
| `vsyscall=none` | Disables the legacy vsyscall page | ROP gadgets at known fixed address |
| `debugfs=off` | Completely disables debugfs (`/sys/kernel/debug`) | Information disclosure, kernel state manipulation |

### Decision

```
pti=on (Page Table Isolation)
├── Set if:      Intel CPU (always recommended)
├── Optional:    AMD CPU (Meltdown doesn't apply, but minimal overhead)
├── Note:        Fedora enables this by default on Intel; explicit param ensures it
└── Performance: ~2-5% on syscall-heavy workloads

vsyscall=none
├── Set if:      Always — no modern software uses vsyscall
├── Skip if:     Running ancient 32-bit binaries from ~2010 era
└── What breaks: Nothing on any system from the last decade

debugfs=off
├── Set if:      Production/hardened desktop (recommended)
├── Keep if:     You actively debug kernel drivers or use tools requiring debugfs
└── What breaks: Some debugging/profiling tools (perf, ftrace, some GPU debug tools)
```

### How

Add to `GRUB_CMDLINE_LINUX`:

```
pti=on vsyscall=none debugfs=off
```

### Verify

```bash
# PTI active:
$ dmesg | grep "page tables isolation"
Kernel/User page tables isolation: enabled

# vsyscall disabled:
$ cat /proc/cmdline | grep "vsyscall=none"
# Should appear in output

# debugfs disabled:
$ ls /sys/kernel/debug/ 2>&1
# Should be empty or return an error
```

### What breaks

- `debugfs=off`: GPU debugging tools, some `perf` features, kernel tracing. Not needed on a desktop.
- `pti=on`: ~2-5% overhead on syscall-heavy workloads (negligible for typical desktop use).
- `vsyscall=none`: Nothing on modern systems.

### Undo

Remove parameters, regenerate GRUB config, reboot.

---

## 3. CPU Vulnerability Mitigations

### What

Two parameters that explicitly enable CPU vulnerability mitigations beyond kernel defaults.

### Why

Modern CPUs have architectural flaws (Spectre, Meltdown, MDS, TAA) that allow side-channel attacks. While the Fedora kernel enables most mitigations by default (`mitigations=auto`), two parameters are worth setting explicitly to ensure they remain active regardless of kernel defaults.

### Parameters

| Parameter | What it does | Threat mitigated |
|-----------|-------------|------------------|
| `spec_store_bypass_disable=on` | Disables speculative store bypass | Spectre v4 (CVE-2018-3639) |
| `tsx=off` | Disables Intel TSX (Transactional Synchronization Extensions) | TAA (CVE-2019-11135), TSX-based side channels |

### Decision

```
spec_store_bypass_disable=on
├── Set if:      Always recommended
├── Performance: Minimal — single-digit percentage in worst case
└── Note:        May already be active by default; explicit is safer

tsx=off (Intel-specific)
├── Set if:      Intel CPU with TSX support (varies by SKU, 6th-12th Gen)
├── Skip if:     AMD CPU (parameter is ignored — harmless to include)
├── Skip if:     Intel 13th Gen+ (TSX removed from hardware)
└── What breaks: Hardware Transactional Memory workloads (extremely rare on desktop)
```

### Parameters NOT set — conscious decisions

These parameters are frequently recommended but intentionally omitted:

```
mitigations=auto
├── Already the kernel default — setting it explicitly is redundant
├── DO NOT set mitigations=off — it disables ALL CPU mitigations
└── No action needed

nosmt (disable Hyperthreading / SMT)
├── Set if:      Maximum security AND you accept ~50% CPU throughput loss
├── Skip if:     Desktop workstation (performance impact too severe)
├── Note:        Modern CPUs (Intel 12th Gen+, AMD Zen 3+) are not affected
│                by most SMT-specific attacks (MDS, L1TF, TAA)
└── What breaks: Halves available CPU threads — significant performance regression

oops=panic
├── Set if:      Server that must reboot on kernel bugs rather than continue in degraded state
├── Skip if:     Desktop — you want to investigate the oops, not lose your session
└── What breaks: System reboots immediately on any kernel oops (including non-critical ones)
```

### How

Add to `GRUB_CMDLINE_LINUX`:

```
spec_store_bypass_disable=on tsx=off
```

### Verify

```bash
# Check all CPU vulnerability mitigations at once:
$ for v in /sys/devices/system/cpu/vulnerabilities/*; do
    echo "$(basename $v): $(cat $v)"
done

# Expected: No "Vulnerable" entries
# "Not affected" = your CPU is architecturally immune
# "Mitigation: ..." = your CPU is affected but the attack is mitigated
```

### What breaks

- `tsx=off`: Programs using Intel Hardware Transactional Memory (virtually none on desktop).
- `spec_store_bypass_disable=on`: Negligible performance impact.

### Undo

Remove parameters, regenerate GRUB config, reboot.

---

## 4. Module Signature Enforcement

### What

Force the kernel to only load cryptographically signed modules.

### Why

Without signature enforcement, an attacker with root access could load a malicious kernel module — effectively a kernel-level rootkit with full system control. With `module.sig_enforce=1`, only modules signed by the distribution key (Fedora's key) or your own MOK (Machine Owner Key) can be loaded.

This works in conjunction with Secure Boot: the UEFI firmware verifies the bootloader, the bootloader verifies the kernel, and `module.sig_enforce` verifies every module loaded into the kernel. The entire boot chain is cryptographically validated.

### Parameter

| Parameter | What it does | Threat mitigated |
|-----------|-------------|------------------|
| `module.sig_enforce=1` | Rejects all unsigned kernel modules | Kernel rootkits, unauthorized drivers |

### Decision

```
module.sig_enforce=1
├── Set if:      Secure Boot is enabled (recommended for all users)
├── Skip if:     You regularly load custom/unsigned kernel modules
├── Requires:    Secure Boot enabled in UEFI firmware
└── What breaks: Any unsigned or improperly signed kernel module fails to load
```

> **Proprietary GPU drivers:** NVIDIA's proprietary driver (akmod-nvidia) requires MOK (Machine Owner Key) signing. Fedora's `akmods` handles this automatically. **AMD and Intel users need no action** — their GPU drivers (`amdgpu`, `i915`/`xe`) are in-tree and already Fedora-signed. See [Doc 19 — GPU & Secure Boot](19-gpu-secureboot.md) for details. Exception: AMD's proprietary AMDGPU PRO/ROCm (via `amdgpu-install`) uses DKMS and does require MOK — same as NVIDIA.

### How

Add to `GRUB_CMDLINE_LINUX`:

```
module.sig_enforce=1
```

### Verify

```bash
# Secure Boot active:
$ mokutil --sb-state
SecureBoot enabled

# Module signature enforcement in cmdline:
$ cat /proc/cmdline | grep "module.sig_enforce"
# Should show module.sig_enforce=1
```

### What breaks

- Any unsigned kernel module (including manually compiled ones without MOK signing)
- DKMS modules that haven't been signed (Fedora's `akmods` handles this automatically)

### Undo

Remove `module.sig_enforce=1`, regenerate GRUB config, reboot.

---

## 5. IOMMU — DMA Protection

### What

Enable the IOMMU (Input/Output Memory Management Unit) to isolate DMA-capable devices.

### Why

Every PCIe device (GPU, NIC, USB controller, NVMe drive) has Direct Memory Access — the ability to read and write **any physical memory address** without CPU involvement. This is by design for performance, but it means a compromised or malicious device can:

- Read encryption keys from RAM
- Inject code into the running kernel
- Bypass all software security (including SELinux, firewalls, everything)

IOMMU creates per-device memory domains, ensuring each device can only access memory regions the kernel explicitly maps for it. A rogue NIC cannot read GPU memory, a compromised USB controller cannot read your LUKS keys.

**Real-world attacks:** DMA attacks are practical via Thunderbolt (DMA over USB-C), FireWire, and compromised firmware in PCIe devices. IOMMU is your only defense.

### Parameters

| Parameter | What it does | Applies to |
|-----------|-------------|------------|
| `intel_iommu=on` | Enables Intel VT-d IOMMU | Intel systems only |
| `amd_iommu=on` | Enables AMD-Vi IOMMU | AMD systems only |
| `iommu=pt` | Passthrough mode — DMA remapping active, no performance overhead | Both Intel and AMD |

### Decision

```
IOMMU
├── Enable if:     Always recommended — protects against DMA attacks from any PCIe device
├── Skip if:       Very old hardware without VT-d/AMD-Vi support (pre-2013)
├── Intel systems: intel_iommu=on iommu=pt
├── AMD systems:   amd_iommu=on iommu=pt
│                  (AMD may enable IOMMU by default on Zen+ processors)
└── What breaks:   Rarely anything on hardware from the last decade

⚠️  iommu=pt vs iommu=force
├── iommu=pt:     RECOMMENDED — passthrough mode with DMA protection, fully compatible
├── iommu=force:  NOT recommended — forces DMA translation for all devices, can crash
│                 systems with proprietary GPU drivers (NVIDIA, some AMD)
└── Always use iommu=pt
```

### How

**Intel systems** — add to `GRUB_CMDLINE_LINUX`:

```
intel_iommu=on iommu=pt
```

**AMD systems** — add to `GRUB_CMDLINE_LINUX`:

```
amd_iommu=on iommu=pt
```

> **Finding your CPU vendor:** `lscpu | grep "Vendor ID"` — `GenuineIntel` = Intel, `AuthenticAMD` = AMD.

### Verify

```bash
# IOMMU active:
$ dmesg | grep -i "IOMMU\|DMAR\|AMD-Vi"
# Intel: Look for "DMAR: IOMMU enabled"
# AMD: Look for "AMD-Vi" initialization messages

# Count IOMMU groups (each group = isolated DMA domain):
$ ls /sys/kernel/iommu_groups/ | wc -l
# Should be > 0 (typically 10-20+ on modern hardware)

# List all devices and their IOMMU groups:
$ for g in /sys/kernel/iommu_groups/*/devices/*; do
    group=$(basename $(dirname $(dirname "$g")))
    device=$(basename "$g")
    echo "Group $group: $(lspci -nns "$device" 2>/dev/null || echo "$device")"
done
```

### What breaks

- Very old PCIe devices without proper DMA support (extremely rare on post-2015 hardware)
- `iommu=force` crashes systems with some GPU drivers — always use `iommu=pt`

### Undo

Remove `intel_iommu=on iommu=pt` (or `amd_iommu=on iommu=pt`), regenerate GRUB config, reboot.

---

## 6. GPU Boot Parameters (Conditional)

### What

Boot-level driver selection for systems with proprietary GPU drivers.

### Why

Proprietary GPU drivers (NVIDIA) conflict with the open-source alternatives (nouveau). Both cannot be loaded simultaneously — loading both causes crashes, display failures, or silent fallback to software rendering. Boot-level blacklisting ensures the open-source driver never loads, not even during early boot (initramfs stage).

### Decision

```
GPU driver blacklisting
├── NVIDIA proprietary (akmod-nvidia):
│   ├── Add:     rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core
│   └── See:     Doc 19 for full Secure Boot + MOK setup
│
├── AMD (amdgpu — open-source kernel driver):
│   ├── No boot parameters needed — amdgpu is the default
│   └── amdgpu-pro (proprietary userspace) does not require boot params
│
├── Intel integrated only:
│   ├── No boot parameters needed
│   └── i915/xe drivers work out of the box
│
└── Open-source drivers only (no proprietary GPU driver):
    └── No GPU-related boot parameters needed — skip this section
```

### How (NVIDIA only)

Add to `GRUB_CMDLINE_LINUX`:

```
rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core
```

| Parameter | What it does |
|-----------|-------------|
| `rd.driver.blacklist=nouveau,nova_core` | Blocks nouveau in initramfs (earliest boot stage) |
| `modprobe.blacklist=nouveau,nova_core` | Blocks nouveau in userspace modprobe |

> `nova_core` is the in-development Rust-based nouveau replacement. Blacklisting it now prevents future conflicts when it becomes available in mainline kernels.
>
> **Note:** `nvidia-drm.modeset=1` and `nvidia-drm.fbdev=1` are **not needed as GRUB parameters**. They are set via two redundant mechanisms: (1) compiled-in module defaults since NVIDIA driver 580.x (`modinfo nvidia-drm | grep parm` shows `(default)`), and (2) RPM Fusion's `/etc/modprobe.d/nvidia.conf` (`options nvidia-drm modeset=1 fbdev=1`). Verify: `cat /sys/module/nvidia_drm/parameters/modeset` (expected: `Y`).

### Verify

```bash
# Correct driver loaded:
$ lsmod | grep -E "nvidia|nouveau"
# Should show nvidia modules, NOT nouveau

# Modesetting active:
$ cat /sys/module/nvidia_drm/parameters/modeset
Y
```

### What breaks

Without these parameters (on NVIDIA systems): proprietary driver fails to load, or conflicts with nouveau causing display issues.

### Undo

Remove NVIDIA parameters, regenerate GRUB config, reboot. System falls back to nouveau.

---

## 7. Kernel Lockdown

### What

Kernel lockdown restricts what even the root user can do to the running kernel.

### Why

Root compromise is a realistic scenario — a vulnerable service, a malicious package, or a browser exploit with privilege escalation can give an attacker root. Without lockdown, root can:

- Load unsigned kernel modules (rootkits)
- Read raw physical memory via `/dev/mem` (extract encryption keys)
- Modify CPU MSR registers (disable security features)
- Use `kexec` to boot an unsigned kernel (bypass Secure Boot entirely)

Lockdown prevents all of this. On Fedora, it activates **automatically** when Secure Boot is enabled — no explicit boot parameter needed.

### Modes

| Mode | Protection level | Compatibility |
|------|-----------------|---------------|
| `none` | No restrictions | Default without Secure Boot |
| **`integrity`** | **Prevents kernel code/data modification** | **Default with Secure Boot — recommended** |
| `confidentiality` | Integrity + prevents reading kernel memory | Breaks proprietary GPU drivers |

### Decision

```
Kernel Lockdown
├── integrity (recommended):
│   ├── Automatically enabled by Secure Boot on Fedora
│   ├── No explicit boot parameter needed
│   └── Compatible with NVIDIA/AMD proprietary drivers
│
├── confidentiality (maximum):
│   ├── Stronger — also prevents reading kernel memory (/dev/mem read)
│   ├── Set via: lockdown=confidentiality in GRUB_CMDLINE_LINUX
│   ├── ⚠️ Breaks NVIDIA proprietary drivers (they need kernel memory access)
│   └── Only viable with pure open-source GPU stack (Intel i915/xe, AMD amdgpu)
│
└── none:
    ├── Only active if Secure Boot is disabled
    ├── NOT recommended — kernel is fully exposed to root-level attacks
    └── Enable Secure Boot in UEFI firmware to get integrity mode
```

### What lockdown=integrity protects against

- Loading unsigned kernel modules
- Direct access to `/dev/mem`, `/dev/kmem`, `/dev/port`
- `kexec` with unsigned kernels
- Write access to CPU MSR registers
- eBPF write access to kernel memory
- Hibernation image tampering

### Verify

```bash
$ cat /sys/kernel/security/lockdown
none [integrity] confidentiality
# The value in [brackets] is the active mode
```

### What breaks

- **integrity**: Nothing on a properly configured Fedora system.
- **confidentiality**: NVIDIA proprietary drivers, some hardware monitoring tools, `perf` hardware counters, any tool that reads `/dev/mem`.

---

## 8. Linux Security Module (LSM) Stack

### What

The LSM stack defines which security frameworks are active in the kernel, loaded in order.

### Why

Each LSM provides a different security layer. Modern Fedora ships with a comprehensive 9-module stack. This is not something you configure via boot parameters — it is compiled into the kernel. This section is for **verification** that your kernel has the expected stack.

### Expected Fedora 43+ LSM Stack

```bash
$ cat /sys/kernel/security/lsm
lockdown,capability,yama,selinux,bpf,landlock,ipe,ima,evm
```

| LSM | Purpose | Related doc |
|-----|---------|-------------|
| `lockdown` | Kernel integrity protection (Section 7 above) | — |
| `capability` | POSIX capabilities (fine-grained privilege control) | — |
| `yama` | ptrace restrictions (`ptrace_scope`) | [Doc 02](02-sysctl-hardening.md) |
| `selinux` | Mandatory Access Control | [Doc 12](12-selinux-auditd.md) |
| `bpf` | eBPF security controls | — |
| `landlock` | Unprivileged application sandboxing | — |
| `ipe` | Integrity Policy Enforcement | — |
| `ima` | Integrity Measurement Architecture | — |
| `evm` | Extended Verification Module | — |

### Decision

```
LSM Stack
├── Modify:  Never — Fedora's default stack is comprehensive and correctly ordered
├── Note:    Removing any LSM weakens the overall security model
└── Action:  Verify after kernel updates to ensure the full stack is present
```

> **No configuration needed.** This section exists to document what the stack contains and why. Verify after major kernel upgrades.

---

## 9. `modules_disabled` — Why It's NOT Set

This parameter deserves its own section because it is frequently recommended in hardening guides without adequate context about its limitations.

### What `modules_disabled=1` does

Once set (it's a one-way toggle), it permanently prevents **all** kernel module loading for the remainder of the boot. Even root cannot load modules after the flag is set.

### Decision

```
modules_disabled=1
├── Set if:      All needed modules are built-in or auto-loaded at boot,
│                AND you never hot-plug ANY hardware
├── Skip if:     You use proprietary GPU drivers (NVIDIA/AMD — they load modules at boot + runtime)
├── Skip if:     You use USBGuard (needs to load USB device modules dynamically)
├── Skip if:     You use any USB devices (storage, YubiKey, phone, etc.)
├── Skip if:     You ever dock/undock hardware
├── What it does: Makes module loading impossible — no exceptions, no override, until reboot
└── Tradeoff:    Maximum kernel protection vs. zero hardware flexibility
```

> **For desktop users:** `module.sig_enforce=1` (Section 4) provides strong module security — only signed modules can load — without the extreme inflexibility of `modules_disabled`. This is the recommended approach.

---

## 🔧 Applying All Changes

### Step 1: Identify Your System

```bash
# CPU vendor (Intel or AMD):
$ lscpu | grep "Vendor ID"

# LUKS UUID (for boot):
$ sudo blkid | grep crypto_LUKS
# Note the UUID after "UUID="

# GPU driver in use:
$ lspci -k | grep -A 2 -E "VGA|3D"
# Look at "Kernel driver in use:" — nvidia, amdgpu, i915, or xe
```

### Step 2: Build Your Parameter String

Start with the **universal baseline** (works on all x86_64 systems):

```
GRUB_CMDLINE_LINUX="rd.luks.uuid=luks-<YOUR-LUKS-UUID> rhgb quiet init_on_alloc=1 init_on_free=1 slab_nomerge pti=on vsyscall=none debugfs=off page_alloc.shuffle=1 randomize_kstack_offset=on spec_store_bypass_disable=on tsx=off module.sig_enforce=1"
```

Then **add IOMMU** based on your CPU:

| CPU | Add to GRUB_CMDLINE_LINUX |
|-----|--------------------------|
| Intel | `intel_iommu=on iommu=pt` |
| AMD | `amd_iommu=on iommu=pt` |

Then **add GPU parameters** if using NVIDIA proprietary:

```
rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core
```

> **AMD GPU and Intel GPU users:** No GPU boot parameters needed.

### Step 3: Edit GRUB

```bash
$ sudo nano /etc/default/grub
# Replace the GRUB_CMDLINE_LINUX line with your built string
```

> **Important: ALWAYS edit `/etc/default/grub`**, not BLS entries directly. On Fedora (BLS-based), `grub2-mkconfig` regenerates boot entries from `/etc/default/grub`. Parameters added via `grubby --update-kernel --args="..."` are written to BLS entries directly — but they get **overwritten** the next time `grub2-mkconfig` runs (e.g., during kernel updates). Only `/etc/default/grub` is the persistent source of truth.

### Step 4: Regenerate GRUB Configuration

```bash
# Fedora 43+ (BLS-based — works for both BIOS and UEFI):
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Step 5: Reboot

```bash
$ sudo reboot
```

---

## ✅ Complete Verification

Run all checks after reboot:

```bash
# 1. All boot parameters applied:
$ cat /proc/cmdline
# Verify all your parameters are present

# 2. Kernel lockdown active:
$ cat /sys/kernel/security/lockdown
# Expected: none [integrity] confidentiality

# 3. Secure Boot enabled:
$ mokutil --sb-state
# Expected: SecureBoot enabled

# 4. LSM stack complete:
$ cat /sys/kernel/security/lsm
# Expected: lockdown,capability,yama,selinux,bpf,landlock,ipe,ima,evm

# 5. IOMMU active:
$ dmesg | grep -i "IOMMU\|DMAR\|AMD-Vi"
# Intel: "DMAR: IOMMU enabled"
# AMD: "AMD-Vi" initialization messages
$ ls /sys/kernel/iommu_groups/ | wc -l
# Expected: > 0 (typically 10-20+ groups)

# 6. CPU vulnerabilities mitigated:
$ for v in /sys/devices/system/cpu/vulnerabilities/*; do
    echo "$(basename $v): $(cat $v)"
done
# Expected: No "Vulnerable" entries
# "Not affected" = architecturally immune
# "Mitigation: ..." = affected but mitigated

# 7. Module signature enforcement:
$ cat /proc/cmdline | grep "module.sig_enforce"
# Expected: module.sig_enforce=1
```

---

## ⚠️ Known Tradeoffs

| Parameter | Cost | Benefit |
|-----------|------|---------|
| `init_on_alloc=1` / `init_on_free=1` | ~1-2% performance overhead | Eliminates entire class of memory info leaks |
| `pti=on` | ~2-5% on syscall-heavy workloads | Prevents Meltdown-class kernel memory reads |
| `debugfs=off` | Kernel debugging/profiling tools unavailable | Removes significant kernel info disclosure surface |
| `module.sig_enforce=1` | Cannot load unsigned modules | Prevents kernel rootkit installation |
| `iommu=pt` | None (passthrough has no measurable overhead) | Isolates all DMA-capable devices |
| `lockdown=integrity` | Cannot escalate to `confidentiality` at runtime | Prevents kernel tampering even by root |
| NVIDIA + `lockdown` | Cannot use `confidentiality` mode | `integrity` is sufficient; `confidentiality` requires open-source GPU |
| `tsx=off` | No Hardware Transactional Memory | Eliminates TAA side-channel (no desktop software uses TSX) |

---

*Previous: [00 — Overview](00-overview.md)*
*Next: [02 — Sysctl Hardening](02-sysctl-hardening.md) — 64 kernel runtime parameters for network, memory, and process security*
