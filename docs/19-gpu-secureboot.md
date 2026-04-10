# 🔏 GPU Drivers & Secure Boot

> Proprietary GPU drivers with Secure Boot and kernel lockdown — MOK signing, akmod workflow, trusted boot chain.
> Applies to: Fedora Workstation 43+ | NVIDIA (akmod), AMD, Intel GPUs
> Reboot required after driver installation and MOK enrollment.

---

## Overview

Secure Boot ensures only trusted code runs during boot — from UEFI firmware through bootloader, kernel, and kernel modules. Proprietary GPU drivers (NVIDIA) require special handling because they ship unsigned kernel modules that Secure Boot would reject.

This document covers Secure Boot verification, kernel lockdown modes, and GPU driver installation with module signing.

### Why

Without Secure Boot, malware can inject code into the boot process (bootkits, evil maid attacks). Without module signing, a compromised system could load malicious kernel modules. The boot chain must be trusted end-to-end:

```
UEFI Firmware → Shim (Fedora-signed) → GRUB (Fedora-signed) → Kernel (Fedora-signed)
  → Kernel modules (Fedora CA or MOK-signed)
```

If any link is unsigned or tampered with, Secure Boot rejects it.

### GPU driver landscape

| GPU | Driver | Kernel module | Secure Boot | Action needed |
|-----|--------|--------------|-------------|---------------|
| **NVIDIA** | Proprietary (akmod-nvidia) | `nvidia.ko` — built from source per kernel | Must be MOK-signed | MOK enrollment + akmod signing |
| **AMD** | Open source (mesa + amdgpu) | `amdgpu` — ships with kernel | Already Fedora-signed | None — works out of the box |
| **Intel** | Open source (mesa + i915/xe) | `i915`/`xe` — ships with kernel | Already Fedora-signed | None — works out of the box |

**AMD and Intel users**: Your GPU drivers are part of the kernel and already signed by Fedora. Skip to Section 2 (Kernel Lockdown) — no MOK enrollment needed.

> **Exception**: If you install AMD's proprietary **AMDGPU PRO** or **ROCm** via `amdgpu-install`, those use DKMS (out-of-tree modules) and **do** require MOK enrollment — same as NVIDIA. The in-tree `amdgpu` driver (Fedora default) does not.

---

## 1. Secure Boot

### Verify

```bash
mokutil --sb-state
# Expected: SecureBoot enabled
```

If Secure Boot is disabled, enable it in your UEFI/BIOS settings. Fedora supports Secure Boot out of the box via the Shim bootloader.

### Enrolled keys (MOK — Machine Owner Key)

```bash
# List all enrolled MOK keys:
mokutil --list-enrolled
```

On a fresh Fedora installation with NVIDIA akmod:

| Key | Purpose |
|-----|---------|
| Fedora Secure Boot CA | Signs Fedora kernel and in-tree modules |
| akmods signing key | Signs third-party kernel modules (NVIDIA, VirtualBox, etc.) |

> **Note**: The akmods key is generated automatically during `akmod-nvidia` installation. You enroll it via the MOK Manager on next reboot.

### Decision

```
Secure Boot
├── Enable (strongly recommended):
│   ├── Prevents boot-level malware (bootkits, evil maid)
│   ├── Enables kernel lockdown (integrity mode)
│   ├── Required for full trust chain
│   └── NVIDIA: requires MOK enrollment (one-time setup)
│
├── Disable (avoid unless absolutely necessary):
│   ├── Only if: hardware/firmware doesn't support Secure Boot
│   ├── Risk: no boot chain verification, no kernel lockdown
│   └── All kernel module signing becomes advisory only
│
└── Troubleshooting:
    ├── "SecureBoot disabled" but enabled in BIOS → check shim-x64 is installed
    ├── NVIDIA module fails to load → MOK key not enrolled (see Section 3)
    └── "module verification failed" in dmesg → module not signed or wrong key
```

---

## 2. Kernel Lockdown

### What

Kernel lockdown restricts what userspace can do to the running kernel. It's normally activated automatically by Secure Boot via the Shim bootloader.

> **Important**: CVE-2025-1272 affected Fedora kernels 6.12–6.12.13 where lockdown was not auto-activated despite Secure Boot being enabled (`CONFIG_SECURITY_LOCKDOWN_LSM_EARLY` was disabled in build config). Fixed in kernel 6.12.14+. **Always verify** lockdown is actually active — don't assume.

### Verify

```bash
cat /sys/kernel/security/lockdown
# Expected: none [integrity] confidentiality
# The value in brackets is the active mode
```

If you see `[none]` despite Secure Boot being enabled, add `lockdown=integrity` to your kernel command line (→ Doc 01).

### Lockdown modes

| Mode | What's blocked | Use case |
|------|---------------|----------|
| `none` | Nothing | Secure Boot disabled |
| **`integrity`** | Unsigned modules, `/dev/mem`, `kexec`, hibernate with unsigned image | **Recommended — default with Secure Boot** |
| `confidentiality` | Everything in integrity + `/proc/kcore`, `/dev/kmem`, perf events, eBPF reads | Maximum restriction |

### Decision

```
Kernel lockdown mode
├── integrity (default with Secure Boot — recommended):
│   ├── Blocks unsigned kernel modules
│   ├── Blocks direct memory access (/dev/mem)
│   ├── Blocks kexec (kernel replacement at runtime)
│   ├── Compatible with NVIDIA proprietary drivers (when MOK-signed)
│   └── Compatible with all standard desktop workloads
│
├── confidentiality (stricter):
│   ├── Additionally blocks kernel memory reads (/proc/kcore)
│   ├── Blocks perf hardware events and eBPF kernel reads
│   ├── May break: some profiling/debugging tools
│   ├── May break: NVIDIA driver (depends on version)
│   └── Only if: you need maximum kernel memory protection
│
└── none (Secure Boot disabled):
    ├── No module signing enforcement
    ├── All kernel interfaces accessible
    └── Not recommended — enables boot-level attacks
```

> **Important**: `lockdown=confidentiality` can break NVIDIA drivers because they may need kernel memory access for GPU communication. Test thoroughly before using confidentiality mode with proprietary drivers.

---

## 3. NVIDIA Driver Installation (Secure Boot)

> **AMD/Intel users**: Skip this section entirely. Your drivers are in-tree and already signed.

### Prerequisites

```bash
# RPM Fusion repositories (required for NVIDIA packages):
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

### Install

```bash
# Core NVIDIA driver (akmod builds module per kernel automatically):
sudo dnf install akmod-nvidia

# Optional but recommended:
sudo dnf install \
  xorg-x11-drv-nvidia-cuda \
  nvidia-settings \
  libva-nvidia-driver

# kernel-devel is required for module compilation:
sudo dnf install kernel-devel
```

### What each package does

| Package | Purpose | Required |
|---------|---------|----------|
| `akmod-nvidia` | Auto-builds signed kernel module on each kernel update | Yes |
| `xorg-x11-drv-nvidia-cuda` | CUDA compute support | If you use CUDA |
| `nvidia-settings` | GUI settings panel | Optional |
| `libva-nvidia-driver` | VA-API backend for hardware video decoding | Recommended |
| `kernel-devel` | Kernel headers for module compilation | Yes — **must always be installed** |

### Wait for module build

```bash
# Force immediate build:
sudo akmods --force

# Or check build status:
sudo akmods --check
# Expected: "is already built" for your current kernel
```

> **Important**: Wait 2-5 minutes after installation before rebooting. akmod needs to compile and sign the module. Rebooting too early = no NVIDIA driver = software rendering.

### MOK enrollment (one-time)

After the first `akmod-nvidia` installation, a signing key is auto-generated in `/etc/pki/akmods/`. You must import and enroll it in MOK:

```bash
# Import the public key into MOK (sets a one-time password):
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
# You will be prompted to create a one-time password — remember it for the next reboot
```

Then reboot and complete enrollment:

1. **Reboot** the system
2. **MOK Manager** appears automatically (blue screen before GRUB)
3. Select **"Enroll MOK"**
4. Confirm the key fingerprint
5. Enter the **one-time password** you set with `mokutil --import`
6. Select **"Reboot"**

After enrollment, the akmods signing key is trusted by Secure Boot. All future NVIDIA module builds are automatically signed with this key. No re-enrollment needed.

### Verify installation

```bash
# Secure Boot still enabled:
mokutil --sb-state
# Expected: SecureBoot enabled

# NVIDIA module loaded:
lsmod | grep nvidia
# Expected: nvidia, nvidia_modeset, nvidia_drm, nvidia_uvm

# Module is signed:
modinfo nvidia | grep signer
# Expected: signer: <YOUR-AKMODS-KEY>

# GPU recognized:
nvidia-smi
# Expected: your GPU listed with driver version and temperature

# No unsigned module warnings:
dmesg | grep -i "module verification failed"
# Expected: no output
```

### How the signing chain works

```
Kernel update (dnf upgrade)
  │
  ├─ akmods detects new kernel version
  ├─ Compiles nvidia.ko from source against new kernel headers
  ├─ Signs nvidia.ko with private key from /etc/pki/akmods/private/
  ├─ Installs signed module in /lib/modules/<kernel>/extra/nvidia/
  └─ Next boot: kernel verifies signature via MOK → loads module
```

The signing infrastructure lives in `/etc/pki/akmods/`:

```
/etc/pki/akmods/
├── certs/       # Public certificates (enrolled in MOK)
├── private/     # Private signing key (root-only, used by akmods)
└── *.config     # OpenSSL configuration for certificate generation
```

> **Security**: The `private/` directory is root-only. The private key never leaves the system. Only akmods uses it during module compilation.

---

## 4. NVIDIA Wayland Configuration

### DRM Modesetting

NVIDIA requires DRM modesetting (`nvidia-drm.modeset=1`) for Wayland compositing. Since driver 580.x, this is the **compiled-in module default** — no boot parameter needed. RPM Fusion additionally sets it via `/etc/modprobe.d/nvidia.conf` for redundancy.

**No GRUB parameter required.** Two independent sources ensure modesetting is always active:

| Source | How it sets modesetting | Survives updates |
|--------|------------------------|-----------------|
| Module default | Compiled into `nvidia-drm.ko` since 580.x | Yes — part of driver binary |
| RPM Fusion | `options nvidia-drm modeset=1` in `/etc/modprobe.d/nvidia.conf` | Yes — RPM-managed |

### Verify

```bash
cat /sys/module/nvidia_drm/parameters/modeset
# Expected: Y

cat /sys/module/nvidia_drm/parameters/fbdev
# Expected: Y
```

> If either shows `N`, check that your NVIDIA driver is 580.x or newer (`modinfo nvidia | grep ^version`) and that `/etc/modprobe.d/nvidia.conf` exists (`cat /etc/modprobe.d/nvidia.conf`).

### Decision

```
NVIDIA + Wayland
├── nvidia-drm.modeset=1 (required — but automatic since driver 580.x):
│   ├── Enables kernel modesetting for NVIDIA
│   ├── Required for Wayland compositing
│   ├── Module default since 580.x + RPM Fusion modprobe.d redundancy
│   └── No GRUB parameter needed — verify with /sys/module check above
│
├── nvidia-drm.fbdev=1 (automatic since driver 580.x):
│   ├── Framebuffer device for early boot display
│   ├── Gives you a graphical boot splash (Plymouth)
│   └── Also module default + RPM Fusion redundancy
│
└── GSP firmware (automatic for 530+):
    ├── GPU System Processor offloads driver work to GPU
    ├── Loaded automatically from /lib/firmware/nvidia/
    └── No action needed — just be aware it exists
```

---

## 5. Accepted Risk: Proprietary GPU Stack

| Aspect | Detail |
|--------|--------|
| Code transparency | Open kernel module (since R515/2022, default since R560 for Turing+), but proprietary userspace |
| Ring 0 access | Kernel module has full kernel access — same as any driver |
| GPU firmware | Proprietary GSP firmware runs on GPU's own processor |
| Auditability | Kernel module: open source, auditable. Userspace + firmware: not auditable |

### Decision

```
GPU driver trust model
├── NVIDIA proprietary (akmod):
│   ├── Accept if: you need CUDA, VA-API HW decoding, or Wayland compositing
│   ├── Risk: proprietary userspace + firmware not auditable
│   ├── Mitigation: open kernel module, MOK-signed, locked down kernel
│   └── Alternative: nouveau (open source) — limited performance, no CUDA
│
├── AMD open source (mesa + amdgpu):
│   ├── Fully open source — kernel driver + userspace (mesa)
│   ├── No trust trade-off — everything is auditable
│   ├── No MOK enrollment needed (in-tree driver)
│   ├── Exception: AMDGPU PRO / ROCm (amdgpu-install) uses DKMS → needs MOK
│   └── Recommended if choosing hardware for a hardened system
│
└── Intel open source (mesa + i915/xe):
    ├── Fully open source — same as AMD
    ├── No MOK enrollment needed (integrated and discrete Arc — both in-tree)
    └── Zero trust issues — everything is auditable
```

---

## 6. Nouveau (Open Source NVIDIA Alternative)

### Decision

```
nouveau vs nvidia proprietary
├── Use nouveau if:
│   ├── You don't need CUDA or hardware video decoding
│   ├── You want a fully auditable GPU stack
│   ├── Your GPU is older (Kepler or earlier — good nouveau support)
│   └── Accept: lower performance, possible feature gaps
│
├── Use nvidia proprietary if:
│   ├── You need CUDA compute
│   ├── You need VA-API hardware video decoding
│   ├── You need full Wayland compositing performance
│   └── Your GPU is Turing or newer (best proprietary driver support)
│
└── Block nouveau when using nvidia proprietary:
    ├── Both drivers cannot coexist
    ├── RPM Fusion akmod-nvidia automatically blacklists nouveau
    ├── Verify: grep nouveau /etc/modprobe.d/*.conf
    └── Expected: "blacklist nouveau" present
```

### What if you use nouveau

If you use nouveau (no proprietary NVIDIA), you need **no MOK enrollment** and no RPM Fusion. The nouveau driver is in-tree and Fedora-signed. Secure Boot works without any additional configuration.

---

## 7. Complete Verification

```bash
# 1. Secure Boot enabled:
mokutil --sb-state
# Expected: SecureBoot enabled

# 2. Kernel lockdown integrity:
cat /sys/kernel/security/lockdown
# Expected: none [integrity] confidentiality

# 3. MOK keys enrolled (NVIDIA users):
mokutil --list-enrolled | grep -c "SHA1"
# Expected: 2 (Fedora CA + akmods key)

# 4. NVIDIA module signed (NVIDIA users):
modinfo nvidia | grep signer
# Expected: signer: <YOUR-AKMODS-KEY>

# 5. GPU functional:
# NVIDIA:
nvidia-smi
# AMD/Intel (requires mesa-utils: sudo dnf install mesa-utils):
glxinfo | grep "OpenGL renderer"

# 6. VA-API hardware decoding (requires libva-utils: sudo dnf install libva-utils):
vainfo 2>&1 | head -5
# Expected: profiles listed (H264, HEVC, VP9, AV1 depending on GPU)

# 7. No unsigned module warnings:
dmesg | grep -i "module verification failed"
# Expected: no output

# 8. Wayland active (not X11 fallback):
echo $XDG_SESSION_TYPE
# Expected: wayland

# 9. nouveau blocked (NVIDIA proprietary users only):
lsmod | grep nouveau
# Expected: no output (nouveau not loaded)
```

---

## Important Notes

- **kernel-devel must always be installed**: Without it, akmods cannot compile NVIDIA modules after kernel updates. Your system boots into software rendering. Install it: `sudo dnf install kernel-devel`.
- **Debug kernels break NVIDIA builds**: If `kernel-debug` is installed, akmods may fail to build the NVIDIA module. Remove it: `sudo dnf remove kernel-debug*`.
- **Wait before rebooting after kernel update**: Give akmods 2-5 minutes to build the new module. Check with `sudo akmods --check`.
- **AIDE database after kernel updates**: The NVIDIA module changes with each kernel update. Rebuild the AIDE database (→ Doc 13) to avoid false positives.
- **nouveau blacklist is automatic**: RPM Fusion's akmod-nvidia package creates `/etc/modprobe.d/blacklist-nouveau.conf` automatically.
- **MOK enrollment is one-time**: After the initial enrollment, all future akmods builds use the same key. No re-enrollment needed.
- **RPM Fusion repos required**: NVIDIA packages are not in Fedora's repos due to licensing. RPM Fusion Free (for ffmpeg) and Nonfree (for NVIDIA) are well-established community repos.
- **lockdown=integrity, not confidentiality**: Confidentiality mode can break NVIDIA drivers. Integrity mode provides strong security while maintaining GPU compatibility.

---

*Previous: [18 — Flatpak Sandboxing](18-flatpak-sandboxing.md)*
*Next: [20 — Snapshots](20-snapshots.md) — Btrfs snapshots for rollback and recovery*
