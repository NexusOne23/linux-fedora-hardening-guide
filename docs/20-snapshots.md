# 📸 Btrfs Snapshots (Snapper)

> Automatic filesystem snapshots for rollback, forensics, and recovery.
> Applies to: Fedora Workstation 43+ | Btrfs filesystem (Fedora default since F33)
> No reboot required — snapshots work immediately after setup.

---

## Overview

Snapper creates automatic Btrfs snapshots on a schedule (hourly) and around package operations (pre/post). If an update, configuration change, or hardening step breaks your system, you can roll back to a known-good state in seconds.

### Why

On a hardened system, you're modifying kernel parameters, firewall rules, SELinux policies, and dozens of config files. One wrong change can leave you with an unbootable or broken system. Snapshots provide:

| Capability | How it helps |
|------------|-------------|
| **Rollback** | Undo a bad update or config change instantly |
| **Pre/Post comparison** | See exactly what changed during a `dnf upgrade` |
| **Forensics** | Compare system state across time — detect unauthorized changes |
| **Recovery** | Restore accidentally deleted files from hourly snapshots |

### What you get

| Config | Subvolume | Timeline | Pre/Post | Cleanup |
|--------|-----------|----------|----------|---------|
| `root` | `/` | Hourly | Yes (manual — see Section 4) | Automatic |
| `home` | `/home` | Hourly | No | Automatic |

---

## 1. Installation

### Prerequisites

Btrfs must be your root filesystem. This is the Fedora default since Fedora 33.

```bash
# Verify Btrfs:
findmnt -n -o FSTYPE /
# Expected: btrfs
```

### Install

```bash
sudo dnf install snapper
```

### Create snapshot configurations

```bash
# Root filesystem:
sudo snapper -c root create-config /

# Home directory:
sudo snapper -c home create-config /home
```

### Enable timers

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

| Timer | Function | Interval |
|-------|----------|----------|
| `snapper-timeline.timer` | Creates hourly timeline snapshots | Every hour |
| `snapper-cleanup.timer` | Removes old snapshots per retention policy | Every hour |

---

## 2. Configuration

### File: `/etc/snapper/configs/root`

```bash
SUBVOLUME="/"
FSTYPE="btrfs"

# --- Storage limits ---
SPACE_LIMIT="0.5"
FREE_LIMIT="0.2"

# --- Access control ---
ALLOW_USERS=""
ALLOW_GROUPS=""
SYNC_ACL="no"

# --- Timeline snapshots ---
TIMELINE_CREATE="yes"
TIMELINE_CLEANUP="yes"
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="4"
TIMELINE_LIMIT_MONTHLY="3"
TIMELINE_LIMIT_QUARTERLY="0"
TIMELINE_LIMIT_YEARLY="1"

# --- Numbered snapshots (pre/post) ---
NUMBER_CLEANUP="yes"
NUMBER_MIN_AGE="3600"
NUMBER_LIMIT="50"
NUMBER_LIMIT_IMPORTANT="10"

# --- Empty pre/post cleanup ---
EMPTY_PRE_POST_CLEANUP="yes"
EMPTY_PRE_POST_MIN_AGE="3600"

# --- Background comparison ---
BACKGROUND_COMPARISON="yes"
```

> **Home config**: Create the same configuration for `/home` — same retention values apply. Use `sudo snapper -c home set-config KEY=VALUE` to adjust.

### Parameters explained

**Storage limits:**

| Parameter | Value | Why |
|-----------|-------|-----|
| `SPACE_LIMIT` | `0.5` | Snapshots may use max 50% of the Btrfs volume |
| `FREE_LIMIT` | `0.2` | At least 20% free space must remain |

**Timeline retention:**

| Parameter | Value | Effect |
|-----------|-------|--------|
| `TIMELINE_LIMIT_HOURLY` | `5` | Keep last 5 hourly snapshots |
| `TIMELINE_LIMIT_DAILY` | `7` | Keep last 7 daily snapshots |
| `TIMELINE_LIMIT_WEEKLY` | `4` | Keep last 4 weekly snapshots |
| `TIMELINE_LIMIT_MONTHLY` | `3` | Keep last 3 monthly snapshots |
| `TIMELINE_LIMIT_QUARTERLY` | `0` | No quarterly snapshots |
| `TIMELINE_LIMIT_YEARLY` | `1` | Keep 1 yearly snapshot |

**Pre/Post cleanup:**

| Parameter | Value | Effect |
|-----------|-------|--------|
| `NUMBER_LIMIT` | `50` | Max 50 numbered (pre/post) snapshots |
| `NUMBER_LIMIT_IMPORTANT` | `10` | Max 10 marked as "important" |
| `EMPTY_PRE_POST_CLEANUP` | `yes` | Delete pre/post pairs with no changes after 1 hour |

**Access control:**

| Parameter | Value | Effect |
|-----------|-------|--------|
| `ALLOW_USERS` | empty | Only root can manage snapshots |
| `ALLOW_GROUPS` | empty | No group access |

### Decision

```
Timeline retention tuning
├── Conservative (recommended — good balance):
│   ├── Hourly: 5, Daily: 7, Weekly: 4, Monthly: 3, Yearly: 1
│   ├── ~20 snapshots active at any time
│   └── Covers: last 5 hours + last week + last month
│
├── Aggressive (more history, more space):
│   ├── Hourly: 10, Daily: 14, Weekly: 8, Monthly: 6, Yearly: 2
│   ├── ~40 snapshots active at any time
│   └── Use if: you have plenty of disk space (500GB+)
│
├── Minimal (space-constrained):
│   ├── Hourly: 2, Daily: 3, Weekly: 2, Monthly: 1, Yearly: 0
│   ├── ~8 snapshots active at any time
│   └── Use if: disk space is tight (<128GB)
│
└── Space limit (safety net):
    ├── SPACE_LIMIT=0.5 prevents snapshots from consuming >50% of volume
    ├── FREE_LIMIT=0.2 ensures 20% free space remains
    └── Cleanup timer enforces these limits automatically
```

### How to apply settings

```bash
# Set individual parameters:
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY=5
sudo snapper -c root set-config TIMELINE_LIMIT_DAILY=7

# Or edit the config file directly:
sudo nano /etc/snapper/configs/root
```

---

## 3. Snapshot Types

### Timeline snapshots (automatic)

Created hourly by `snapper-timeline.timer`. Example output:

```bash
sudo snapper -c root list
# Type: single | Cleanup: timeline | Description: timeline
```

### Pre/Post snapshots (around updates)

Created in pairs: one **before** a change (pre) and one **after** (post). Allows you to see exactly what changed and roll it back.

```bash
sudo snapper -c root list --type pre-post
# Shows paired pre/post snapshots with their IDs
```

---

## 4. DNF Pre/Post Snapshots

### The dnf5 compatibility issue

> **Important**: Fedora 43+ uses `dnf5` as the default package manager. The `python3-dnf-plugin-snapper` package was designed for `dnf4` and **does not work with dnf5**. Automatic pre/post snapshots during `dnf install/upgrade/remove` do **not** happen automatically.

### Solution: Manual pre/post snapshots

Create snapshots manually around update operations:

```bash
# Before update:
sudo snapper -c root create --type pre --print-number --description "dnf upgrade" --cleanup-algorithm number
# Note the returned snapshot number (e.g., 42)

# Run your update:
sudo dnf upgrade

# After update:
sudo snapper -c root create --type post --pre-number 42 --description "dnf upgrade" --cleanup-algorithm number
```

### Automated wrapper (recommended)

Add this to your update script or create a shell function:

```bash
# In your update script or ~/.bashrc:
dnf-snap() {
  local pre_num post_num
  pre_num=$(sudo snapper -c root create --type pre --print-number \
    --description "$*" --cleanup-algorithm number)
  sudo dnf "$@"
  post_num=$(sudo snapper -c root create --type post --pre-number "$pre_num" \
    --print-number --description "$*" --cleanup-algorithm number)
  echo "Snapshots: pre=#${pre_num}, post=#${post_num}"
}

# Usage:
# dnf-snap upgrade
# dnf-snap install <package>
```

### Decision

```
DNF snapshot strategy
├── Manual pre/post (safest):
│   ├── Create snapshots before every dnf operation
│   ├── Full control over snapshot descriptions
│   └── See Section above for commands
│
├── Update script with built-in snapshots (recommended):
│   ├── Wrap dnf in a script that auto-creates pre/post pairs
│   ├── Never forget to snapshot before an update
│   └── Reference: → Doc 25 (Update Process)
│
└── Install python3-dnf-plugin-snapper anyway (partial):
    ├── May work for some dnf4-compatible operations
    ├── Does NOT work with dnf5 (Fedora 43+ default)
    └── Not recommended as sole strategy
```

---

## 5. Rollback Procedures

### Which rollback method to use

```
System state after a bad change
├── System boots normally, just misconfigured:
│   ├── Use: snapper undochange (fast, surgical)
│   ├── Reverts only the changes between pre/post snapshots
│   └── No reboot needed (unless kernel/boot files changed)
│
├── System boots but a service/app is broken:
│   ├── Use: restore single file from snapshot
│   ├── Copy just the broken config from a known-good snapshot
│   └── Minimal impact — only the specific file is restored
│
├── System won't boot at all:
│   ├── Use: Live USB emergency rollback
│   ├── Replaces the entire root subvolume with a snapshot
│   └── Nuclear option — use only when system is unbootable
│
└── Fedora 44+: GRUB snapshot boot (see note below)
    ├── Select a snapshot directly from the GRUB menu
    ├── No Live USB needed for unbootable systems
    └── Uses snapm + boom (automatic GRUB entries per snapshot)
```

### Undo a specific update (pre/post rollback)

```bash
# 1. List pre/post pairs:
sudo snapper -c root list --type pre-post

# 2. Show what changed between pre and post:
sudo snapper -c root diff <PRE-ID>..<POST-ID>

# 3. Undo all changes between pre and post:
sudo snapper -c root undochange <PRE-ID>..<POST-ID>

# 4. Reboot to apply (if kernel/boot files changed):
sudo reboot
```

### Restore a single file from a snapshot

```bash
# Find the snapshot with the file version you want:
sudo snapper -c root list

# Copy the file from the snapshot:
sudo cp /.snapshots/<SNAPSHOT-ID>/snapshot/etc/some-config /etc/some-config

# For home:
sudo cp /home/.snapshots/<SNAPSHOT-ID>/snapshot/<YOUR-USERNAME>/some-file ~/some-file
```

### Emergency rollback from Live USB

If the system won't boot:

```bash
# 1. Boot from Fedora Live USB

# 2. Open LUKS volume (find your partition with: lsblk -f):
sudo cryptsetup open /dev/<YOUR-LUKS-PARTITION> luks-restore

# 3. Mount Btrfs top-level (subvolid=5):
sudo mount -o subvolid=5 /dev/mapper/luks-restore /mnt

# 4. Verify Fedora subvolume layout:
sudo btrfs subvolume list /mnt
# Expected: subvolumes named "root" and "home"
# (Fedora uses "root"/"home", NOT "@"/"@home" like Ubuntu/Arch)

# 5. List available snapshots:
ls /mnt/root/.snapshots/
# Each numbered directory contains a snapshot

# 6. Pick a known-good snapshot and inspect it:
ls /mnt/root/.snapshots/<SNAPSHOT-ID>/snapshot/etc/
# Verify it looks like a complete root filesystem

# 7. Replace broken root with snapshot:
sudo mv /mnt/root /mnt/root.broken
#    ↑ snapshots are now at /mnt/root.broken/.snapshots/ (moved with root!)
sudo btrfs subvolume snapshot /mnt/root.broken/.snapshots/<SNAPSHOT-ID>/snapshot /mnt/root

# 8. Recreate .snapshots as a proper Btrfs subvolume:
#    (The snapshot contains an empty .snapshots directory placeholder,
#     but snapper needs it to be a subvolume)
sudo rmdir /mnt/root/.snapshots 2>/dev/null
sudo btrfs subvolume create /mnt/root/.snapshots
#    Old snapshots remain accessible at /mnt/root.broken/.snapshots/ if needed

# 9. Unmount and reboot:
sudo umount /mnt
sudo cryptsetup close luks-restore
sudo reboot

# 10. After confirmed boot — clean up (optional):
# sudo mount -o subvolid=5 /dev/mapper/<YOUR-LUKS-NAME> /mnt
# # Delete nested subvolumes first (required — btrfs can't delete non-empty subvolumes):
# for snap in /mnt/root.broken/.snapshots/*/snapshot; do sudo btrfs subvolume delete "$snap"; done
# sudo btrfs subvolume delete /mnt/root.broken/.snapshots
# sudo btrfs subvolume delete /mnt/root.broken
# sudo umount /mnt
```

> **Important**: After step 7, old snapshots remain in `root.broken/.snapshots/`. Step 8 creates a fresh `.snapshots` subvolume so snapper can create new snapshots going forward. The old `root.broken` subvolume can be deleted after confirming the system boots correctly.

> **Note on /boot**: On standard Fedora Btrfs installations, `/boot` is inside the root subvolume — it is restored together with the snapshot. If your system has `/boot` as a **separate partition** (non-standard), the kernel in `/boot` may not match the restored root's modules. In that case, boot from the restored snapshot's kernel via GRUB's "Advanced options" or reinstall the matching kernel after rollback.

### Forensics: compare system state over time

```bash
# What changed in the last 24 hours?
sudo snapper -c root diff <YESTERDAY-SNAPSHOT-ID>..<TODAY-SNAPSHOT-ID>

# What files were modified since last update?
sudo snapper -c root status <PRE-ID>..<POST-ID>
# Shows: +created, -deleted, c=content changed, t=type changed
```

---

## 6. Security Considerations

### Snapshots and LUKS

Snapshots reside on the same Btrfs volume, which is inside the LUKS container. They are encrypted at rest — no access without the LUKS passphrase. However:

| Aspect | Implication |
|--------|------------|
| Encryption | Snapshots are encrypted (same LUKS volume) |
| Deleted files | Files you "delete" may persist in snapshots until cleanup |
| Sensitive data | If you wrote a password to a file and deleted it, it may exist in a snapshot |
| Disk space | Snapshots use copy-on-write — only changed blocks consume extra space |

### Decision

```
Snapshot security trade-offs
├── Deleted file persistence:
│   ├── Risk: sensitive files survive in snapshots after deletion
│   ├── Mitigation: SPACE_LIMIT + timeline cleanup remove old snapshots
│   ├── For immediate removal: sudo snapper -c root delete <SNAPSHOT-ID>
│   └── Acceptable because: LUKS protects at-rest, snapshots are root-only
│
├── Rollback after security hardening:
│   ├── Risk: rolling back undoes security improvements
│   ├── Mitigation: verify system state after rollback
│   └── Always re-apply hardening changes after emergency rollback
│
└── Snapshot access control:
    ├── ALLOW_USERS="" — only root can manage snapshots
    ├── Snapshot directories are root-owned (700 permissions)
    └── Non-root users cannot browse or restore from snapshots
```

---

## 7. Complete Verification

```bash
# 1. Configs exist:
sudo snapper list-configs
# Expected: root (/) and home (/home)

# 2. Timers active:
systemctl is-enabled snapper-timeline.timer snapper-cleanup.timer
# Expected: enabled, enabled

# 3. Snapshots exist (after at least 1 hour):
sudo snapper -c root list | tail -5
# Expected: timeline snapshots listed

# 4. Btrfs filesystem:
findmnt -n -o FSTYPE /
# Expected: btrfs

# 5. Space usage:
sudo btrfs filesystem usage / | head -10
# Expected: plausible usage numbers

# 6. Verify retention settings:
sudo snapper -c root get-config | grep TIMELINE_LIMIT
# Expected: your configured retention values
```

---

## Important Notes

- **Btrfs required**: Snapper only works with Btrfs (Fedora default since F33). If you chose ext4 or XFS during installation, snapshots are not available.
- **No GRUB snapshot boot (Fedora 43)**: Unlike openSUSE, Fedora 43 does not configure GRUB to boot from snapshots. Rollback must be done via `snapper undochange` or Live USB. **Fedora 44 changes this** — the approved Change "BtrfsWithFullSystemSnapshots" introduces `snapm` + `boom` for automatic GRUB boot entries per snapshot. Community alternative for F43: [grub-btrfs](https://github.com/Antynea/grub-btrfs) (available via Copr, requires manual configuration).
- **Copy-on-write efficiency**: Snapshots only consume space for data that has changed since the snapshot was taken. An hourly snapshot of an idle system uses almost zero extra space.
- **dnf5 plugin gap**: The Snapper dnf plugin does not work with dnf5 (Fedora 43+ default). Use manual pre/post snapshots or a wrapper script (→ Doc 25).
- **Home snapshots**: Useful for recovering accidentally deleted personal files. `sudo snapper -c home list` shows available snapshots.
- **EMPTY_PRE_POST_CLEANUP**: Pre/post pairs where nothing actually changed (e.g., `dnf check-update` with no updates) are automatically deleted after 1 hour.
- **Snapshot count monitoring**: If you run many package operations, numbered snapshots can accumulate. `NUMBER_LIMIT=50` ensures automatic cleanup. Check with `sudo snapper -c root list | wc -l`.

---

*Previous: [19 — GPU & Secure Boot](19-gpu-secureboot.md)*
*Next: [21 — Kernel Module Blacklisting](21-kernel-module-blacklisting.md) — 46 blacklisted modules across 12 categories*
