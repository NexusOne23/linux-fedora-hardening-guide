# 🔌 USBGuard

> Block-by-default USB policy with hash-based device identification.
> Applies to: Fedora Workstation 43+ | All hardware with USB ports
> No reboot required — service starts immediately.

---

## Overview

USBGuard controls USB device access at the kernel level. Only explicitly whitelisted devices are allowed — all unknown USB devices are **blocked** before they can interact with the system.

### Why

USB is the most accessible physical attack vector:
- **BadUSB**: A USB device that pretends to be a keyboard and types malicious commands
- **USB storage exfiltration**: Unauthorized data copy to/from removable media
- **USB network adapters**: Rogue network interfaces bypassing firewall rules
- **USB-based exploits**: Firmware-level attacks via malicious USB descriptors

USBGuard blocks all of this by default. Only devices you explicitly trust (identified by cryptographic hash, not just VID:PID) are allowed.

### Key features

| Feature | What it does |
|---------|-------------|
| `ImplicitPolicyTarget=block` | Unknown devices blocked — not just logged |
| Hash-based identification | Devices identified by cryptographic fingerprint — prevents VID:PID spoofing |
| Desktop notifications | `usbguard-notifier` shows GNOME notification when a device is blocked |
| Audit logging | Every USB connection logged to `/var/log/usbguard/usbguard-audit.log` |

---

## 1. Daemon Configuration

### File: `/etc/usbguard/usbguard-daemon.conf`

```bash
RuleFile=/etc/usbguard/rules.conf
RuleFolder=/etc/usbguard/rules.d/
ImplicitPolicyTarget=block
PresentDevicePolicy=apply-policy
PresentControllerPolicy=keep
InsertedDevicePolicy=apply-policy
RestoreControllerDeviceState=false
DeviceManagerBackend=uevent
IPCAllowedUsers=root <YOUR-USERNAME>
IPCAllowedGroups=wheel
IPCAccessControlFiles=/etc/usbguard/IPCAccessControl.d/
DeviceRulesWithPort=false
AuditBackend=FileAudit
AuditFilePath=/var/log/usbguard/usbguard-audit.log
HidePII=false
```

### Parameters

| Parameter | Value | Why |
|-----------|-------|-----|
| `ImplicitPolicyTarget` | `block` | **Unknown devices are blocked** — default would be `allow` |
| `PresentDevicePolicy` | `apply-policy` | On daemon start: check existing devices against rules |
| `PresentControllerPolicy` | `keep` | Keep USB controllers (hubs) on startup — prevents input loss |
| `InsertedDevicePolicy` | `apply-policy` | Newly plugged devices: check against rules |
| `RestoreControllerDeviceState` | `false` | Don't restore controller state — more secure |
| `IPCAllowedUsers` | `root <YOUR-USERNAME>` | Only root and your user can communicate with the daemon |
| `IPCAllowedGroups` | `wheel` | Wheel group can use IPC |
| `DeviceRulesWithPort` | `false` | Auto-generated rules don't include port binding (more flexible) |
| `AuditBackend` | `FileAudit` | Log audit events to file |
| `HidePII` | `false` | Serial numbers are logged — forensics over privacy for USB events |

### Decision

```
USBGuard policy strictness
├── ImplicitPolicyTarget=block (recommended):
│   ├── Unknown devices cannot interact with the system
│   ├── Must whitelist each new device manually
│   └── BadUSB protection — rogue devices are silently blocked
│
├── ImplicitPolicyTarget=reject:
│   ├── Like block, but sends USB reject signal
│   └── Device knows it was rejected (minor information leak)
│
└── PresentControllerPolicy=keep (important):
    ├── USB controllers (root hubs) are kept on daemon start
    ├── Without this: keyboard/mouse may stop working after boot
    └── Only affects controllers, not regular devices
```

---

## 2. Whitelist Rules

### How rules work

Each rule in `/etc/usbguard/rules.conf` contains:
- **VID:PID** (Vendor ID : Product ID) — identifies the device type
- **Hash** — cryptographic fingerprint of the device descriptors (prevents spoofing)
- **Interface class** — what the device claims to be (HID, storage, etc.)

### Example rules (your devices will differ)

```bash
# USB Root Hubs (always needed)
allow id 1d6b:0002 hash "..." name "xHCI Host Controller" with-interface 09:00:00
allow id 1d6b:0003 hash "..." name "xHCI Host Controller" with-interface 09:00:00

# Example: Keyboard/Mouse receiver
allow id 046d:c52b hash "..." name "USB Receiver" with-interface { 03:01:01 03:01:02 03:00:00 }

# Example: USB flash drive
allow id 0781:5591 hash "..." name "Ultra" with-interface 08:06:50
```

> **These are examples.** Your rules will contain YOUR specific devices with their unique hashes. Use `usbguard generate-policy` to create your initial ruleset.

### Hash-based identification

Every rule contains a `hash` value — a cryptographic fingerprint computed from the device's USB descriptors. This prevents an attacker from plugging in a device with a spoofed VID:PID (e.g., a BadUSB pretending to be your keyboard). The hash must match exactly.

### Common USB interface classes

| Interface | Class | Meaning |
|-----------|-------|---------|
| `09:00:00` | Hub | USB Hub / Root Controller |
| `03:01:01` | HID Boot Keyboard | Keyboard |
| `03:01:02` | HID Boot Mouse | Mouse |
| `03:00:00` | HID Generic | Generic HID device |
| `08:06:50` | Mass Storage (SCSI) | USB storage devices |
| `06:01:01` | Still Image (MTP) | Media Transfer Protocol (phones) |
| `ff:xx:xx` | Vendor-specific | Proprietary protocols (varies by device) |

---

## 3. Managing Devices

### When a new device is plugged in

1. USBGuard blocks it (ImplicitPolicyTarget=block)
2. `usbguard-notifier` shows a GNOME desktop notification
3. You decide: allow temporarily or whitelist permanently

### Commands

```bash
# List all devices (allowed and blocked):
$ sudo usbguard list-devices

# List only blocked devices:
$ sudo usbguard list-devices --blocked

# Allow a blocked device temporarily (until unplugged):
$ sudo usbguard allow-device <DEVICE-NUMBER>

# Whitelist permanently (hash-based rule added to rules.conf):
$ sudo usbguard allow-device <DEVICE-NUMBER> --permanent

# Block a previously allowed device:
$ sudo usbguard block-device <DEVICE-NUMBER>

# List current rules:
$ sudo usbguard list-rules
```

### Decision

```
When to use --permanent
├── Permanent if:   Device you use regularly (keyboard, mouse, flash drive, phone)
├── Temporary if:   One-time device (borrowed flash drive, charging cable)
└── Never allow:    Unknown devices you didn't plug in yourself
```

---

## 🔧 Applying Changes

### Step 1: Install USBGuard

```bash
$ sudo dnf install usbguard usbguard-selinux usbguard-notifier
```

### Step 2: Generate initial policy from connected devices

**Important:** Connect all your regular USB devices (keyboard, mouse, etc.) BEFORE generating the policy.

```bash
$ sudo usbguard generate-policy > /tmp/rules.conf
$ sudo cp /tmp/rules.conf /etc/usbguard/rules.conf
```

### Step 3: Configure the daemon

```bash
$ sudo tee /etc/usbguard/usbguard-daemon.conf << 'USBGUARD'
RuleFile=/etc/usbguard/rules.conf
RuleFolder=/etc/usbguard/rules.d/
ImplicitPolicyTarget=block
PresentDevicePolicy=apply-policy
PresentControllerPolicy=keep
InsertedDevicePolicy=apply-policy
RestoreControllerDeviceState=false
DeviceManagerBackend=uevent
IPCAllowedUsers=root <YOUR-USERNAME>
IPCAllowedGroups=wheel
IPCAccessControlFiles=/etc/usbguard/IPCAccessControl.d/
DeviceRulesWithPort=false
AuditBackend=FileAudit
AuditFilePath=/var/log/usbguard/usbguard-audit.log
HidePII=false
USBGUARD
```

> **Edit** the file to replace `<YOUR-USERNAME>` with your actual username.

### Step 4: Enable and start

```bash
$ sudo systemctl enable --now usbguard
```

### Step 5: Whitelist additional devices as needed

```bash
# Plug in a new device → it gets blocked
$ sudo usbguard list-devices --blocked
# Note the device number

# Whitelist permanently:
$ sudo usbguard allow-device <DEVICE-NUMBER> --permanent
```

---

## ✅ Complete Verification

```bash
# 1. Service active:
$ systemctl is-active usbguard
# Expected: active

# 2. ImplicitPolicyTarget=block:
$ sudo grep "ImplicitPolicyTarget" /etc/usbguard/usbguard-daemon.conf
# Expected: ImplicitPolicyTarget=block

# 3. Rules exist:
$ sudo usbguard list-rules
# Expected: Your whitelisted devices with hashes

# 4. Connected devices all allowed:
$ sudo usbguard list-devices
# Expected: All showing "allow", no "block"

# 5. Test: Plug in an unknown USB device
# Expected: Blocked, notification appears, shows as "block" in list-devices

# 6. Audit log exists:
$ sudo tail -5 /var/log/usbguard/usbguard-audit.log
# Expected: Log entries present
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| `ImplicitPolicyTarget=block` | Must whitelist every new device | Unknown USB devices cannot attack |
| Hash-based rules | Device replacement requires new rule | Prevents VID:PID spoofing (BadUSB) |
| `HidePII=false` | Serial numbers in logs | Full forensic trail for USB events |
| `PresentControllerPolicy=keep` | Controllers not re-validated on restart | Keyboard/mouse always work after boot |

---

## Notes

- **USB keyboard/mouse must be whitelisted** before enabling USBGuard — otherwise you lose input after boot. The `generate-policy` step (Step 2) handles this by including all currently connected devices.
- **`PresentControllerPolicy=keep`** is critical. Without it, USB root hubs are re-evaluated on daemon start — if the rule is missing or changed, your keyboard and mouse stop working.
- **Devices with multiple modes** (e.g., phones with MTP + ADB) need separate rules for each mode, as they present different VID:PID and interface classes depending on the selected mode.
- **`usbguard-notifier`** runs as a user service via XDG autostart. It provides desktop notifications when devices are blocked — you don't have to watch the terminal.
- **Hash changes on firmware update.** If a device receives a firmware update that changes its USB descriptors, USBGuard will block it. Re-whitelist with `--permanent`.
- **SELinux integration:** The `usbguard-selinux` package provides the SELinux policy for USBGuard. Install it alongside `usbguard`.

---

*Previous: [13 — AIDE](13-aide.md)*
*Next: [15 — Intel ME Mitigation](15-intel-me-mitigation.md) — Multi-layer mitigation for Intel Management Engine*
