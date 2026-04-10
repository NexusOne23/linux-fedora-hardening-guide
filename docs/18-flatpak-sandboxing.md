# 📦 Flatpak Sandboxing

> Audit and restrict Flatpak application permissions using Flatseal and CLI overrides.
> Applies to: Fedora Workstation 43+ | Any hardware
> No reboot required — permission changes apply on next app launch.

---

## Overview

Flatpak applications run in sandboxed containers with restricted access to the host system. However, many Flatpaks request overly broad permissions by default — `filesystem=host`, `devices=all`, or unrestricted D-Bus access can effectively disable the sandbox.

This document covers how to audit Flatpak permissions, restrict them to the minimum needed, and decide when Flatpak vs RPM is the better choice.

### Why

A Flatpak app with `filesystem=host` has **full read/write access to your entire home directory** — SSH keys, browser profiles, password databases, everything. The "sandboxed" label is misleading if permissions aren't reviewed. Auditing and restricting permissions is the difference between real isolation and security theater.

### What you get

| Action | Effect |
|--------|--------|
| Permission audit | Know exactly what each app can access |
| Filesystem restrictions | Apps only see directories they actually need |
| Device restrictions | Camera/microphone only for apps that need them |
| Network removal | Offline apps can't phone home |
| D-Bus restrictions | Apps can't activate arbitrary system services |

---

## 1. Flatpak vs RPM: When to Use What

### Decision

```
Application packaging choice
├── Use Flatpak for:
│   ├── Third-party apps not in Fedora repos (Signal, Slack, Discord)
│   ├── Apps you don't fully trust — sandbox provides real isolation
│   ├── Apps that handle untrusted content (media players, image viewers)
│   └── Apps where filesystem isolation is valuable (messengers, games)
│
├── Use RPM for:
│   ├── System tools that need full host access (IDE, terminal, file manager)
│   ├── Apps from trusted repos (Fedora, vendor RPM repos with GPG signing)
│   ├── Apps where Flatpak sandbox causes breakage (development tools)
│   └── CLI tools and system utilities
│
└── Key insight:
    ├── Flatpak with filesystem=host ≈ RPM (sandbox is effectively disabled)
    ├── If an app needs host access to function, RPM is more honest
    └── Flatpak's value is ONLY in apps where you can actually restrict permissions
```

### The filesystem=host trap

An app with `filesystem=host` permission can read and write your entire home directory. This includes:

| What's exposed | Risk |
|---------------|------|
| `~/.ssh/` | SSH private keys |
| `~/.gnupg/` | GPG keys |
| `~/.config/` | All application configs |
| `~/.mozilla/` | Browser profiles, cookies, passwords |
| `~/Documents/`, `~/Downloads/` | All personal files |

**If an app needs `filesystem=host`, install it as RPM instead** — you get the same access level without the false sense of security.

---

## 2. Installing Flatseal

Flatseal is a GUI tool for managing Flatpak permissions. It shows every permission each app has and lets you toggle them individually.

### How

```bash
flatpak install flathub com.github.tchx84.Flatseal
```

### Flatseal's own permissions

Flatseal itself has a tight sandbox:

| Category | Permission | Why |
|----------|-----------|-----|
| Shared | `ipc` | Inter-process communication (no network) |
| Sockets | `wayland`, `x11`, `fallback-x11` | Display (GUI app) |
| Devices | `dri` | GPU rendering only |
| Network | **None** | Manages local permissions only — no internet needed |

> **Tip**: Flatseal having no network access is a good sign — a permission manager that phones home would be a red flag.

---

## 3. Auditing Flatpak Permissions

### CLI method

```bash
# List all installed Flatpak apps:
flatpak list --app --columns=application,name

# Show permissions for a specific app:
flatpak info --show-permissions <APP-ID>

# Example:
flatpak info --show-permissions org.signal.Signal
```

### Key permission categories

| Category | Key | What it controls |
|----------|-----|-----------------|
| **Shared** | `network` | Internet access |
| **Shared** | `ipc` | Inter-process communication with other apps |
| **Sockets** | `x11` | X11 display access (weaker isolation than Wayland) |
| **Sockets** | `wayland` | Wayland display access (strong isolation) |
| **Sockets** | `pulseaudio` | Audio playback and recording |
| **Devices** | `all` | Access to ALL devices (camera, microphone, USB, GPU) |
| **Devices** | `dri` | GPU only — no camera, no microphone |
| **Filesystems** | `host` | Full read/write to home directory + `/run/media` (removable media) |
| **Filesystems** | `home` | Read/write to `~/` only |
| **Filesystems** | `xdg-download` | Only `~/Downloads/` |
| **Filesystems** | `xdg-documents` | Only `~/Documents/` |
| **D-Bus (talk)** | `org.freedesktop.secrets` | Access to GNOME Keyring / KDE Wallet |
| **D-Bus (talk)** | `org.freedesktop.Notifications` | Send desktop notifications |

### Red flags to look for

```
Permission audit checklist
├── filesystem=host or filesystem=home:
│   ├── RED FLAG — app can read all your files
│   ├── Ask: does this app actually need broad file access?
│   └── Restrict to specific directories if possible (xdg-download, etc.)
│
├── devices=all:
│   ├── YELLOW FLAG — app can access camera, microphone, all USB
│   ├── Acceptable for: video call apps, media apps
│   ├── Restrict to: devices=dri (GPU only) if no camera/mic needed
│   └── Note: Flatpak portals handle camera/mic on a per-request basis
│       on Wayland — devices=all is the legacy approach
│
├── socket=x11 without wayland:
│   ├── YELLOW FLAG — app uses X11 only (weaker isolation)
│   ├── X11 apps under Xwayland can keylog other X11 apps
│   └── Prefer apps with Wayland support (→ Doc 17)
│
├── Talk permissions (D-Bus):
│   ├── org.freedesktop.secrets — keyring access (passwords)
│   ├── Acceptable for apps that store credentials
│   └── Suspicious for apps that have no credential storage
│
└── No network needed?
    ├── Remove network permission for offline tools
    ├── Example: Flatseal, image editors, PDF viewers
    └── Prevents data exfiltration even if app is compromised
```

---

## 4. Restricting Permissions

### Via Flatseal (GUI)

1. Open Flatseal
2. Select the app from the left panel
3. Toggle permissions off/on
4. Changes apply on next app launch

### Via CLI

```bash
# Remove a permission:
flatpak override --user <APP-ID> --nofilesystem=host
flatpak override --user <APP-ID> --nodevice=all
flatpak override --user <APP-ID> --nosocket=x11
flatpak override --user <APP-ID> --unshare=network

# Add a restricted permission instead:
flatpak override --user <APP-ID> --filesystem=xdg-download
flatpak override --user <APP-ID> --device=dri

# View current overrides:
flatpak override --user --show <APP-ID>

# Reset all overrides (return to app defaults):
flatpak override --user --reset <APP-ID>
```

### Common restriction patterns

```bash
# App that only needs ~/Downloads for file transfer:
flatpak override --user <APP-ID> \
  --nofilesystem=host \
  --filesystem=xdg-download

# App that needs no camera/microphone (GPU only):
flatpak override --user <APP-ID> \
  --nodevice=all \
  --device=dri

# Offline tool (no internet):
flatpak override --user <APP-ID> \
  --unshare=network

# Combine multiple restrictions:
flatpak override --user <APP-ID> \
  --nofilesystem=host \
  --filesystem=xdg-download \
  --nodevice=all \
  --device=dri \
  --unshare=network
```

### Decision per app type

```
Messenger (Signal, Element, Telegram):
├── Network:     Keep (required)
├── Filesystem:  Remove host/home — messengers don't need file access
│                (file sharing uses portal file picker automatically)
├── Devices:     Keep all if you use video/audio calls
│                Reduce to dri if text-only
├── Sockets:     Keep wayland + pulseaudio
└── D-Bus:       org.freedesktop.secrets is normal (credential storage)

Media player (VLC, Celluloid):
├── Network:     Remove if playing local files only
├── Filesystem:  Restrict to xdg-videos or specific directories
├── Devices:     dri only (GPU for video decoding)
├── Sockets:     Keep wayland + pulseaudio
└── D-Bus:       Minimal

Image editor (GIMP, Inkscape):
├── Network:     Remove (offline tool)
├── Filesystem:  Restrict to xdg-pictures or project directory
├── Devices:     dri only
├── Sockets:     Keep wayland
└── D-Bus:       Minimal

Permission manager (Flatseal):
├── Network:     None (already restricted)
├── Filesystem:  Flatpak directories only (automatic)
├── Devices:     dri only
├── Sockets:     Keep wayland
└── D-Bus:       Minimal — verify no network
```

---

## 5. Flatpak Security Architecture

### How the sandbox works

```
┌─────────────────────────────────────────────┐
│  Host System                                │
│  ┌───────────────────────────────────────┐  │
│  │  Flatpak Runtime (shared libraries)   │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  App Container                  │  │  │
│  │  │  - Own /usr, /app filesystem    │  │  │
│  │  │  - Filtered /dev                │  │  │
│  │  │  - Namespaced PID, network      │  │  │
│  │  │  - seccomp syscall filter       │  │  │
│  │  │  - Portals for host interaction │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

| Layer | What it does |
|-------|-------------|
| **Namespaces** | Isolated PID, mount, network, user namespaces |
| **seccomp** | Blocks dangerous syscalls (no `ptrace`, no kernel module loading) |
| **Filesystem** | App sees its own `/usr` and `/app`, host filesystem only via explicit permissions |
| **Portals** | XDG Desktop Portals mediate access to files, camera, screen — user must approve |
| **D-Bus filter** | App can only talk to explicitly allowed D-Bus services |

### Portals vs direct permissions

Portals are the modern way to handle host interaction. Instead of giving an app `filesystem=host`, the app requests a file via the portal — GNOME shows a file picker, the user selects a file, and only that file is shared.

| Method | How it works | Security |
|--------|-------------|----------|
| `filesystem=host` | App reads files directly | No isolation — app sees everything |
| Portal file picker | User selects file, only that file is shared | Strong isolation — user controls access |
| `devices=all` | App accesses camera directly | Always-on access |
| Portal camera | App requests camera, user approves | Per-use approval |

> **Note**: Portals require Wayland. On X11, some portals fall back to unrestricted access. This is another reason to use Wayland (→ Doc 17).

---

## 6. Flatpak Update Management

### Why it matters

Flatpak apps and runtimes have their own update cycle, separate from `dnf`. Forgetting to update Flatpaks leaves known vulnerabilities unpatched.

### How

```bash
# Update all Flatpak apps and runtimes:
flatpak update

# Check for updates without installing:
flatpak update --no-deploy

# Include in your update routine (→ Doc 25):
sudo dnf upgrade && flatpak update
```

### Unused runtimes

Over time, old runtimes accumulate. Clean them up:

```bash
# Remove unused runtimes:
flatpak uninstall --unused
```

---

## 7. Complete Verification

```bash
# 1. List installed Flatpak apps:
flatpak list --app --columns=application,name
# Review: only apps you actually use should be listed

# 2. Check permissions for each app:
for app in $(flatpak list --app --columns=application); do
  echo "=== $app ==="
  flatpak info --show-permissions "$app" 2>/dev/null
  echo
done

# 3. Check for filesystem=host (red flag):
for app in $(flatpak list --app --columns=application); do
  if flatpak info --show-permissions "$app" 2>/dev/null | grep -q "host"; then
    echo "WARNING: $app has host filesystem access"
  fi
done

# 4. Check for overrides you've applied:
ls ~/.local/share/flatpak/overrides/ 2>/dev/null
# Lists apps with custom permission overrides

# 5. Verify Flatseal has no network:
flatpak info --show-permissions com.github.tchx84.Flatseal | grep -c "network"
# Expected: 0 (no network permission)

# 6. Check for pending updates:
flatpak update --no-deploy
```

---

## Important Notes

- **Flatpak sandbox is only as strong as its permissions**: An app with `filesystem=host` has effectively no sandbox. Always audit permissions after installation.
- **Flatseal overrides are per-user**: Stored in `~/.local/share/flatpak/overrides/` — survive Flatpak updates.
- **App updates can add new permissions**: After `flatpak update`, re-check permissions in Flatseal. New versions may request additional access.
- **Portals require Wayland**: On X11 sessions, some portals fall back to unrestricted access. Use Wayland (→ Doc 17).
- **User namespaces required**: Flatpak needs unprivileged user namespaces to function. Doc 02 limits them to 256 (not disabled) — sufficient for Flatpak sandboxing while reducing kernel exploit surface.
- **Flathub vs Fedora Flatpaks**: Fedora ships some Flatpaks from its own repo. Flathub apps are community-maintained — same trust model as AUR on Arch. Prefer Fedora repo Flatpaks when available.
- **chrome-sandbox SUID bit**: Electron-based Flatpaks and RPMs include a `chrome-sandbox` binary with SUID root. This is expected — it enables Chromium's namespace-based renderer sandbox. Without SUID, the browser has weaker process isolation.

---

*Previous: [17 — Desktop Stack](17-desktop-stack.md)*
*Next: [19 — GPU & Secure Boot](19-gpu-secureboot.md) — Proprietary GPU drivers with MOK signing*
