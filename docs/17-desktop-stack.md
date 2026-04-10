# 🖥️ Desktop Stack Hardening (GNOME/Wayland)

> Harden the GNOME desktop: Wayland isolation, masked services, D-Bus overrides, privacy settings.
> Applies to: Fedora Workstation 43+ | GNOME 47+ | Wayland session
> No reboot required — changes apply immediately or on next login.

---

## Overview

The GNOME desktop exposes attack surface through background services, D-Bus activation, media handling, and display protocol features. This document hardens the desktop layer: enforce Wayland isolation, mask unnecessary user services, override D-Bus service activation, and configure privacy-relevant gsettings.

### Why

A default GNOME installation runs dozens of user-space services you likely don't need — Evolution calendar sync, file indexing, online accounts, desktop sharing. Each is a potential attack surface. The display server choice (Wayland vs X11) determines whether applications can spy on each other's input and screen content.

### What you get

| Layer | Hardening | Effect |
|-------|-----------|--------|
| Display server | Wayland-only session | Input isolation between applications |
| Xwayland | Auto-close when idle | X11 compatibility layer only active when needed |
| User services | 16–18 units masked (hardware-dependent) | Reduced background attack surface |
| D-Bus | 2 service overrides | Cloud accounts and Kerberos blocked at activation |
| gsettings | 17 privacy settings | No autorun, no thumbnails, no tracking, lockscreen hardened |

---

## 1. Display Server: Wayland

### Why Wayland matters for security

| Feature | X11 | Wayland |
|---------|-----|---------|
| Keylogging | Any app can read all keystrokes | Isolated — apps see only their own input |
| Screenshots | Any app can read the entire screen | Only with portal permission |
| Clipboard | Globally readable by all apps | Only the focused app has access |
| Input injection | Any app can simulate keyboard/mouse | Not possible |

X11 has **zero isolation** between applications. Any X11 app can keylog, screenshot, and inject input into any other app. Wayland isolates each application by design.

### Verify

```bash
echo $XDG_SESSION_TYPE
# Expected: wayland
```

Fedora Workstation defaults to Wayland since Fedora 25. If you see `x11`, check if a proprietary GPU driver is forcing X11 (older NVIDIA drivers did this — NVIDIA 545+ supports Wayland natively).

### Decision

```
Display server
├── Wayland (default on Fedora — keep):
│   ├── Full input/output isolation between apps
│   ├── Portal-based permission model for screenshots, screencasts
│   └── Native support for GNOME 47+
│
├── X11 (avoid unless required):
│   ├── Only if: legacy app absolutely requires X11 (rare in 2025+)
│   ├── Risk: zero isolation — any app can keylog/screenshot all others
│   └── If needed: use Xwayland (automatic) rather than full X11 session
│
└── GDM Wayland enforcement:
    ├── Default on Fedora — no action needed
    ├── If /etc/gdm/custom.conf contains WaylandEnable=false → remove it
    └── NVIDIA users: ensure nvidia-drm.modeset=1 in boot params (→ Doc 01)
```

---

## 2. Xwayland Auto-Close

### What

Xwayland is an X11 compatibility layer that runs inside your Wayland session. It starts automatically when any X11 app launches — but by default stays running even after all X11 apps close.

### Why

While Xwayland is running, the X11 security model (no isolation) applies to all X11 apps. Auto-closing Xwayland when no X11 app is running eliminates this attack surface during normal use.

### How

```bash
# Check current experimental features
gsettings get org.gnome.mutter experimental-features

# Add autoclose-xwayland (preserve your existing features)
gsettings set org.gnome.mutter experimental-features \
  "['autoclose-xwayland']"
```

> **Note**: If you already have other experimental features enabled (like `scale-monitor-framebuffer` for fractional scaling), include them in the array:
> ```bash
> gsettings set org.gnome.mutter experimental-features \
>   "['scale-monitor-framebuffer', 'autoclose-xwayland']"
> ```

### Verify

```bash
gsettings get org.gnome.mutter experimental-features
# Expected: contains 'autoclose-xwayland'
```

### What breaks

Nothing in normal use. Xwayland still starts automatically when any X11 app launches. It just doesn't linger after the last X11 app closes.

> **Note**: Some older Electron apps use X11 via Xwayland. Modern Electron (28+) supports Wayland natively with `--ozone-platform=wayland`.

### Undo

```bash
# Remove autoclose-xwayland from the array (keep other features)
gsettings set org.gnome.mutter experimental-features "[]"
```

---

## 3. GDM (Display Manager)

### File: `/etc/gdm/custom.conf`

Fedora defaults are secure — all sections empty:

```ini
[daemon]
[security]
[xdmcp]
[chooser]
[debug]
```

### Verify

| Feature | Expected | Risk if wrong |
|---------|----------|--------------|
| Wayland | Enabled (default) | X11 session has zero app isolation |
| XDMCP | Disabled (empty section) | Remote login protocol — must be off |
| Automatic login | Disabled (no `AutomaticLogin=` line) | Bypasses authentication entirely |
| User list | Shown (default) | Acceptable on single-user desktop |

### Decision

```
GDM configuration
├── Single-user desktop (typical):
│   ├── Keep defaults — no changes needed
│   └── User list visible is acceptable
│
├── Multi-user or shared machine:
│   ├── Hide user list via dconf:
│   │   sudo mkdir -p /etc/dconf/db/gdm.d
│   │   echo -e '[org/gnome/login-screen]\ndisable-user-list=true' | \
│   │     sudo tee /etc/dconf/db/gdm.d/00-login-screen
│   │   sudo dconf update
│   └── Prevents username enumeration at login screen
│
└── Never enable:
    ├── AutomaticLogin — bypasses all authentication
    ├── XDMCP — remote display protocol, unencrypted
    └── WaylandEnable=false — forces insecure X11 session
```

---

## 4. Masked User Services

### Why mask services

Every running service is attack surface. GNOME starts many user-space services via D-Bus activation and systemd user units. Services you don't use should be masked — not just stopped (masking prevents them from being started by other services or D-Bus activation).

### How

```bash
systemctl --user mask \
  evolution-addressbook-factory.service \
  evolution-alarm-notify.service \
  evolution-calendar-factory.service \
  evolution-source-registry.service \
  evolution-user-prompter.service \
  localsearch-3.service \
  localsearch-control-3.service \
  localsearch-writeback-3.service \
  org.gnome.SettingsDaemon.PrintNotifications.service \
  org.gnome.SettingsDaemon.Sharing.service \
  org.gnome.SettingsDaemon.Smartcard.service \
  at-spi-dbus-bus.service \
  gvfs-afc-volume-monitor.service \
  gvfs-goa-volume-monitor.service \
  gvfs-gphoto2-volume-monitor.service \
  gnome-software.service
```

### Decision per service category

```
Evolution (5 services):
  evolution-addressbook-factory, evolution-alarm-notify,
  evolution-calendar-factory, evolution-source-registry,
  evolution-user-prompter
├── Mask if:  You don't use Evolution for email/calendar/contacts
├── Keep if:  Evolution is your email/calendar client
└── What breaks: Evolution stops working (can't be uninstalled — GNOME core dependency)
    Masking is the correct approach since removal breaks GNOME.

LocalSearch / Tracker (3 services):
  localsearch-3, localsearch-control-3, localsearch-writeback-3
├── Mask if:  You don't need full-text file search in GNOME Files
├── Keep if:  You rely on GNOME Files search to find files by content
└── What breaks: GNOME Files search becomes filename-only (no content search).
    File browsing and manual search still work normally.

Print Notifications:
  org.gnome.SettingsDaemon.PrintNotifications
├── Mask if:  No printer connected / CUPS already masked (→ Doc 08)
├── Keep if:  You use a printer
└── What breaks: No print job notifications

Desktop Sharing:
  org.gnome.SettingsDaemon.Sharing
├── Mask if:  No VNC/RDP remote desktop needed (recommended)
├── Keep if:  You use GNOME Remote Desktop for incoming connections
└── What breaks: GNOME screen sharing / remote desktop

Smartcard:
  org.gnome.SettingsDaemon.Smartcard
├── Mask if:  No smartcard reader / pcscd already masked (→ Doc 08)
├── Keep if:  You use smartcard authentication (PIV, PKCS#11)
└── What breaks: Smartcard detection and authentication

Accessibility (at-spi-dbus-bus):
├── Mask if:  No screen reader / accessibility tools needed
├── Keep if:  You use Orca, screen magnifier, or other assistive technology
└── What breaks: ALL accessibility features — screen readers, high contrast
    switchers, keyboard accessibility. Only mask if certain.

GVFS volume monitors:
  gvfs-afc-volume-monitor     — Apple iOS devices (AFC protocol)
  gvfs-goa-volume-monitor     — GNOME Online Accounts cloud storage
  gvfs-gphoto2-volume-monitor — PTP cameras
├── Mask if:  No iOS devices / no GNOME Online Accounts / no PTP cameras
├── Keep selectively: only keep the ones matching your hardware
└── What breaks: Auto-detection of the respective device type

GNOME Software (gnome-software):
├── Mask if:  You manage packages via dnf CLI (recommended)
├── Keep if:  You prefer a graphical software center
└── What breaks: No GUI software center. dnf works normally.
```

### Hardware-dependent services (mask only if hardware absent)

```
WiFi/Bluetooth RF kill:
  org.gnome.SettingsDaemon.Rfkill
├── Mask if:  No WiFi/Bluetooth hardware (desktop without wireless cards)
├── Keep if:  You have WiFi or Bluetooth (even if currently disabled)
└── What breaks: RF kill toggle in GNOME Settings stops working

Mobile broadband:
  org.gnome.SettingsDaemon.Wwan
├── Mask if:  No WWAN/cellular modem (most desktops)
├── Keep if:  Laptop with built-in LTE/5G modem
└── What breaks: Cellular data connection management
```

### Verify

```bash
systemctl --user list-unit-files --state=masked --no-legend | wc -l
# Expected: number matches how many you masked

# Check specific service:
systemctl --user is-enabled evolution-source-registry.service
# Expected: masked
```

### Undo

```bash
# Unmask a single service:
systemctl --user unmask evolution-source-registry.service

# Unmask all at once (paste the same list with unmask instead of mask)
```

---

## 5. D-Bus Service Overrides

### What

D-Bus is GNOME's inter-process communication system. Services can be auto-started ("activated") when any application requests them over D-Bus. This happens even if you've stopped or disabled the systemd unit.

### Why

Two services auto-activate for cloud integration you likely don't need:

| Service | Function | Why block |
|---------|----------|-----------|
| `org.gnome.OnlineAccounts` | Google, Microsoft, cloud integration | Cloud account sync — privacy risk |
| `org.gnome.Identity` | Kerberos / enterprise SSO login | Enterprise network authentication — not needed on personal systems |

Masking the systemd unit is not enough — D-Bus activation bypasses systemd. You need a D-Bus override.

### How

```bash
# Create override directory
mkdir -p ~/.local/share/dbus-1/services/

# Override GNOME Online Accounts
cat > ~/.local/share/dbus-1/services/org.gnome.OnlineAccounts.service << 'EOF'
[D-BUS Service]
Name=org.gnome.OnlineAccounts
Exec=/bin/false
EOF

# Override GNOME Identity (Kerberos)
cat > ~/.local/share/dbus-1/services/org.gnome.Identity.service << 'EOF'
[D-BUS Service]
Name=org.gnome.Identity
Exec=/bin/false
EOF
```

### How it works

D-Bus searches for service files in this order:
1. `~/.local/share/dbus-1/services/` (user overrides — checked first)
2. `/usr/share/dbus-1/services/` (system-installed services)

Your override with `Exec=/bin/false` intercepts the activation — D-Bus calls `/bin/false` instead of the real service, which immediately exits with failure. The requesting app gets a "service failed to start" error and moves on.

### Decision

```
D-Bus overrides
├── GNOME Online Accounts:
│   ├── Override if:  No cloud accounts (Google, Microsoft, Nextcloud) in GNOME
│   ├── Keep if:      You use GNOME Online Accounts for calendar/contacts/files sync
│   └── What breaks:  GNOME Settings → Online Accounts shows nothing. No cloud integration.
│
└── GNOME Identity (Kerberos):
    ├── Override if:  Personal system, no enterprise network (recommended)
    ├── Keep if:      You authenticate via Kerberos/Active Directory at work
    └── What breaks:  Enterprise SSO login. Has no effect on personal systems.
```

### Verify

```bash
ls ~/.local/share/dbus-1/services/
# Expected: org.gnome.Identity.service  org.gnome.OnlineAccounts.service

grep Exec ~/.local/share/dbus-1/services/*.service
# Expected: Exec=/bin/false (for both)
```

### Update persistence

These overrides survive system updates — `~/.local/share/` is user-owned and not touched by `dnf`.

### Undo

```bash
# Remove overrides to restore original behavior:
rm ~/.local/share/dbus-1/services/org.gnome.OnlineAccounts.service
rm ~/.local/share/dbus-1/services/org.gnome.Identity.service
```

---

## 6. Desktop Privacy Settings (gsettings)

### Why

GNOME tracks app usage, maintains recent file lists, generates thumbnails, auto-mounts media, and sends crash reports by default. Each of these is either a privacy leak or an attack vector.

### How

```bash
# --- Media handling ---
gsettings set org.gnome.desktop.media-handling automount false
gsettings set org.gnome.desktop.media-handling automount-open false
gsettings set org.gnome.desktop.media-handling autorun-never true

# --- Privacy ---
gsettings set org.gnome.desktop.privacy remember-app-usage false
gsettings set org.gnome.desktop.privacy remember-recent-files false
gsettings set org.gnome.desktop.privacy recent-files-max-age 1
gsettings set org.gnome.desktop.privacy remove-old-temp-files true
gsettings set org.gnome.desktop.privacy remove-old-trash-files true
gsettings set org.gnome.desktop.privacy old-files-age 7
gsettings set org.gnome.desktop.privacy report-technical-problems false
gsettings set org.gnome.desktop.privacy send-software-usage-stats false

# --- Thumbnails ---
gsettings set org.gnome.desktop.thumbnailers disable-all true

# --- Search ---
gsettings set org.gnome.desktop.search-providers disable-external true

# --- Lockscreen ---
gsettings set org.gnome.desktop.screensaver lock-enabled true
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.desktop.notifications show-in-lock-screen false

# --- Camera ---
gsettings set org.gnome.desktop.privacy disable-camera true
```

### Settings explained

| Setting | Value | Why |
|---------|-------|-----|
| `automount` | `false` | USB devices not auto-mounted — defense-in-depth with USBGuard (→ Doc 14) |
| `automount-open` | `false` | Don't open file manager on mount |
| `autorun-never` | `true` | Never execute programs from mounted media — prevents autorun malware |
| `remember-app-usage` | `false` | GNOME doesn't track which apps you use and when |
| `remember-recent-files` | `false` | No "recent files" list — prevents information disclosure |
| `recent-files-max-age` | `1` | If recent files are somehow tracked: delete after 1 day |
| `remove-old-temp-files` | `true` | Auto-clean temporary files |
| `remove-old-trash-files` | `true` | Auto-clean trash |
| `old-files-age` | `7` | Clean temp/trash after 7 days |
| `report-technical-problems` | `false` | No crash reports sent to Fedora/GNOME |
| `send-software-usage-stats` | `false` | No usage statistics |
| `disable-all` (thumbnailers) | `true` | **No thumbnail generation** — prevents code execution via malicious files |
| `disable-external` (search) | `true` | No online search providers in GNOME Activities search |
| `lock-enabled` | `true` | Screen locks on idle |
| `idle-delay` | `300` | Lock after 5 minutes (adjust to your preference) |
| `show-in-lock-screen` | `false` | **No notifications on lock screen** — prevents information leakage |
| `disable-camera` | `true` | Camera access blocked for all apps |

### Decision trees

```
Thumbnails (disable-all)
├── Disable (recommended):
│   ├── Eliminates code execution via thumbnail parsers
│   ├── Historical CVEs in libjpeg, libpng, ffmpeg thumbnail generators
│   └── Files display as generic icons in file manager
│
└── Keep enabled if:
    ├── You heavily rely on visual file browsing (image/video workflows)
    └── Accept the risk of thumbnail parser exploits

Camera (disable-camera)
├── Disable (recommended if no webcam use):
│   ├── No app can access the camera
│   └── Includes video calls — check your needs
│
└── Keep enabled if:
    ├── You use video calls (Signal, browser-based conferencing)
    └── Camera access is then controlled per-app via portals

Microphone:
├── Not disabled in this config — needed for voice/video calls
├── Disable if: No voice calls, no audio recording
│   gsettings set org.gnome.desktop.privacy disable-microphone true
└── What breaks: All microphone access (calls, voice memos, dictation)

Idle delay (idle-delay)
├── 300 (5 min) — good balance for desktop use
├── 60 (1 min) — high-security environments, shared spaces
├── 0 — never auto-lock (not recommended)
└── Adjust: gsettings set org.gnome.desktop.session idle-delay <SECONDS>

Automount
├── Disable (recommended):
│   ├── Defense-in-depth with USBGuard (→ Doc 14)
│   ├── USB devices must be manually mounted via file manager
│   └── Three layers: USBGuard blocks → automount disabled → autorun disabled
│
└── Keep enabled if:
    ├── You frequently use USB drives and want convenience
    └── Still keep autorun-never=true regardless
```

### Verify

```bash
# Media handling:
gsettings get org.gnome.desktop.media-handling autorun-never
# Expected: true

# Thumbnails:
gsettings get org.gnome.desktop.thumbnailers disable-all
# Expected: true

# Lock screen notifications:
gsettings get org.gnome.desktop.notifications show-in-lock-screen
# Expected: false

# Camera:
gsettings get org.gnome.desktop.privacy disable-camera
# Expected: true

# Crash reports:
gsettings get org.gnome.desktop.privacy report-technical-problems
# Expected: false
```

### What breaks

| Setting | Impact |
|---------|--------|
| `automount=false` | USB drives don't auto-mount — use file manager to mount manually |
| `disable-all=true` (thumbnails) | Files show generic icons instead of previews in file manager |
| `remember-recent-files=false` | No "Recent" section in file dialogs |
| `disable-camera=true` | No camera access for any app — re-enable if you need video calls |
| `disable-external=true` (search) | GNOME Activities search only searches local apps, not web |

### Undo

```bash
# Restore any setting to default:
gsettings reset org.gnome.desktop.media-handling automount
gsettings reset org.gnome.desktop.privacy disable-camera
# (use 'reset' for any key to return to GNOME default)
```

---

## 7. Electron App Security Note

Many modern desktop apps (Signal, VS Code, Slack, Discord) are Electron-based (Chromium). Security considerations:

| Aspect | Detail |
|--------|--------|
| Display protocol | Modern Electron (28+) supports Wayland natively via `--ozone-platform=wayland` |
| Sandbox | RPM-installed Electron apps run with **full user permissions** — no sandbox |
| Localhost listeners | Electron apps commonly open `127.0.0.1:<random-port>` for internal IPC — this is normal |
| Flatpak alternative | Flatpak-packaged Electron apps get filesystem/network sandbox (→ Doc 18) |

### Decision

```
Electron app installation
├── Flatpak (preferred for untrusted apps):
│   ├── Filesystem sandbox — app can't read ~/Documents etc.
│   ├── Portal-based permissions for camera, microphone
│   └── Trade-off: larger install size, potential permission issues
│
├── RPM (acceptable for trusted vendors):
│   ├── Full user permissions — can read all user files
│   ├── Smaller install, native integration
│   └── Only for apps from trusted repos (vendor GPG-signed)
│
└── AppImage / direct download (avoid):
    ├── No sandbox, no signature verification
    ├── No automatic updates
    └── Use only if no RPM or Flatpak alternative exists
```

---

## 8. Complete Verification

```bash
# 1. Wayland session
echo $XDG_SESSION_TYPE
# Expected: wayland

# 2. Xwayland autoclose
gsettings get org.gnome.mutter experimental-features
# Expected: contains 'autoclose-xwayland'

# 3. Masked user services
systemctl --user list-unit-files --state=masked --no-legend | wc -l
# Expected: matches your masked count

# 4. D-Bus overrides
ls ~/.local/share/dbus-1/services/
# Expected: org.gnome.Identity.service  org.gnome.OnlineAccounts.service

# 5. Autorun disabled
gsettings get org.gnome.desktop.media-handling autorun-never
# Expected: true

# 6. Lockscreen notifications off
gsettings get org.gnome.desktop.notifications show-in-lock-screen
# Expected: false

# 7. Screen lock active
gsettings get org.gnome.desktop.screensaver lock-enabled
# Expected: true

# 8. Thumbnails disabled
gsettings get org.gnome.desktop.thumbnailers disable-all
# Expected: true

# 9. Camera disabled
gsettings get org.gnome.desktop.privacy disable-camera
# Expected: true

# 10. No crash reports
gsettings get org.gnome.desktop.privacy report-technical-problems
# Expected: false
```

---

## Important Notes

- **Wayland is non-negotiable**: X11 has fundamental design flaws (zero isolation) that cannot be patched. If your GPU driver forces X11, fix the driver situation first (→ Doc 19).
- **Evolution can't be uninstalled**: It's a GNOME core dependency. Masking the services is the correct approach — it prevents them from running without breaking the desktop.
- **D-Bus overrides survive updates**: Files in `~/.local/share/` are user-owned and not modified by `dnf upgrade`.
- **LocalSearch masking is permanent until unmasked**: If you later want file content search in GNOME Files, unmask the three localsearch services.
- **gsettings are per-user**: These settings apply only to the current user. Other users on the system are unaffected.
- **Thumbnail CVE history**: Thumbnail parsers have a long history of vulnerabilities — libjpeg, libpng, libwebp, ffmpeg thumbnailers have all had code execution CVEs. Disabling thumbnails eliminates this entire class of attack.

---

*Previous: [16 — Firefox Hardening](16-firefox-browser.md)*
*Next: [18 — Flatpak Sandboxing](18-flatpak-sandboxing.md) — Audit and restrict Flatpak application permissions*
