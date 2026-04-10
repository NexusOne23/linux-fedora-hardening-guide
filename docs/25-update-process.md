# 🔄 Update Process

> Structured update routine: snapshots, packages, Flatpaks, firmware, integrity database.
> Applies to: Fedora Workstation 43+ | All hardware
> Some updates require reboot (kernel, firmware).

---

## Overview

A hardened system needs a disciplined update process. Updates must happen promptly (security patches), safely (with rollback capability), and completely (packages + Flatpaks + firmware + integrity database). Skipping any component leaves gaps.

### Why

| Risk | What happens |
|------|-------------|
| Delayed security patches | Known vulnerabilities remain exploitable |
| No snapshot before update | Broken update → no rollback possible |
| Flatpaks forgotten | Sandboxed apps stay vulnerable independently of dnf |
| Firmware forgotten | UEFI/NVMe/ME vulnerabilities persist below OS level |
| AIDE not rebuilt | Next integrity check flags all updated files as "tampered" |
| NVIDIA module not rebuilt | Next boot after kernel update → no display |

### Recommended update order

```
1. Snapshot (pre)       — rollback safety net
2. DNF upgrade          — packages, kernel, microcode
3. NVIDIA rebuild       — if kernel updated (NVIDIA users only)
4. Flatpak update       — sandboxed applications
5. Firmware check       — fwupd / LVFS
6. GPG signature check  — verify repo integrity
7. Integrity rebuild    — AIDE database
8. Snapshot (post)      — marks end of update
9. Reboot check         — if kernel/firmware changed
```

---

## 1. Automatic Security Updates (dnf5-automatic)

### Why

Manual updates are thorough but depend on you running them. Between manual runs, security patches should be applied automatically.

### Setup

```bash
# Install:
sudo dnf install dnf5-plugin-automatic

# Enable timer:
sudo systemctl enable --now dnf5-automatic.timer
```

### Configuration

The timer runs daily with a random delay (prevents all machines updating simultaneously):

```bash
systemctl cat dnf5-automatic.timer
# OnCalendar=*-*-* 6:00
# RandomizedDelaySec=60m
# Persistent=true (missed runs are caught up)
```

### NVIDIA users: akmods drop-in (critical)

If dnf5-automatic installs a kernel update overnight, the NVIDIA module must be rebuilt before the next boot. Without this drop-in, your system boots without a GPU driver.

Create `/etc/systemd/system/dnf5-automatic.service.d/akmods.conf`:

```ini
[Service]
ExecStartPost=/usr/sbin/akmods --force
ExecStartPost=/usr/sbin/dracut --regenerate-all --force
```

Then reload systemd:

```bash
sudo systemctl daemon-reload
```

### Decision

```
Automatic updates
├── Enable dnf5-automatic (recommended):
│   ├── Security patches applied daily without manual intervention
│   ├── Kernel updates trigger akmods rebuild (with drop-in)
│   ├── Complements manual update routine — not a replacement
│   └── Check logs: journalctl -u dnf5-automatic --since today
│
├── Manual only (acceptable for disciplined users):
│   ├── Full control over update timing
│   ├── Risk: security patches delayed until you remember
│   └── Must run updates at least weekly
│
└── NVIDIA users: akmods drop-in is MANDATORY
    ├── Without it: auto kernel update → no GPU driver on next boot
    ├── Symptom: black screen or low-resolution fallback after reboot
    └── The drop-in rebuilds nvidia.ko + initramfs after every dnf5-automatic run
```

---

## 2. Manual Update Routine

### Step 1: Snapper Pre-Snapshot

```bash
PRE_NUM=$(sudo snapper -c root create --type pre --print-number \
  --description "system update" --cleanup-algorithm number)
echo "Pre-snapshot: #${PRE_NUM}"
```

Everything that follows can be rolled back with:
```bash
sudo snapper -c root undochange ${PRE_NUM}..<POST_NUM>
```

> Skip if you don't use Snapper (→ Doc 20).

### Step 2: DNF Upgrade

```bash
sudo sh -c 'umask 022; dnf upgrade --refresh -y'
```

This updates all RPM packages — kernel, microcode, SELinux policies, system libraries, CLI tools.

> **Why `umask 022`?** If your shell uses `umask 027` (→ Doc 10), `sudo dnf upgrade` inherits it. GDM session files in `/usr/share/wayland-sessions/` get created with 640 instead of 644 — GDM cannot read them and **you cannot log in**. The `sh -c 'umask 022; ...'` subshell ensures DNF always creates files with correct permissions. See Doc 10 for full explanation.
>
> **Note**: `systemctl daemon-reload` is NOT needed manually — Fedora uses systemd file triggers that handle this automatically during RPM transactions.

### Step 3: NVIDIA Module Rebuild (NVIDIA users only)

If Step 2 installed a new kernel:

```bash
# Check if a new kernel was installed:
NEWEST_KERNEL=$(rpm -q kernel --last | head -1 | awk '{print $1}' | sed 's/kernel-//')
RUNNING_KERNEL=$(uname -r)

if [ "$NEWEST_KERNEL" != "$RUNNING_KERNEL" ]; then
  echo "New kernel detected: ${NEWEST_KERNEL}"
  sudo akmods --force --kernels "${NEWEST_KERNEL}"
  sudo dracut -f --kver "${NEWEST_KERNEL}"
fi
```

> **AMD/Intel GPU users**: Skip this step — your drivers are in-tree (→ Doc 19).

### Step 4: Flatpak Update

```bash
flatpak update -y
```

Updates all Flatpak apps and runtimes. Runs as your user — no sudo needed.

### Step 5: Firmware Check

```bash
sudo fwupdmgr refresh --force
fwupdmgr get-updates
```

If updates are available, install interactively (firmware updates require reboot):

```bash
sudo fwupdmgr update
```

> See Doc 24 for full firmware update details.

### Step 6: GPG Signature Safety Check

```bash
# Check for repos with disabled package signature verification:
grep -rInE '^\s*(gpgcheck|pkg_gpgcheck)\s*=\s*0' /etc/yum.repos.d/*.repo
# Expected: no output (all repos should have gpgcheck=1)
```

A repo with `gpgcheck=0` accepts unsigned packages — critical security risk. If found, investigate immediately.

### Step 7: AIDE Integrity Database Rebuild

```bash
sudo sh -c "aide --update > /var/log/aide/aide-update-$(date +%F).log 2>&1"
sudo sh -c 'cp -f /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz'
```

Must run **after** all package updates — otherwise AIDE flags every updated file as tampered on the next check.

> **AIDE exit codes**: 0=no changes, 1-7=bitmask (added/removed/changed — normal after updates), ≥14=real errors. Use `;` (not `&&`) before the `cp` command because exit codes 1-7 are expected.

### Step 8: Snapper Post-Snapshot

```bash
POST_NUM=$(sudo snapper -c root create --type post --pre-number "${PRE_NUM}" \
  --print-number --description "system update" --cleanup-algorithm number)
echo "Post-snapshot: #${POST_NUM}"
```

### Step 9: Reboot Check

```bash
# Compare running kernel vs newest installed:
echo "Running: $(uname -r)"
echo "Newest:  $(rpm -q kernel --last | head -1 | awk '{print $1}' | sed 's/kernel-//')"

# Check if services need restart:
sudo dnf needs-restarting
```

If the kernel was updated or firmware was flashed, reboot.

---

## 3. Update Script Template

You can combine the above steps into a script. Key design principles:

```bash
#!/bin/bash
# update-all.sh — Structured system update
# Run as your user (NOT root) — script uses sudo where needed

set -uo pipefail

ERRORS=0

# --- Cleanup trap: ensure post-snapshot even on abort ---
cleanup() {
  if [ -n "${PRE_NUM:-}" ]; then
    sudo snapper -c root create --type post --pre-number "${PRE_NUM}" \
      --description "system update (interrupted)" --cleanup-algorithm number 2>/dev/null
  fi
}
trap cleanup EXIT

# --- Step 1: Pre-snapshot ---
if command -v snapper &>/dev/null; then
  PRE_NUM=$(sudo snapper -c root create --type pre --print-number \
    --description "system update" --cleanup-algorithm number)
  echo "[1/9] Pre-snapshot: #${PRE_NUM}"
fi

# --- Step 2: DNF upgrade (umask 022 prevents GDM session file permission issues) ---
echo "[2/9] DNF upgrade..."
if ! sudo sh -c 'umask 022; dnf upgrade --refresh -y'; then
  ((ERRORS++))
fi

# --- Step 3: NVIDIA module rebuild (skip if no NVIDIA) ---
if command -v akmods &>/dev/null; then
  NEWEST=$(rpm -q kernel --last | head -1 | awk '{print $1}' | sed 's/kernel-//')
  if [ "$NEWEST" != "$(uname -r)" ]; then
    echo "[3/9] NVIDIA module rebuild for ${NEWEST}..."
    sudo akmods --force --kernels "${NEWEST}"
    sudo dracut -f --kver "${NEWEST}"
  fi
fi

# --- Step 4: Flatpak update ---
if command -v flatpak &>/dev/null; then
  echo "[4/9] Flatpak update..."
  flatpak update -y || ((ERRORS++))
fi

# --- Step 5: Firmware check ---
if command -v fwupdmgr &>/dev/null; then
  echo "[5/9] Firmware check..."
  sudo fwupdmgr refresh --force 2>/dev/null
  fwupdmgr get-updates 2>/dev/null || true
fi

# --- Step 6: GPG check ---
echo "[6/9] Repo signature check..."
if grep -rInE '^\s*(gpgcheck|pkg_gpgcheck)\s*=\s*0' /etc/yum.repos.d/*.repo 2>/dev/null; then
  echo "WARNING: Found repos with gpgcheck disabled!"
  ((ERRORS++))
fi

# --- Step 7: AIDE rebuild ---
if command -v aide &>/dev/null; then
  echo "[7/9] AIDE rebuild..."
  sudo sh -c "aide --update > /var/log/aide/aide-update-$(date +%F).log 2>&1"
  sudo sh -c 'cp -f /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz'
fi

# --- Step 8: Post-snapshot ---
if [ -n "${PRE_NUM:-}" ]; then
  POST_NUM=$(sudo snapper -c root create --type post --pre-number "${PRE_NUM}" \
    --print-number --description "system update" --cleanup-algorithm number)
  echo "[8/9] Post-snapshot: #${POST_NUM}"
fi

# --- Step 9: Reboot check ---
echo "[9/9] Reboot check..."
NEWEST=$(rpm -q kernel --last | head -1 | awk '{print $1}' | sed 's/kernel-//')
RUNNING=$(uname -r)
if [ "$NEWEST" != "$RUNNING" ]; then
  echo "REBOOT NEEDED: running ${RUNNING}, newest ${NEWEST}"
fi

# --- Summary ---
if [ "$ERRORS" -gt 0 ]; then
  echo "Completed with ${ERRORS} error(s) — check output above"
else
  echo "All steps completed successfully"
fi
```

### Design decisions

| Decision | Why |
|----------|-----|
| `set -uo pipefail` (no `-e`) | A failed step should not abort the entire script — continue and report |
| Cleanup trap | Post-snapshot is created even if you press Ctrl+C |
| `command -v` checks | Missing tools are skipped gracefully |
| Error counter | Summary at end shows total failures |
| No `sudo` for the whole script | Only specific commands run as root — Flatpak runs as user |
| Firmware interactive | Not included in `-y` automation — reboot decision is manual |

> **Customize this template**: Add or remove steps based on your setup. The NVIDIA rebuild (Step 3) auto-skips if akmods is not installed. arkenfox users should add a user.js update step. The template is a starting point, not a rigid prescription.

---

## 4. Post-Update Checks

After the update routine completes, verify:

```bash
# 1. Snapper pre/post pair exists:
sudo snapper -c root list --type pre-post | tail -3
# Expected: your update pair listed

# 2. No gpgcheck=0 repos:
grep -rInE '^\s*(gpgcheck|pkg_gpgcheck)\s*=\s*0' /etc/yum.repos.d/*.repo
# Expected: no output

# 3. AIDE log exists:
ls -la /var/log/aide/aide-update-$(date +%F).log
# Expected: today's log file

# 4. Kernel match (or reboot pending):
echo "Running: $(uname -r) | Newest: $(rpm -q kernel --last | head -1)"

# 5. Flatpak up to date:
flatpak update --no-deploy 2>/dev/null
# Expected: "Nothing to do" or similar
```

---

## Important Notes

- **Don't run as root**: The script uses `sudo` selectively. Running the entire script as root causes Flatpak operations to affect the system scope instead of your user scope.
- **Order matters**: DNF before AIDE (AIDE must capture new file hashes). Snapper before everything (rollback safety). Reboot check last.
- **AIDE exit codes are bitmasks**: After updates, exit codes 1-7 are expected (files changed/added/removed). Only ≥14 indicates a real error. Use `;` not `&&` before the database copy.
- **AIDE logs in /var/log/aide/**: Logs are stored securely (root:root, mode 700 directory) — not in world-writable `/tmp/`.
- **Firmware requires manual confirmation**: Firmware updates are intentionally left interactive — they require reboot and a failed flash can brick hardware.
- **GDM session file permissions**: If you use `umask 027` (→ Doc 10), check that GNOME session files in `/usr/share/xsessions/` and `/usr/share/wayland-sessions/` retain 644 permissions after DNF upgrades. Incorrect permissions (640) prevent GDM login.
- **dnf5-automatic + manual updates complement each other**: Automatic updates handle daily security patches. Manual runs handle the full routine (Flatpaks, firmware, AIDE, verification).

---

*Previous: [24 — Firmware Updates](24-firmware-updates.md)*
*Next: [26 — Package Removal](26-package-removal.md) — Attack surface reduction through package cleanup*
