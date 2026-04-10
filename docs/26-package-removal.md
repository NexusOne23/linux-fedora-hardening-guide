# 🗑️ Package Removal

> Remove unnecessary packages to reduce attack surface — safely, without breaking GNOME.
> Applies to: Fedora Workstation 43+ | All hardware
> No reboot required (unless kernel packages are removed).

---

## Overview

A default Fedora Workstation installation includes packages you likely don't need — web servers, print drivers, Samba clients, mDNS responders, Bluetooth userspace tools, and installer remnants. Each unnecessary package is attack surface: code that runs, listens on ports, or processes untrusted input.

### Why

| Category | Example packages | Risk if kept |
|----------|-----------------|-------------|
| Web server | `httpd` | Listening on ports, remote attack surface |
| Network discovery | `avahi`, `nss-mdns` | mDNS responder — broadcasts presence on network |
| File sharing | `samba-client`, `cifs-utils` | SMB protocol attack surface |
| Print system | `cups-browsed`, `hplip` | Network print discovery, CVE history |
| Installer | `anaconda-*` | Post-install remnant, no purpose on running system |
| Bluetooth userspace | `gnome-bluetooth` | UI for Bluetooth (modules already blacklisted → Doc 21) |

### Safety rules

Before removing any package:

1. **Check reverse dependencies**: `dnf repoquery --whatrequires <package>`
2. **Protect core GNOME**: Never remove packages required by `gnome-shell`, `gnome-control-center`, or `pipewire`
3. **Use `dnf remove`** (not `rpm -e`): dnf resolves dependencies and shows what else would be removed
4. **Review the removal list**: Always read what dnf proposes to remove before confirming

---

## 1. Packages You Cannot Remove (GNOME Dependencies)

These packages are pulled in by core GNOME components. Removing them breaks the desktop:

| Package | Required by | Mitigation |
|---------|------------|------------|
| `wsdd` | `gvfs` → `gnome-shell` chain | Mask the service + block in firewall (→ Doc 03, Doc 08) |
| `cups`, `cups-pk-helper` | `gnome-control-center` | Mask CUPS service, remove print plugins instead |
| `bluez`, `bluez-libs` | `pipewire-libs` (needs `libbluetooth.so.3`) | Blacklist kernel modules (→ Doc 21), mask BT service (→ Doc 08) |

> **Key insight**: You don't need to remove a package to neutralize it. Masking services (→ Doc 08), blacklisting kernel modules (→ Doc 21), and firewall rules (→ Doc 03) achieve the same security result without breaking GNOME dependencies.

---

## 2. Safe Removal Categories

### 2.1 Web Server and Related

```
httpd, mod_lua, httpd-tools
├── Remove (strongly recommended):
│   ├── A web server has no place on a desktop workstation
│   ├── Removes: httpd + automatic dependencies (mod_lua, httpd-tools)
│   └── What breaks: nothing — you're not running a web server
│
└── Keep if: you develop/test web applications locally (consider containers instead)
```

```bash
sudo dnf remove httpd
```

### 2.2 Installer Remnants

```
anaconda-core, anaconda-live, anaconda-tui, anaconda-webui
├── Remove (recommended):
│   ├── The Fedora installer is not needed on a running system
│   ├── Pulls in Cockpit, Python dependencies, and other packages
│   └── What breaks: nothing — you can't reinstall from a running system anyway
│
└── Keep if: no reason to keep these
```

```bash
sudo dnf remove anaconda-core anaconda-live anaconda-tui anaconda-webui
# Also removes: cockpit-*, various Python dependencies
```

### 2.3 GNOME Apps You Don't Use

```
GNOME apps: gnome-tour, gnome-contacts, gnome-maps, gnome-weather,
            gnome-clocks, gnome-calendar
├── Remove selectively:
│   ├── gnome-tour — first-run wizard, never needed again
│   ├── gnome-contacts — if you don't sync contacts via GNOME Online Accounts
│   ├── gnome-maps — if you use web-based maps
│   ├── gnome-weather — if you don't need desktop weather
│   ├── gnome-clocks — if you don't use world clocks/alarms
│   └── gnome-calendar — if you don't use GNOME's calendar
│
├── Keep if: you actually use these apps
│
└── Safe to remove: these are standalone apps, not GNOME core
```

```bash
sudo dnf remove gnome-tour gnome-contacts gnome-maps gnome-weather gnome-clocks gnome-calendar
```

### 2.4 Samba / CIFS (File Sharing)

```
samba-client, cifs-utils, cifs-utils-info, gvfs-smb
├── Remove if:
│   ├── No Windows file shares on your network
│   ├── No NAS that requires SMB access
│   └── You use SFTP/SSH for file transfer instead
│
├── Keep if:
│   ├── You access Windows/Samba shares
│   ├── Your NAS uses SMB protocol
│   └── Note: wsdd cannot be removed (GNOME dep) — mask it instead
│
└── What breaks: SMB file sharing in Nautilus (GNOME Files)
```

```bash
sudo dnf remove samba-client cifs-utils gvfs-smb
```

### 2.5 Print System Components

```
Print packages: cups-browsed, hplip, gutenprint-cups, bluez-cups
├── Remove if:
│   ├── No printer connected
│   ├── CUPS service already masked (→ Doc 08)
│   └── Note: cups and cups-pk-helper CANNOT be removed (GNOME dep)
│
├── Keep if:
│   ├── You use a printer (keep cups-browsed for network printer discovery)
│   ├── You use an HP printer (keep hplip)
│   └── You use a printer that needs Gutenprint drivers
│
└── What breaks: printer auto-discovery, HP printer support, Bluetooth printing
```

```bash
sudo dnf remove cups-browsed hplip gutenprint-cups bluez-cups
```

### 2.6 Avahi / mDNS (Network Discovery)

```
avahi, nss-mdns
├── Remove (recommended):
│   ├── mDNS broadcasts your hostname on the local network
│   ├── Enables .local hostname resolution (rarely needed)
│   ├── CVE history in Avahi daemon
│   └── Removes: avahi + automatic dependencies
│
├── Keep if:
│   ├── You use AirPlay, AirPrint, or Chromecast
│   ├── You need .local hostname resolution
│   └── Network printers that use mDNS discovery
│
└── What breaks: .local name resolution, mDNS-based device discovery
```

```bash
sudo dnf remove avahi nss-mdns
```

### 2.7 Bluetooth Userspace

```
gnome-bluetooth, NetworkManager-bluetooth
├── Remove if:
│   ├── No Bluetooth hardware
│   ├── Bluetooth kernel modules already blacklisted (→ Doc 21)
│   └── Bluetooth service already masked (→ Doc 08)
│
├── Keep if:
│   ├── You use Bluetooth devices (even occasionally)
│   └── Note: bluez + bluez-libs CANNOT be removed (pipewire dep)
│
└── What breaks: GNOME Bluetooth UI, NetworkManager BT tethering
```

```bash
sudo dnf remove gnome-bluetooth NetworkManager-bluetooth
```

### 2.8 Help System and Misc

```
yelp (GNOME Help), evolution-ews (Exchange Web Services)
├── Remove if:
│   ├── yelp — you read documentation online or via man pages
│   ├── evolution-ews — you don't use Microsoft Exchange
│   └── These are safe to remove without affecting GNOME core
│
├── Keep if:
│   ├── You read GNOME's built-in help documentation
│   └── You use Evolution with Microsoft Exchange
│
└── What breaks: F1 help in GNOME apps (yelp), Exchange sync in Evolution
```

```bash
sudo dnf remove yelp evolution-ews
```

### 2.9 Other Candidates

| Package | Purpose | Remove if |
|---------|---------|-----------|
| `gamemode` | Game performance optimizer | Not gaming on this system |
| `nfs-utils` | NFS client/server | No NFS shares to mount |
| `antiword` | Word document converter | Not needed for security |
| `rpcbind` | RPC port mapper (NFS) | No NFS/RPC services |

```bash
sudo dnf remove gamemode nfs-utils antiword rpcbind
```

---

## 3. libvirt Default Network

If libvirt is installed (for virtual machines), it creates a default NAT network (`virbr0`) with a dnsmasq instance listening on `192.168.122.1:53`. This runs even if no VMs are active.

### Disable

```bash
# Stop the default network:
sudo virsh net-destroy default

# Prevent auto-start:
sudo virsh net-autostart default --disable
```

### If libvirt sockets are masked (→ Doc 08)

With masked libvirt sockets, `virsh` cannot connect. To temporarily use VMs:

```bash
# 1. Unmask sockets + service:
sudo systemctl unmask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd.service

# 2. Start service:
sudo systemctl start libvirtd.socket libvirtd

# 3. Start network:
sudo virsh net-start default

# 4. Use your VM...

# 5. When done — re-mask:
sudo virsh net-destroy default
sudo systemctl stop libvirtd
sudo systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd.service
```

---

## 4. Cleaning Up Orphaned Dependencies

After removing packages, check for orphaned dependencies:

```bash
# List packages no longer required by anything:
dnf repoquery --unneeded

# Review the list, then remove if safe:
sudo dnf autoremove
```

> **Caution with autoremove**: Review what dnf proposes to remove. `autoremove` can sometimes flag packages you installed intentionally (if they were pulled in as dependencies initially). When in doubt, remove specific packages by name rather than using `autoremove`.

---

## 5. Complete Verification

```bash
# 1. Verify removed packages are gone:
rpm -q httpd avahi samba-client anaconda-core
# Expected: "package <name> is not installed" for each

# 2. No unexpected listeners:
sudo ss -tlnp
# Expected: only services you recognize (no httpd, avahi, cups-browsed)

# 3. wsdd masked (can't be removed):
systemctl is-enabled wsdd 2>/dev/null
# Expected: masked

# 4. libvirt network stopped (if applicable):
sudo virsh net-list --all 2>/dev/null
# Expected: default inactive, autostart no

# 5. No orphaned dependencies:
dnf repoquery --unneeded | wc -l
# Expected: 0 or low number
```

---

## Important Notes

- **Always check dependencies before removing**: `dnf repoquery --whatrequires <package>` shows what depends on a package. Never force-remove without checking.
- **dnf shows what will be removed**: Read the removal list before pressing 'y'. dnf resolves the full dependency tree — removing one package can trigger dozens of automatic removals.
- **GNOME dependencies are sacred**: `gnome-shell`, `gnome-control-center`, and `pipewire` must never be removed. Any package in their dependency chain must be neutralized via masking/blacklisting, not removal.
- **Masking > Removing when removal fails**: If a package can't be removed due to dependencies, mask its service (`systemctl mask`), blacklist its kernel module, and block its ports in the firewall. The package exists but does nothing.
- **Re-installing is easy**: `sudo dnf install <package>` restores any removed package. Package removal is fully reversible.
- **Run AIDE rebuild after removal**: Package removal changes files on disk. Rebuild the AIDE database (→ Doc 13, Doc 25) to avoid false integrity alerts.
- **NTP fallback**: If `pool.ntp.org` is configured as a fallback in `/etc/chrony.conf`, it uses unencrypted NTP. Remove or comment it out — use only NTS-authenticated servers (→ Doc 11).

---

*Previous: [25 — Update Process](25-update-process.md)*
*Back to: [00 — Overview](00-overview.md)*
