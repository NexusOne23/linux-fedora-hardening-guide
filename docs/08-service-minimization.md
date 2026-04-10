# ⛔ Service Minimization

> Attack surface reduction: 53 masked system units + 18 masked user units + 2 D-Bus overrides = 73 disabled components.
> Applies to: Fedora Workstation 43+ | GNOME desktop
> No reboot required — changes apply immediately.

---

## Overview

Every running service is a potential attack vector. Masked services cannot be started — not manually, not by socket activation, not as a dependency. This document identifies 73 components that serve no purpose on a hardened desktop and can be safely disabled.

### Summary

| Category | Count | Type |
|----------|-------|------|
| System units | 53 | `systemctl mask` |
| User units | 18 | `systemctl --user mask` |
| D-Bus overrides | 2 | `Exec=/bin/false` override files |
| **Total** | **73** | |

> **Important:** Review each category against YOUR hardware and needs. Some services (Bluetooth, CUPS, Wi-Fi) may be required on your system. The decision trees below help you decide.

---

## 1. Masked System Units (53)

### 1.1 ABRT — Automatic Bug Reporting (5 units)

| Unit | Default |
|------|---------|
| `abrtd.service` | enabled |
| `abrt-journal-core.service` | enabled |
| `abrt-oops.service` | enabled |
| `abrt-vmcore.service` | enabled |
| `abrt-xorg.service` | enabled |

```
ABRT (Automatic Bug Reporting Tool)
├── Mask if:    You don't want crash reports sent to Fedora (privacy)
├── Keep if:    You want to contribute crash data to Fedora QA
└── What breaks: No automatic crash reporting — you won't be prompted after crashes
```

### 1.2 Avahi — mDNS/Zeroconf (2 units)

| Unit | Default |
|------|---------|
| `avahi-daemon.service` | enabled |
| `avahi-daemon.socket` | enabled |

```
Avahi (mDNS/Zeroconf)
├── Mask if:    You don't need automatic network service discovery (recommended)
├── Keep if:    You use mDNS to find printers, Chromecast, or AirPlay devices
└── What breaks: No automatic discovery of LAN services (printers, media devices)
                 Manual IP configuration still works for all devices
```

### 1.3 Bluetooth (1 unit)

| Unit | Default |
|------|---------|
| `bluetooth.service` | enabled |

```
Bluetooth
├── Mask if:    No Bluetooth hardware or no Bluetooth needed
├── Keep if:    You use Bluetooth headphones, keyboard, mouse, or other devices
└── What breaks: All Bluetooth functionality
```

### 1.4 CUPS — Printing (3 units)

| Unit | Default |
|------|---------|
| `cups.service` | disabled |
| `cups.socket` | enabled |
| `cups.path` | enabled |

```
CUPS (Printing)
├── Mask if:    No printer connected or used (recommended — CUPS has CVE history)
├── Keep if:    You print to a local or network printer
├── Note:       Mask ALL THREE units — socket/path activation will restart the service
└── What breaks: All printing functionality
```

### 1.5 Smartcard / Fingerprint (3 units)

| Unit | Default |
|------|---------|
| `pcscd.service` | disabled |
| `pcscd.socket` | enabled |
| `fprintd.service` | disabled |

```
Smartcard + Fingerprint
├── Mask if:    No smartcard reader and no fingerprint reader
├── Keep pcscd if:  You use a YubiKey or smartcard for authentication
├── Keep fprintd if: You use fingerprint login
└── What breaks: Smartcard/fingerprint authentication
```

### 1.6 SSSD — Directory Services (3 units)

| Unit | Default |
|------|---------|
| `sssd.service` | enabled |
| `sssd-kcm.service` | disabled |
| `sssd-kcm.socket` | enabled |

```
SSSD (System Security Services Daemon)
├── Mask if:    Standalone desktop — no LDAP, Active Directory, or Kerberos
├── Keep if:    Your system authenticates against a directory server
└── What breaks: LDAP/AD/Kerberos authentication, centralized user management
```

### 1.7 ModemManager / Wi-Fi (2 units)

| Unit | Default |
|------|---------|
| `ModemManager.service` | enabled |
| `wpa_supplicant.service` | disabled |

```
ModemManager + Wi-Fi
├── Mask ModemManager if:     No mobile broadband modem (3G/4G/5G)
├── Mask wpa_supplicant if:   Ethernet only — no Wi-Fi hardware
├── Keep wpa_supplicant if:   You connect via Wi-Fi
└── What breaks: Mobile broadband / Wi-Fi connectivity
```

### 1.8 iSCSI — Network Storage (4 units)

| Unit | Default |
|------|---------|
| `iscsi-onboot.service` | enabled |
| `iscsi-starter.service` | enabled |
| `iscsid.socket` | enabled |
| `iscsiuio.socket` | enabled |

```
iSCSI
├── Mask if:    Always — desktop systems don't use iSCSI network storage
├── Keep if:    You boot from or mount iSCSI targets (extremely rare on desktop)
└── What breaks: iSCSI storage functionality
```

### 1.9 VM Guest Tools (4 units)

| Unit | Default |
|------|---------|
| `qemu-guest-agent.service` | enabled |
| `vboxservice.service` | enabled |
| `vgauthd.service` | enabled |
| `vmtoolsd.service` | enabled |

```
VM Guest Tools
├── Mask if:    Running on bare metal hardware (not inside a VM)
├── Keep if:    Running Fedora inside a virtual machine
└── What breaks: Host-guest integration (clipboard, display resize, etc.)
```

### 1.10 Core Dump Socket (1 unit)

| Unit | Default |
|------|---------|
| `systemd-coredump.socket` | disabled |

```
Core Dump
├── Mask if:    Always recommended — core dumps are disabled system-wide anyway
├── Note:       Complements sysctl fs.suid_dumpable=0 and ulimit core=0
└── What breaks: No core dumps for crash analysis (use journalctl instead)
```

### 1.11 NVIDIA Power Daemon (1 unit, conditional)

| Unit | Default |
|------|---------|
| `nvidia-powerd.service` | enabled |

```
NVIDIA Dynamic Boost
├── Mask if:    Desktop system (Dynamic Boost is laptop-only)
├── Keep if:    NVIDIA laptop with Dynamic Boost support
├── Skip if:    No NVIDIA GPU — this service doesn't exist
└── What breaks: Nothing on desktop — Dynamic Boost is for laptop GPU/CPU power sharing
```

### 1.12 General Purpose (11 units)

| Unit | Default | Why mask |
|------|---------|---------|
| `atd.service` | enabled | `at` job scheduler — cron/systemd timers are sufficient |
| `gssproxy.service` | disabled | GSS-Proxy for Kerberos — no Kerberos |
| `livesys.service` | enabled | Live system setup — not relevant on installed system |
| `livesys-late.service` | enabled | Live system setup — not relevant on installed system |
| `passim.service` | disabled | Peer-to-peer content distribution — privacy concern |
| `rpc-statd-notify.service` | disabled | NFS status notifications — no NFS |
| `wsdd.service` | disabled | Web Services Discovery — triple-blocked by firewall ([Doc 03](03-firewall.md)) |
| `wsdd2.service` | disabled | Web Services Discovery v2 — not needed |
| `geoclue.service` | static | Location service — no app needs GPS/location, privacy risk |
| `systemd-homed.service` | static | Portable home directories — not used, local accounts via `/etc/passwd` |
| `packagekit.service` | enabled | Background package management — use `dnf` manually instead |

> **packagekit masked:** Automatic background metadata fetches are disabled. System updates must be performed manually via `sudo dnf upgrade`. See [Doc 25](25-update-process.md).

### 1.13 libvirt Network and QEMU Sockets (13 units, conditional)

> **Skip this category** if libvirt/QEMU is not installed.

| Unit | Purpose |
|------|---------|
| `virtnetworkd.socket` / `-ro.socket` / `-admin.socket` | Virtual network bridge management |
| `virtnwfilterd.socket` / `-ro.socket` / `-admin.socket` | Network filter daemon |
| `virtinterfaced.socket` / `-ro.socket` / `-admin.socket` | Interface management daemon |
| `virtqemud.service` / `.socket` / `-ro.socket` / `-admin.socket` | Monolithic QEMU daemon (replaced by modular libvirtd) |

```
libvirt modular daemons
├── Mask if:    You use libvirtd (monolithic) or don't need VM networking
├── Keep if:    You specifically use the modular libvirt daemon architecture
├── Note:       VM basic functionality (QEMU/KVM) remains via libvirtd
└── What breaks: Modular daemon socket activation — VMs still work via libvirtd
```

---

## 2. Masked User Units (18)

These are per-user services masked with `systemctl --user mask`.

### 2.1 Evolution — Groupware (5 units)

| Unit | Why mask |
|------|---------|
| `evolution-addressbook-factory.service` | Address book service |
| `evolution-calendar-factory.service` | Calendar service |
| `evolution-source-registry.service` | Account registry |
| `evolution-alarm-notify.service` | Calendar alarms |
| `evolution-user-prompter.service` | User prompts |

```
Evolution
├── Mask if:    You don't use Evolution email/calendar (recommended)
├── Keep if:    You use Evolution as your email/calendar client
└── What breaks: Evolution data services — no address book, calendar, or account sync
```

### 2.2 LocalSearch / Tracker — File Indexing (3 units)

| Unit | Why mask |
|------|---------|
| `localsearch-3.service` | File content indexer |
| `localsearch-control-3.service` | Indexer control daemon |
| `localsearch-writeback-3.service` | Metadata writeback (modifies files) |

```
LocalSearch (formerly Tracker)
├── Mask if:    You don't need file content search in Nautilus (recommended)
├── Keep if:    You rely on GNOME file search to find documents by content
├── Note:       localsearch-writeback modifies file metadata — privacy concern
└── What breaks: File content search in Nautilus, GNOME Photos metadata
```

### 2.3 Accessibility (1 unit)

| Unit | Why mask |
|------|---------|
| `at-spi-dbus-bus.service` | Screen reader D-Bus bridge |

```
Accessibility (AT-SPI)
├── Mask if:    You don't use screen readers or accessibility features
├── Keep if:    You use Orca, screen magnification, or other AT tools
└── What breaks: Screen reader, accessibility API for applications
```

> **Caution:** If you or anyone using your system needs accessibility features, do NOT mask this.

### 2.4 gvfs Volume Monitors (3 units)

| Unit | Why mask |
|------|---------|
| `gvfs-afc-volume-monitor.service` | Apple AFC protocol (iPhone/iPad) |
| `gvfs-goa-volume-monitor.service` | GNOME Online Accounts volumes |
| `gvfs-gphoto2-volume-monitor.service` | PTP camera protocol |

```
gvfs Volume Monitors
├── Mask all if:       You don't connect iPhones, cameras, or use GNOME Online Accounts
├── Keep afc if:       You connect iOS devices via USB
├── Keep gphoto2 if:   You connect cameras via USB (PTP mode)
├── Keep goa if:       You use GNOME Online Accounts for cloud storage
└── What breaks: Automatic mounting of the respective device types in Nautilus
```

### 2.5 GNOME Settings Daemon Plugins (5 units)

| Unit | Why mask |
|------|---------|
| `org.gnome.SettingsDaemon.PrintNotifications.service` | Print job notifications |
| `org.gnome.SettingsDaemon.Rfkill.service` | Wi-Fi/Bluetooth hardware kill switch |
| `org.gnome.SettingsDaemon.Smartcard.service` | Smartcard events |
| `org.gnome.SettingsDaemon.Wwan.service` | Mobile broadband panel |
| `org.gnome.SettingsDaemon.Sharing.service` | Network file/media sharing |

```
GNOME Settings Daemon Plugins
├── PrintNotifications:  Mask if no printer
├── Rfkill:              Mask if no Wi-Fi/Bluetooth hardware
├── Smartcard:           Mask if no smartcard reader
├── Wwan:                Mask if no mobile broadband modem
├── Sharing:             Mask always — network sharing is a privacy/security risk
└── What breaks: Respective GNOME Settings panel functionality
```

### 2.6 GNOME Software (1 unit)

| Unit | Why mask |
|------|---------|
| `gnome-software.service` | Background app store updates |

```
GNOME Software
├── Mask if:    You manage updates manually via dnf (recommended)
├── Keep if:    You want automatic update notifications and Flatpak management via GUI
└── What breaks: GNOME Software background update checks and notifications
```

---

## 3. D-Bus Service Overrides (2)

### What

Some GNOME services are transient D-Bus-activated units — they are not started by systemd but by D-Bus activation. `systemctl --user mask` does not work for these. Instead, override files with `Exec=/bin/false` prevent them from starting.

### Why

GNOME Online Accounts (GOA) and its identity service connect to external cloud providers (Google, Microsoft, etc.) and maintain persistent connections. On a hardened desktop without cloud accounts, these are unnecessary network activity and a privacy risk.

### Override files

**File: `~/.local/share/dbus-1/services/org.gnome.OnlineAccounts.service`**

```ini
[D-BUS Service]
Name=org.gnome.OnlineAccounts
Exec=/bin/false
```

**File: `~/.local/share/dbus-1/services/org.gnome.Identity.service`**

```ini
[D-BUS Service]
Name=org.gnome.Identity
Exec=/bin/false
```

### How it works

When any application tries to activate `org.gnome.OnlineAccounts` via D-Bus, the override file redirects to `/bin/false` which exits immediately. No daemon starts.

The path `~/.local/share/dbus-1/services/` (user scope) takes precedence over `/usr/share/dbus-1/services/` (system scope) and survives DNF updates.

### Decision

```
GNOME Online Accounts
├── Override if:  You don't use Google/Microsoft/cloud accounts in GNOME apps
├── Keep if:      You use GNOME Calendar, Contacts, or Files with cloud providers
└── What breaks:  GNOME apps cannot access cloud accounts (email, calendar, files)
```

---

## 4. Services Intentionally NOT Masked

These services were evaluated but kept active — they are GNOME/Fedora dependencies:

| Service | Why kept |
|---------|---------|
| `accounts-daemon.service` | Required by GDM and GNOME Control Center for user management |
| `low-memory-monitor.service` | OOM handling — informs desktop of memory pressure. No network, no privacy risk |

---

## 🔧 Applying Changes

### Step 1: Mask system units

Adjust the list based on your decision trees above. Remove any unit you need to keep:

```bash
$ sudo systemctl mask \
    abrtd.service abrt-journal-core.service abrt-oops.service \
    abrt-vmcore.service abrt-xorg.service \
    avahi-daemon.service avahi-daemon.socket \
    bluetooth.service \
    cups.service cups.socket cups.path \
    pcscd.service pcscd.socket fprintd.service \
    sssd.service sssd-kcm.service sssd-kcm.socket \
    ModemManager.service wpa_supplicant.service \
    iscsi-onboot.service iscsi-starter.service iscsid.socket iscsiuio.socket \
    qemu-guest-agent.service vboxservice.service vgauthd.service vmtoolsd.service \
    systemd-coredump.socket \
    atd.service gssproxy.service livesys.service livesys-late.service \
    passim.service rpc-statd-notify.service wsdd.service wsdd2.service \
    geoclue.service systemd-homed.service packagekit.service
```

**Conditional — add if applicable:**

```bash
# NVIDIA desktop (not laptop):
$ sudo systemctl mask nvidia-powerd.service

# libvirt installed but modular daemons not needed:
$ sudo systemctl mask \
    virtnetworkd.socket virtnetworkd-ro.socket virtnetworkd-admin.socket \
    virtnwfilterd.socket virtnwfilterd-ro.socket virtnwfilterd-admin.socket \
    virtinterfaced.socket virtinterfaced-ro.socket virtinterfaced-admin.socket \
    virtqemud.service virtqemud.socket virtqemud-ro.socket virtqemud-admin.socket
```

### Step 2: Mask user units

```bash
$ systemctl --user mask \
    evolution-addressbook-factory.service evolution-calendar-factory.service \
    evolution-source-registry.service evolution-alarm-notify.service \
    evolution-user-prompter.service \
    localsearch-3.service localsearch-control-3.service localsearch-writeback-3.service \
    at-spi-dbus-bus.service \
    gvfs-afc-volume-monitor.service gvfs-goa-volume-monitor.service \
    gvfs-gphoto2-volume-monitor.service \
    org.gnome.SettingsDaemon.PrintNotifications.service \
    org.gnome.SettingsDaemon.Rfkill.service \
    org.gnome.SettingsDaemon.Smartcard.service \
    org.gnome.SettingsDaemon.Wwan.service \
    org.gnome.SettingsDaemon.Sharing.service \
    gnome-software.service
```

### Step 3: Create D-Bus overrides

```bash
$ mkdir -p ~/.local/share/dbus-1/services

$ tee ~/.local/share/dbus-1/services/org.gnome.OnlineAccounts.service << 'EOF'
[D-BUS Service]
Name=org.gnome.OnlineAccounts
Exec=/bin/false
EOF

$ tee ~/.local/share/dbus-1/services/org.gnome.Identity.service << 'EOF'
[D-BUS Service]
Name=org.gnome.Identity
Exec=/bin/false
EOF
```

---

## ✅ Complete Verification

```bash
# 1. System units masked (count depends on your selections):
$ systemctl list-unit-files --state=masked --no-pager | grep -c masked
# Expected: ~39 (base) to ~53 (with NVIDIA + libvirt)

# 2. User units masked:
$ systemctl --user list-unit-files --state=masked --no-pager | grep -c masked
# Expected: 18

# 3. Key services are NOT running:
$ for svc in avahi-daemon bluetooth cups ModemManager sssd abrtd; do
    systemctl is-active $svc.service 2>/dev/null && \
        echo "WARNING: $svc running!" || echo "OK: $svc inactive"
done
# Expected: All OK

# 4. D-Bus overrides exist:
$ ls ~/.local/share/dbus-1/services/
# Expected: org.gnome.Identity.service  org.gnome.OnlineAccounts.service

# 5. No GNOME Online Accounts process:
$ pgrep -a goa 2>/dev/null || echo "OK: no GOA process"
# Expected: OK
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| ABRT masked | No automatic crash reporting to Fedora | No data sent externally |
| Avahi masked | No automatic LAN service discovery | mDNS cannot leak presence |
| CUPS masked | No printing | Removes CVE-prone service |
| packagekit masked | No background update checks | No automatic network metadata fetches |
| LocalSearch masked | No file content search in Nautilus | No file indexing (performance + privacy) |
| GOA overridden | No cloud account integration in GNOME | No external connections to cloud providers |
| at-spi masked | No screen reader support | Reduces D-Bus attack surface |

---

## Notes

- **`systemctl mask`** creates a symlink to `/dev/null`. The service cannot start — not manually, not by socket activation, not as a dependency. To undo: `systemctl unmask <unit>`.
- **Socket activation:** Services like CUPS and SSSD-KCM are activated via sockets. Masking only the `.service` is insufficient — mask the `.socket` (and `.path` for CUPS) too.
- **User units** (`systemctl --user mask`) apply only to the current user. On multi-user systems, each user must mask separately.
- **D-Bus override path:** `~/.local/share/dbus-1/services/` overrides `/usr/share/dbus-1/services/`. Survives system updates.
- **gnome-software masked:** Automatic update notifications are disabled. Use `sudo dnf upgrade` for system updates.
- **Fedora updates may re-enable services.** After major Fedora upgrades, verify that masked services are still masked.

---

*Previous: [07 — IPv6 Hardening](07-ipv6-hardening.md)*
*Next: [09 — SSH Hardening](09-ssh-hardening.md) — Key-only authentication, service disabled by default*
