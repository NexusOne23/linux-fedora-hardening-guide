# 🔐 PAM & Login Security

> Yescrypt hashing, empty password removal, SUID audit, core dumps 5× disabled, umask 027.
> Applies to: Fedora Workstation 43+ | Single-user desktop
> No reboot required for most changes.

---

## Overview

This document hardens the authentication and login subsystem:

- **PAM**: Remove `nullok` (no empty passwords), enforce Yescrypt hashing
- **umask 027**: New files are not world-readable (with GDM tradeoff documented)
- **securetty**: Empty file — no root TTY login
- **sudo**: 2-minute credential cache, verified secure defaults
- **Core dumps**: Disabled at 5 independent layers
- **SUID audit**: Remove unnecessary SUID bits from up to 12 binaries

---

## 1. PAM Configuration

### What

PAM (Pluggable Authentication Modules) controls how authentication works on your system. Two critical changes: remove `nullok` and enforce Yescrypt password hashing.

### Why

- **`nullok`**: Fedora's default PAM config includes `nullok` on `pam_unix.so`, which allows accounts with empty passwords to authenticate. This must be removed.
- **Yescrypt**: The default password hashing algorithm. It is memory-hard (like Argon2), making GPU-based brute-force attacks impractical.

### authselect Profile

Fedora uses `authselect` to manage PAM configuration. Use it instead of editing PAM files directly:

```bash
# Check current profile:
$ authselect current
# Expected: Profile with "without-nullok" enabled

# If nullok is still active:
$ sudo authselect select local --force without-nullok with-silent-lastlog
```

### Key PAM settings

| Setting | Status | Detail |
|---------|--------|--------|
| `nullok` | **Removed** | `pam_unix.so` has no `nullok` — empty passwords forbidden |
| Password hashing | `yescrypt` | Memory-hard KDF, GPU-resistant |
| Fail delay | 2 seconds | `pam_faildelay.so delay=2000000` — delay after failed login |
| Password quality | Active | `pam_pwquality.so` — complexity checks enforced |
| Limits | Active | `pam_limits.so` — respects `/etc/security/limits.conf` |

### Decision: `pam_faillock` (account lockout)

> **Note:** On Fedora 43+, `pam_faillock` is active by default in the PAM stack — even without the `with-faillock` authselect option. The defaults (`deny=3`, `unlock_time=600`) are too aggressive for a single-user desktop where CLI tools (AI assistants, automation scripts) may fire multiple `sudo` commands in rapid succession — each failed attempt counts toward lockout.

```
pam_faillock (account lockout after failed logins)
├── Relaxed settings (recommended for single-user desktop):
│   ├── deny = 10 (default 3 is too aggressive for batch-sudo tools)
│   ├── unlock_time = 120 (2 minutes — matches sudo timestamp_timeout)
│   ├── fail_interval = 900 (15 min window — default, fine)
│   ├── Why not default 3? CLI tools that can't prompt for a password
│   │   (e.g., AI coding assistants, cron jobs) generate "no terminal"
│   │   errors that PAM counts as failed auth attempts
│   └── File: /etc/security/faillock.conf
│
├── Disable entirely (acceptable for maximum-security single-user):
│   ├── LUKS passphrase is the first barrier (pre-boot)
│   ├── SSH service is disabled — no remote brute-force vector
│   ├── GDM login is inherently slow (GUI + 2s fail delay)
│   └── Risk: no lockout after repeated failed local login attempts
│
├── Strict (multi-user or SSH enabled):
│   ├── deny = 3 (default)
│   ├── unlock_time = 600 (default)
│   └── Appropriate when network-accessible login exists
│
└── Config file: /etc/security/faillock.conf
    deny = 10
    unlock_time = 120
    fail_interval = 900
```

---

## 2. Password Hashing: Yescrypt

### What

Yescrypt is a memory-hard Key Derivation Function used for password hashing in `/etc/shadow`.

### Why

| Hash method | Security level |
|-------------|---------------|
| DES | Insecure (obsolete) |
| MD5 | Insecure |
| SHA-512 | Secure, but GPU-bruteforceable |
| **Yescrypt** | **Memory-hard, GPU-resistant** |

Yescrypt requires significant RAM per brute-force attempt, making GPU-parallel attacks impractical. It provides similar security to Argon2id (used in LUKS).

### Configuration

Both locations must agree:

```bash
# /etc/login.defs:
ENCRYPT_METHOD YESCRYPT

# PAM (in system-auth):
password sufficient pam_unix.so yescrypt shadow use_authtok
```

### Verify

```bash
$ grep "ENCRYPT_METHOD" /etc/login.defs
# Expected: ENCRYPT_METHOD YESCRYPT
```

---

## 3. umask 027

### What

Set the default file creation mask to `027`, making new files unreadable by "other" users.

### Why

The default umask `022` creates files as `644` (world-readable) and directories as `755` (world-accessible). With umask `027`:
- New files: `640` (owner read/write, group read, others nothing)
- New directories: `750` (owner full, group read/execute, others nothing)

On a single-user desktop this matters less, but it prevents information leaks if additional users or services are ever added.

### Configuration (three locations)

| File | Setting |
|------|---------|
| `/etc/login.defs` | `UMASK 027` |
| `/etc/profile.d/99-security-umask.sh` | `umask 027` |
| `/etc/bashrc` | `umask 027` (replace existing `umask 022` / `umask 002`) |

### File: `/etc/profile.d/99-security-umask.sh`

```bash
# Hardened umask: new files get 640, directories get 750
umask 027
```

### Known Tradeoff: GDM Session-File Permissions

**This is a real issue you MUST know about:**

When DNF installs or updates GNOME session files (`/usr/share/wayland-sessions/*.desktop`, `/usr/share/xsessions/*.desktop`), it creates them with permissions inherited from the shell's umask. With umask `027`, these files get `640` instead of `644`. GDM requires world-read (`644`) to display login sessions. With `640`, GDM finds no sessions and **you cannot log in**.

### Solution: Always run DNF with umask 022

```bash
# Instead of: sudo dnf upgrade
# Always use:
$ sudo sh -c 'umask 022; dnf upgrade --refresh -y'
```

**Why this works:** `sudo dnf upgrade` inherits your shell's umask `027`. `sudo sh -c 'umask 022; ...'` starts a new subshell with its own umask — DNF sees `022` and creates files with correct permissions (`644`/`755`). Your interactive shell keeps umask `027`.

> **Note:** `dnf5-automatic` (systemd timer) does NOT have this problem — systemd services default to umask `022`.

### Fallback check

If DNF was accidentally run with umask `027`:

```bash
# Find session files that are not world-readable:
$ find /usr/share/wayland-sessions /usr/share/xsessions -name "*.desktop" ! -perm -o=r 2>/dev/null

# Fix if any found:
$ sudo chmod 644 /usr/share/wayland-sessions/*.desktop /usr/share/xsessions/*.desktop 2>/dev/null
```

---

## 4. securetty (Root TTY Login)

### What

An empty `/etc/securetty` file — no TTYs are listed for root console login.

### Why

On Fedora 43, `pam_securetty.so` is NOT in the PAM stack, so the empty file alone has no effect. However, the root account is **locked** (`passwd -S root` → `L`) — no password is set, the account is locked. Root access is exclusively through `sudo`.

### Verify

```bash
$ wc -c /etc/securetty
# Expected: 0 (empty file)

$ sudo passwd -S root
# Expected: root L ... (L = locked)
```

---

## 5. sudo Hardening

### What

Reduce the sudo credential cache timeout and verify secure defaults.

### Why

After entering your password for `sudo`, the credentials are cached for 5 minutes by default. During this window, any program running as your user can execute `sudo` commands without re-authentication. Reducing this to 2 minutes shrinks the exposure window.

### Configuration

Create a drop-in file:

```bash
$ echo 'Defaults timestamp_timeout=2' | sudo tee /etc/sudoers.d/hardening
$ sudo chmod 440 /etc/sudoers.d/hardening
$ sudo visudo -cf /etc/sudoers.d/hardening
# Expected: parsed OK
```

### Verified secure defaults (no change needed)

| Parameter | Default (sudo 1.9.8+) | What it does |
|-----------|----------------------|-------------|
| `use_pty` | Yes | sudo runs in its own PTY — prevents terminal hijacking |
| `!visiblepw` | Yes | No password prompt in insecure terminals |
| `passwd_tries` | 3 | Max 3 password attempts |

### Decision

```
sudo timestamp_timeout
├── Set 2 if:    Maximum security — 2-minute cache window
├── Set 5 if:    Default — reasonable for most users
├── Set 0 if:    Password required EVERY time (may be annoying)
└── What breaks: Nothing — only affects how often you re-enter your password
```

### Root access paths (all blocked except sudo)

| Path | Status |
|------|--------|
| Root TTY login | Locked — root account has no password |
| SSH root login | Forbidden — `PermitRootLogin no` ([Doc 09](09-ssh-hardening.md)) |
| SSH service | Disabled |
| `sudo` | Only escalation path (your user in `wheel` group) |

---

## 6. Core Dumps Disabled (5 layers)

### What

Core dumps are disabled at five independent layers for defense-in-depth.

### Why

Core dumps contain a snapshot of process memory at crash time. This can include passwords, encryption keys, authentication tokens, and other sensitive data in RAM. An attacker with read access to core dumps could extract these.

### Five layers

| Layer | Setting | Effect |
|-------|---------|--------|
| `/etc/security/limits.conf` | `* hard core 0` | All users: no core dump |
| Shell | `ulimit -c 0` | Shell limit: no core dump |
| sysctl | `fs.suid_dumpable = 0` | SUID programs: no core dump ([Doc 02](02-sysctl-hardening.md)) |
| systemd | `systemd-coredump.socket` masked | systemd-coredump cannot activate ([Doc 08](08-service-minimization.md)) |
| `/etc/systemd/coredump.conf` | `Storage=none`, `ProcessSizeMax=0` | Safety net if socket is unmasked |

### File: `/etc/systemd/coredump.conf`

```ini
[Coredump]
Storage=none
ProcessSizeMax=0
```

### Verify

```bash
$ ulimit -c
# Expected: 0

$ sysctl fs.suid_dumpable
# Expected: 0

$ systemctl is-enabled systemd-coredump.socket
# Expected: masked
```

> **Why 5 layers?** Each alone is sufficient. But if one is accidentally reset (kernel update, config change), the others still prevent core dumps.

---

## 7. SUID Binary Audit

### What

Remove the SUID (Set User ID) bit from binaries that don't need root execution privileges.

### Why

Every SUID binary runs as **root** regardless of who executes it. A vulnerability (buffer overflow, format string, race condition) in any SUID binary is an instant privilege escalation. The `nosuid` mount option on `/home`, `/tmp`, `/dev/shm` prevents *new* SUID files, but *existing* system SUID binaries in `/usr/bin/` remain active.

### Decision: Which SUID bits to remove

```
SUID binary audit strategy
├── Safe to remove on any single-user desktop:
│   ├── chfn          — change finger info (not needed)
│   ├── chsh          — change login shell (not needed)
│   ├── gpasswd       — group password management (obsolete feature)
│   ├── newgrp        — runtime group switching (single-user: not needed)
│   ├── chage         — password aging info (root can use without SUID)
│   └── userhelper    — legacy GUI su-helper (GNOME uses Polkit instead)
│
├── Safe to remove if you DON'T use the specific feature:
│   ├── pam_timestamp_check   — only for sudo timestamp cache validation
│   ├── fusermount-glusterfs  — only if no GlusterFS
│   ├── libgtop_server2       — GNOME System Monitor (works without SUID)
│   ├── qemu-bridge-helper    — only if no VM bridge networking
│   └── spice-client-*        — only if no SPICE VM display
│
├── KEEP (required for desktop functionality):
│   ├── sudo, su                    — privilege escalation
│   ├── passwd, unix_chkpwd         — password management + PAM auth
│   ├── mount, umount               — USB/removable media as user
│   ├── pkexec, polkit-agent-*      — Polkit authorization (GNOME)
│   ├── fusermount, fusermount3     — FUSE mounts (Nautilus, GVFS)
│   ├── grub2-set-bootflag          — grub-boot-success.timer runs as user, needs SUID to write /boot/grub2/grubenv
│   └── crontab                     — cron job management (if used)
│
├── KEEP (conditional — hardware-specific):
│   ├── nvidia-modprobe             — NVIDIA GPU device node creation
│   └── chrome-sandbox              — Chromium-based app sandbox (VSCode, Electron apps)
│
└── After DNF updates: Re-check — RPM restores original permissions on package updates
```

### How to remove

```bash
# Safe for any single-user desktop:
$ sudo chmod u-s /usr/bin/chfn /usr/bin/chsh /usr/bin/gpasswd \
    /usr/bin/newgrp /usr/bin/chage /usr/bin/userhelper

# Additional (if applicable — check before removing):
$ sudo chmod u-s /usr/bin/pam_timestamp_check 2>/dev/null
$ sudo chmod u-s /usr/bin/fusermount-glusterfs 2>/dev/null
$ sudo chmod u-s /usr/libexec/libgtop_server2 2>/dev/null
$ sudo chmod u-s /usr/libexec/qemu-bridge-helper 2>/dev/null
$ sudo chmod u-s /usr/libexec/spice-gtk-x86_64/spice-client-glib-usb-acl-helper 2>/dev/null
```

### How to audit

```bash
# List all SUID binaries on the system:
$ find /usr -perm -4000 -type f 2>/dev/null | sort

# After removing all recommended, you should see ~14 remaining (varies by installed packages)
```

### What breaks

- `chfn` / `chsh`: Cannot change finger info or login shell as non-root (use `sudo` instead)
- `gpasswd`: Cannot set group passwords (feature rarely used since the 1990s)
- Others: Only if you use the specific feature (GlusterFS, QEMU bridge, SPICE)

> **DNF restores SUID bits on package updates.** After `dnf upgrade`, re-check with `find /usr -perm -4000`. The update process in [Doc 25](25-update-process.md) includes this check.

---

## 8. Login Defaults

### File: `/etc/login.defs` (relevant entries)

| Parameter | Value | Why |
|-----------|-------|-----|
| `UMASK` | `027` | Default file creation mask |
| `ENCRYPT_METHOD` | `YESCRYPT` | Password hash algorithm |
| `PASS_MAX_DAYS` | `99999` | No password expiry (single-user desktop) |
| `PASS_MIN_LEN` | `8` | Minimum password length |

### Decision

```
Password expiry (PASS_MAX_DAYS)
├── Set 99999 if:  Single-user desktop with strong password (recommended)
├── Set 90 if:     Corporate policy requires password rotation
└── Note:          Password rotation is discouraged by NIST SP 800-63B —
                   strong passwords should not be forced to change
```

---

## 🔧 Applying Changes

### Step 1: Remove nullok from PAM

```bash
$ sudo authselect select local --force without-nullok with-silent-lastlog
```

### Step 2: Set umask 027

```bash
$ sudo sed -i 's/^UMASK.*/UMASK\t\t027/' /etc/login.defs

$ sudo tee /etc/profile.d/99-security-umask.sh << 'UMASK'
umask 027
UMASK

$ sudo sed -i 's/umask 002/umask 027/g; s/umask 022/umask 027/g' /etc/bashrc
```

### Step 3: Empty securetty

```bash
$ sudo truncate -s 0 /etc/securetty
```

### Step 4: sudo timestamp timeout

```bash
$ echo 'Defaults timestamp_timeout=2' | sudo tee /etc/sudoers.d/hardening
$ sudo chmod 440 /etc/sudoers.d/hardening
$ sudo visudo -cf /etc/sudoers.d/hardening
```

### Step 5: Disable core dumps

```bash
$ echo "* hard core 0" | sudo tee -a /etc/security/limits.conf

$ sudo systemctl mask systemd-coredump.socket

$ sudo tee /etc/systemd/coredump.conf << 'COREDUMP'
[Coredump]
Storage=none
ProcessSizeMax=0
COREDUMP
```

### Step 6: Remove unnecessary SUID bits

```bash
$ sudo chmod u-s /usr/bin/chfn /usr/bin/chsh /usr/bin/gpasswd
```

---

## ✅ Complete Verification

```bash
# 1. No nullok in PAM:
$ grep "nullok" /etc/pam.d/system-auth /etc/pam.d/password-auth
# Expected: No output

# 2. Yescrypt active:
$ grep "ENCRYPT_METHOD" /etc/login.defs
# Expected: ENCRYPT_METHOD YESCRYPT

# 3. umask 027:
$ umask
# Expected: 0027

# 4. securetty empty:
$ wc -c /etc/securetty
# Expected: 0

# 5. Core dumps disabled:
$ ulimit -c
# Expected: 0
$ systemctl is-enabled systemd-coredump.socket
# Expected: masked

# 6. SUID audit:
$ find /usr -perm -4000 -type f 2>/dev/null | wc -l
# Expected: ~18 (may vary by installed packages)

# 7. sudo drop-in valid:
$ sudo visudo -cf /etc/sudoers.d/hardening
# Expected: parsed OK
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| `nullok` removed | Cannot have accounts with empty passwords | Prevents empty-password login |
| umask 027 | New files not world-readable + GDM session tradeoff | Privacy from other users/services |
| GDM/DNF tradeoff | Must run DNF with `umask 022` subshell | Prevents session-file lockout |
| sudo timeout 2min | Re-enter password more often | Smaller credential cache window |
| Core dumps 5× | Cannot debug crashes via core dump | No sensitive data in crash dumps |
| SUID removed (3+) | Must use `sudo` for chfn/chsh/gpasswd | Reduced privilege escalation surface |
| faillock relaxed (10/120) | More attempts before lockout than default | Avoids false lockouts from batch-sudo tools |

---

## Notes

- **authselect manages PAM.** Manual edits to `/etc/pam.d/system-auth` and `password-auth` will be overwritten by `authselect select`. Use authselect features instead.
- **Yescrypt vs Argon2id:** Both are memory-hard. Yescrypt is the glibc/PAM standard for `/etc/shadow`. Argon2id is used by LUKS ([Doc 22](22-luks-encryption.md)). Both provide GPU resistance.
- **No `TMOUT`:** Shell auto-logout (`TMOUT`) is a server measure. On a GNOME desktop, the screen locks after idle (`idle-delay`). `TMOUT` would interrupt terminal sessions during reading, thinking, or code review — counterproductive.
- **NIST SP 800-63B** discourages forced password rotation. A strong, unique password that doesn't expire is more secure than a weak password changed every 90 days.
- **`pam_securetty.so` is not in Fedora's PAM stack.** The empty securetty file is belt-and-suspenders. Root login is blocked because the account is locked, not because of securetty.

---

*Previous: [09 — SSH Hardening](09-ssh-hardening.md)*
*Next: [11 — DNS & NTP](11-dns-ntp.md) — DNSSEC, DNS-over-HTTPS, chrony with NTS*
