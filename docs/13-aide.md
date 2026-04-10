# 🔍 AIDE

> Daily file integrity monitoring: cryptographic hashes, permissions, SELinux labels.
> Applies to: Fedora Workstation 43+ | All hardware
> Requires database rebuild after system changes.

---

## Overview

AIDE (Advanced Intrusion Detection Environment) detects unauthorized changes to system files by comparing current state against a known-good database. It checks cryptographic hashes, file permissions, ownership, ACLs, and SELinux labels.

| What it detects | How |
|----------------|-----|
| Modified binaries (trojanized programs) | SHA-256/SHA-512 hash comparison |
| Changed permissions (privilege weakening) | Permission + ACL tracking |
| SELinux label tampering | SELinux context comparison |
| New/removed files | Database diff |

### AIDE + auditd: Complementary layers

```
System file modified
  │
  ├─ auditd (real-time): Logs WHO changed WHAT and WHEN → /var/log/audit/audit.log
  │
  └─ AIDE (daily): Detects WHAT changed (hash/permissions) → /var/log/aide/aide.log
```

| Tool | Detection | Timing | Strength |
|------|-----------|--------|----------|
| auditd | Who changed what, when | Real-time | Forensics, attribution |
| AIDE | What changed (hashes, permissions) | Periodic (daily) | Deep integrity verification |

> auditd is configured in [Doc 12](12-selinux-auditd.md). AIDE complements it — auditd catches changes as they happen, AIDE catches anything that was missed.

---

## 1. Configuration

### File: `/etc/aide.conf` (key entries)

```bash
# Database paths
database_in=file:/var/lib/aide/aide.db.gz
database_out=file:/var/lib/aide/aide.db.new.gz
gzip_dbout=yes

# Logging
log_level=warning
report_level=changed_attributes
report_url=file:/var/log/aide/aide.log
report_url=stdout

# Hash groups
FIPSR = p+i+n+u+g+s+m+c+acl+selinux+xattrs+sha256
NORMAL = FIPSR+sha512
DIR = p+i+n+u+g+acl+selinux+xattrs
PERMS = p+i+u+g+acl+selinux
DATAONLY = p+n+u+g+s+acl+selinux+xattrs+sha256

# Monitored directories
/boot   NORMAL
/bin    NORMAL
/sbin   NORMAL
/lib    NORMAL
/lib64  NORMAL
/opt    NORMAL
/usr    NORMAL
/root   NORMAL
!/usr/src
!/usr/tmp
/etc    PERMS
/etc/exports  NORMAL
/etc/fstab    NORMAL
/etc/passwd   NORMAL
/etc/group    NORMAL
/etc/gshadow  NORMAL
/etc/shadow   NORMAL
/etc/security/opasswd  NORMAL
/etc/hosts.allow  NORMAL
/etc/hosts.deny   NORMAL
```

### Hash Groups Explained

| Group | Attributes checked | Used for |
|-------|-------------------|----------|
| `NORMAL` | Permissions, inode, links, owner, group, size, mtime, ctime, ACL, SELinux, xattrs, SHA-256, SHA-512 | Binaries, libraries, boot files |
| `PERMS` | Permissions, inode, owner, group, ACL, SELinux | `/etc` (frequent changes — only check permissions, not content) |
| `DATAONLY` | Permissions, links, owner, group, size, ACL, SELinux, xattrs, SHA-256 | Data files |

### Why `/etc` uses PERMS instead of NORMAL

Files in `/etc` change frequently (config edits, package updates). Using `NORMAL` (with content hashing) would generate excessive false positives after every edit. `PERMS` catches unauthorized permission/ownership changes without flagging expected content modifications. Critical config files (`passwd`, `shadow`, etc.) are individually set to `NORMAL` for full hash checking.

---

## 2. Database Management

### How it works

| File | Purpose |
|------|---------|
| `aide.db.gz` | Active reference database (comparison baseline) |
| `aide.db.new.gz` | Latest scan output (becomes new reference after update) |

### Rebuilding after system changes

After any system change (DNF updates, config edits, package installation), the database must be rebuilt:

```bash
$ sudo aide --update
$ sudo cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

> **Important gotchas:**
> - `/var/lib/aide/` is root-only — use `sudo sh -c '...'` for glob operations
> - AIDE exit codes are **bitmasks** (1=added, 2=removed, 4=changed) — use `;` not `&&` before the `cp` command, otherwise the copy won't execute when changes are detected

### Decision

```
When to rebuild AIDE database
├── After: sudo dnf upgrade (package hashes change)
├── After: Any config file change in /etc
├── After: Kernel module changes
├── After: GRUB/bootloader changes
├── DO NOT rebuild: If you didn't make the changes (investigate first!)
└── Workflow: aide --update ; cp aide.db.new.gz aide.db.gz
```

---

## 3. Scheduling: systemd Timer

### Service: `/etc/systemd/system/aide-check.service`

```ini
[Unit]
Description=AIDE File Integrity Check
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/aide --check
StandardOutput=journal
StandardError=journal
```

### Timer: `/etc/systemd/system/aide-check.timer`

```ini
[Unit]
Description=Daily AIDE integrity check
Wants=aide-check.service

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

| Parameter | Value | Why |
|-----------|-------|-----|
| `OnCalendar` | `daily` | Run once per day |
| `RandomizedDelaySec` | `1h` | Random delay up to 1 hour (avoids predictable timing) |
| `Persistent` | `true` | Missed runs are caught up after boot |

### Exit Code Drop-In: `/etc/systemd/system/aide-check.service.d/exitcode.conf`

```ini
[Service]
SuccessExitStatus=1 2 3 4 5 6 7
```

**Why needed:** AIDE returns exit codes 1-7 when changes are detected (bitmask: added/removed/changed). Without this drop-in, systemd marks the service as "failed" after every run that finds changes — which is normal behavior after system updates. The drop-in tells systemd: exit 1-7 = success.

---

## 4. Known False Positives

| Warning | Cause | Status |
|---------|-------|--------|
| SELinux AVC: AIDE → `xdm_var_run_t` | AIDE reads GDM runtime files | Benign — expected |
| SELinux AVC: AIDE → `dosfs_t` | AIDE reads EFI partition | Benign — expected |
| Exit code ≠ 0 after updates | Bitmask (changes detected = expected) | Normal — rebuild database |

---

## 5. Desktop Notifications (Optional)

### What

A notification script that sends a GNOME desktop alert when AIDE detects file integrity changes. Without this, AIDE logs silently — you only find out about changes when you manually check the journal.

### Why

AIDE runs daily via systemd timer. If an attacker modifies system files, you won't notice until you actively run `journalctl -u aide-check.service`. The notification script bridges this gap — changes trigger an immediate desktop alert with details about what was detected.

### Script: `/usr/local/bin/aide-notify.sh`

```bash
#!/bin/bash
# AIDE Result Notification — sends GNOME desktop notification if AIDE found changes
# Triggered by aide-check.service via ExecStartPost

AIDE_EXIT=${1:-0}
AIDE_LOG="/var/log/aide/aide.log"
DISPLAY_USER="<YOUR-USERNAME>"
DISPLAY_UID=$(id -u "$DISPLAY_USER")

send_notification() {
    local urgency="$1"
    local title="$2"
    local body="$3"

    sudo -u "$DISPLAY_USER" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${DISPLAY_UID}/bus" \
        DISPLAY=:0 \
        notify-send --urgency="$urgency" --icon="dialog-warning" "$title" "$body"
}

case "$AIDE_EXIT" in
    0)
        # No changes — no notification needed (silent success)
        ;;
    1|2|3|4|5|6|7)
        # Changes detected (bitmask: 1=added, 2=removed, 4=changed)
        DETAILS=""
        [ $((AIDE_EXIT & 1)) -ne 0 ] && DETAILS="${DETAILS}New files found. "
        [ $((AIDE_EXIT & 2)) -ne 0 ] && DETAILS="${DETAILS}Files removed. "
        [ $((AIDE_EXIT & 4)) -ne 0 ] && DETAILS="${DETAILS}Files modified. "

        send_notification "critical" \
            "AIDE: File Integrity Changes Detected" \
            "${DETAILS}Check: sudo journalctl -u aide-check.service"
        ;;
    *)
        # Exit >= 14 = real error
        send_notification "critical" \
            "AIDE: Integrity Check ERROR" \
            "AIDE exited with code ${AIDE_EXIT}. Check: sudo journalctl -u aide-check.service"
        ;;
esac
```

> Replace `<YOUR-USERNAME>` with your actual username (`whoami`).

### How it works

The script is triggered by a systemd drop-in after every AIDE check. It decodes AIDE's bitmask exit code:

| Exit code | Meaning | Notification |
|-----------|---------|-------------|
| `0` | No changes | Silent — no notification |
| `1` | New files added | Critical alert |
| `2` | Files removed | Critical alert |
| `4` | Files modified | Critical alert |
| `1-7` | Any combination | Critical alert with decoded details |
| `≥ 14` | AIDE error | Error alert |

### Drop-In: `/etc/systemd/system/aide-check.service.d/notify.conf`

```ini
[Service]
ExecStartPost=/bin/bash -c '/usr/local/bin/aide-notify.sh $EXIT_STATUS'
```

> **Why `$EXIT_STATUS`?** Available in ExecStartPost since systemd v254 (Fedora 43 ships v257+). Contains the exit code of the preceding ExecStart command (AIDE's bitmask exit code). Do NOT use `%t` — that is a systemd unit specifier expanding to `/run` (the runtime directory), not the exit code.

### Decision

```
AIDE desktop notifications
├── Enable (recommended):
│   ├── Immediate visibility when files change
│   ├── No impact on AIDE operation
│   └── Silent on success — only alerts on changes
│
├── Skip if:
│   ├── Headless server (no GNOME session for notify-send)
│   └── You check aide-check.service logs regularly
│
└── What breaks: nothing — the notification is advisory only
```

---

## 🔧 Applying Changes

### Step 1: Install and initialize

```bash
$ sudo dnf install aide
$ sudo aide --init
$ sudo cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

### Step 2: Create log directory

```bash
$ sudo mkdir -p /var/log/aide
$ sudo chmod 700 /var/log/aide
```

### Step 3: Create systemd timer and service

```bash
$ sudo tee /etc/systemd/system/aide-check.service << 'SVC'
[Unit]
Description=AIDE File Integrity Check
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/aide --check
StandardOutput=journal
StandardError=journal
SVC

$ sudo tee /etc/systemd/system/aide-check.timer << 'TIMER'
[Unit]
Description=Daily AIDE integrity check
Wants=aide-check.service

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
TIMER

$ sudo mkdir -p /etc/systemd/system/aide-check.service.d
$ sudo tee /etc/systemd/system/aide-check.service.d/exitcode.conf << 'EXIT'
[Service]
SuccessExitStatus=1 2 3 4 5 6 7
EXIT

$ sudo systemctl daemon-reload
$ sudo systemctl enable --now aide-check.timer
```

### Step 4: Install notification script (optional)

```bash
# Create the script:
$ sudo tee /usr/local/bin/aide-notify.sh << 'NOTIFY'
#!/bin/bash
AIDE_EXIT=${1:-0}
DISPLAY_USER="<YOUR-USERNAME>"
DISPLAY_UID=$(id -u "$DISPLAY_USER")

send_notification() {
    local urgency="$1"
    local title="$2"
    local body="$3"
    sudo -u "$DISPLAY_USER" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${DISPLAY_UID}/bus" \
        DISPLAY=:0 \
        notify-send --urgency="$urgency" --icon="dialog-warning" "$title" "$body"
}

case "$AIDE_EXIT" in
    0) ;;
    1|2|3|4|5|6|7)
        DETAILS=""
        [ $((AIDE_EXIT & 1)) -ne 0 ] && DETAILS="${DETAILS}New files found. "
        [ $((AIDE_EXIT & 2)) -ne 0 ] && DETAILS="${DETAILS}Files removed. "
        [ $((AIDE_EXIT & 4)) -ne 0 ] && DETAILS="${DETAILS}Files modified. "
        send_notification "critical" "AIDE: File Integrity Changes Detected" \
            "${DETAILS}Check: sudo journalctl -u aide-check.service" ;;
    *)
        send_notification "critical" "AIDE: Integrity Check ERROR" \
            "AIDE exited with code ${AIDE_EXIT}. Check: sudo journalctl -u aide-check.service" ;;
esac
NOTIFY
$ sudo chmod 755 /usr/local/bin/aide-notify.sh

# Create the systemd drop-in:
$ sudo tee /etc/systemd/system/aide-check.service.d/notify.conf << 'DROP'
[Service]
ExecStartPost=/bin/bash -c '/usr/local/bin/aide-notify.sh $EXIT_STATUS'
DROP
$ sudo systemctl daemon-reload
```

> **Edit** the script to replace `<YOUR-USERNAME>` with your actual username.

---

## ✅ Complete Verification

```bash
# 1. AIDE database exists:
$ sudo ls -la /var/lib/aide/aide.db.gz
# Expected: File exists

# 2. Timer active:
$ systemctl is-active aide-check.timer
# Expected: active

# 3. Manual check:
$ sudo aide --check
# Expected: No unexpected changes (exit code is bitmask — see notes)

# 4. Log directory exists:
$ ls -ld /var/log/aide
# Expected: drwx------ root root

# 5. Exit code drop-in active:
$ systemctl cat aide-check.service | grep SuccessExitStatus
# Expected: SuccessExitStatus=1 2 3 4 5 6 7

# 6. Notification script exists (if installed):
$ ls -la /usr/local/bin/aide-notify.sh
# Expected: -rwxr-xr-x root root

# 7. Notification drop-in active (if installed):
$ systemctl cat aide-check.service | grep ExecStartPost
# Expected: ExecStartPost=/bin/bash -c '/usr/local/bin/aide-notify.sh $EXIT_STATUS'

# 8. Test notification (should show desktop alert):
$ sudo /usr/local/bin/aide-notify.sh 4
# Expected: Desktop notification "AIDE: File Integrity Changes Detected"
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| Daily AIDE check | CPU/IO during scan (~minutes) | Detects unauthorized file modifications |
| Database rebuild required | Manual step after every system change | Ensures baseline is always current |
| `/etc` uses PERMS only | Content changes to /etc not hash-checked | Avoids false positives from frequent config edits |
| Exit code drop-in | Hides "changes found" from systemd status | Timer doesn't show "failed" after every update |
| Desktop notifications | Requires active GNOME session | Immediate visibility of file integrity changes |

---

## Notes

- **Rebuild after EVERY `dnf upgrade`.** Updated packages change binary hashes. Without a database rebuild, AIDE will report hundreds of false changes on the next check.
- **AIDE exit codes are bitmasks:** Exit 1 = new files, 2 = removed files, 4 = changed files. Combinations possible (e.g., 7 = all three). Exit ≥ 14 = actual errors. Never use `&&` after `aide --update` — use `;` instead.
- **`/var/lib/aide/` is root-only.** For glob operations: `sudo sh -c 'ls /var/lib/aide/*.gz'`.
- **Do NOT log to `/tmp/`.** It's world-writable — an attacker could manipulate AIDE logs. Use `/var/log/aide/` (root:root, mode 700).
- **AIDE v0.19 config changes:** `database` → `database_in`, `summarize_changes` → `report_summarize_changes`. Fedora 43 ships v0.19+ with correct defaults.
- **Suspicious changes:** If AIDE reports changes you didn't make, **do NOT rebuild the database.** Investigate first — this could indicate a compromise.

---

*Previous: [12 — SELinux & auditd](12-selinux-auditd.md)*
*Next: [14 — USBGuard](14-usbguard.md) — Block-by-default USB policy with hash-based identification*
