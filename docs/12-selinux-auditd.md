# 📊 SELinux & auditd

> SELinux enforcing with targeted policy + hardened booleans, 38 immutable audit rules across 18 categories.
> Applies to: Fedora Workstation 43+ | All hardware
> auditd rule changes require reboot (immutable mode).

---

## Overview

Two complementary monitoring systems:

- **SELinux**: Mandatory Access Control — restricts what each process can do, even if running as root
- **auditd**: System audit — logs security-relevant events (file changes, privilege escalation, module loading)

SELinux **prevents** attacks. auditd **detects** them. Together they provide both enforcement and visibility.

---

## 1. SELinux

### What

SELinux (Security-Enhanced Linux) enforces mandatory access control policies. Every process runs in a security context that limits what files it can access, what ports it can bind to, and what syscalls it can make — regardless of Unix permissions.

### Why

Without SELinux, a compromised service (e.g., a browser exploit, a vulnerable daemon) gets the full permissions of the user running it. With SELinux enforcing, even a compromised root process is confined to its policy — it cannot access files, ports, or resources outside its designated context.

### Configuration

#### File: `/etc/selinux/config`

```
SELINUX=enforcing
SELINUXTYPE=targeted
```

| Parameter | Value | Why |
|-----------|-------|-----|
| `SELINUX` | `enforcing` | MAC rules enforced — violations blocked and logged |
| `SELINUXTYPE` | `targeted` | Only targeted processes are confined (Fedora default) |

### What SELinux protects

| Protection | Detail |
|-----------|--------|
| Process isolation | Each service runs in its own SELinux context |
| File access | Only permitted contexts can access specific files |
| Network | Only permitted ports/protocols per service |
| Memory protection | `actual (secure)` — execmem/execstack protection |
| Kernel modules | Only signed modules (lockdown + SELinux combined) |

### Decision

```
SELinux mode
├── enforcing (REQUIRED):
│   ├── Fedora default — do NOT change to permissive or disabled
│   ├── Disabling SELinux removes an entire security layer
│   └── If something breaks: fix the policy, don't disable SELinux
│
├── permissive (testing only):
│   ├── Logs violations but doesn't enforce
│   └── Use temporarily to diagnose policy issues, then switch back
│
└── disabled (NEVER):
    ├── Completely removes MAC protection
    ├── Re-enabling requires full filesystem relabel (slow)
    └── There is no valid reason to disable SELinux on Fedora
```

### Hardened SELinux Booleans

Two Fedora-default booleans weaken W^X (Write XOR Execute) memory protection. They are `on` by default in the targeted policy and should be disabled on hardened systems:

| Boolean | Default | Recommended | What it controls |
|---------|---------|-------------|-----------------|
| `selinuxuser_execstack` | `on` | **`off`** | Allows programs to execute code on the stack — foundation for classic buffer overflow exploits |
| `selinuxuser_execmod` | `on` | **`off`** | Allows marking modified memory as executable (W→X transition) — enables JIT spray and ROP chain attacks |

#### Decision

```
SELinux memory protection booleans
├── Set BOTH to off (recommended):
│   ├── No modern desktop application needs executable stack
│   ├── Firefox, Chromium, Electron apps, Flatpak apps all work with both off
│   └── Significantly strengthens kernel memory protection enforcement
│
├── Keep execmod on if:
│   ├── You run Java applications (JVM JIT compiler)
│   ├── You use Wine (Windows emulation)
│   └── You use other JIT-heavy runtimes that modify code pages
│
├── Keep execstack on if:
│   └── Almost never — only legacy compiled binaries with -z execstack flag
│
└── What breaks: If an app fails to start, check: ausearch -m AVC -ts recent
    Undo: sudo setsebool -P selinuxuser_execstack on (or execmod)
```

#### Apply

```bash
$ sudo setsebool -P selinuxuser_execstack off
$ sudo setsebool -P selinuxuser_execmod off
```

#### Verify

```bash
$ getsebool selinuxuser_execstack selinuxuser_execmod
# Expected: selinuxuser_execstack --> off
#           selinuxuser_execmod --> off
```

> **Note:** These booleans are not mentioned in most Fedora hardening guides despite being default `on` and significantly weakening kernel memory protection. The `-P` flag makes the change persistent across reboots.

### Known false-positive AVCs

| AVC | Context | Status |
|-----|---------|--------|
| AIDE → `xdm_var_run_t` (GDM) | AIDE reads GDM files during integrity scan | Benign — expected |
| AIDE → `dosfs_t` (EFI) | AIDE reads EFI partition during scan | Benign — expected |

> Use `sudo ausearch -m AVC -ts recent` to check for new AVCs. Known false positives above can be safely ignored.

### Verify

```bash
$ getenforce
# Expected: Enforcing

$ grep "^SELINUX=" /etc/selinux/config
# Expected: SELINUX=enforcing
```

---

## 2. auditd Configuration

### What

The Linux audit daemon logs security-relevant system events to `/var/log/audit/audit.log`.

### Why

If an attacker modifies system files, loads kernel modules, or escalates privileges, auditd records it. Combined with AIDE ([Doc 13](13-aide.md)) for file integrity and SELinux for enforcement, audit provides the detection layer.

### File: `/etc/audit/auditd.conf` (key parameters)

| Parameter | Value | Why |
|-----------|-------|-----|
| `log_format` | `ENRICHED` | Human-readable log entries (names instead of UIDs) |
| `flush` | `INCREMENTAL_ASYNC` | Async flushing — balance of performance and reliability |
| `max_log_file` | `8` | Max 8 MB per log file |
| `num_logs` | `5` | Max 5 rotated log files (total: ~40 MB) |
| `space_left_action` | `SYSLOG` | Warn in syslog when disk space is low |
| `disk_full_action` | `SUSPEND` | Pause auditing if disk is full (don't lose other data) |

---

## 3. Audit Rules (38 rules)

> **Critical — Fedora default disables syscall auditing:** Fedora ships `/etc/audit/rules.d/audit.rules` with `-a task,never` which **silently prevents all file watch and syscall audit rules from generating events**. You MUST replace this file (see Applying Changes below) or no audit rules will work — they load without errors but never fire.

### Rule format: why `-a always,exit -F path=` instead of `-w`

The traditional `-w /path -p wa` watch syntax is deprecated. It is inode-based and can silently stop working when:
- Files are replaced via write-new + rename (common editor pattern)
- The filesystem uses copy-on-write (Btrfs)
- Inode numbers exceed 32-bit range (Btrfs uses 64-bit inodes)

The explicit syscall format `-a always,exit -F arch=b64 -F path=/path -F perm=wa` uses path-based resolution and is reliable on all filesystems including Btrfs.

### File: `/etc/audit/rules.d/99-hardening.rules`

```bash
## === Buffer and failure behavior ===
-b 8192
-f 1

## === GNOME Session Files ===
-a always,exit -F arch=b64 -F dir=/usr/share/gnome-session/sessions -F perm=wa -k gnome_session_files

## === Identity & Authentication ===
-a always,exit -F arch=b64 -F path=/etc/passwd -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/shadow -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/group -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/gshadow -F perm=wa -k identity

## === Sudo/Su Configuration ===
-a always,exit -F arch=b64 -F path=/etc/sudoers -F perm=wa -k sudoers
-a always,exit -F arch=b64 -F dir=/etc/sudoers.d -F perm=wa -k sudoers

## === SSH Configuration ===
-a always,exit -F arch=b64 -F path=/etc/ssh/sshd_config -F perm=wa -k sshd_config
-a always,exit -F arch=b64 -F dir=/etc/ssh/sshd_config.d -F perm=wa -k sshd_config

## === Audit Configuration (protect itself) ===
-a always,exit -F arch=b64 -F dir=/etc/audit -F perm=wa -k audit_config
-a always,exit -F arch=b64 -F dir=/etc/audit/rules.d -F perm=wa -k audit_config

## === Kernel Parameters ===
-a always,exit -F arch=b64 -F path=/etc/sysctl.conf -F perm=wa -k sysctl
-a always,exit -F arch=b64 -F dir=/etc/sysctl.d -F perm=wa -k sysctl

## === Kernel Modules ===
-a always,exit -F arch=b64 -F dir=/etc/modprobe.d -F perm=wa -k modprobe
-a always,exit -F arch=b64 -S init_module,finit_module,delete_module -k kernel_modules
-a always,exit -F arch=b32 -S init_module,finit_module,delete_module -k kernel_modules

## === Bootloader & initramfs ===
-a always,exit -F arch=b64 -F path=/etc/default/grub -F perm=wa -k bootloader
-a always,exit -F arch=b64 -F dir=/etc/dracut.conf.d -F perm=wa -k bootloader

## === Systemd Units ===
-a always,exit -F arch=b64 -F dir=/etc/systemd/system -F perm=wa -k systemd

## === Firewall ===
-a always,exit -F arch=b64 -F dir=/etc/firewalld -F perm=wa -k firewall

## === Network ===
-a always,exit -F arch=b64 -F dir=/etc/NetworkManager/system-connections -F perm=wa -k network_config

## === User Management Binaries ===
-a always,exit -F arch=b64 -F path=/usr/sbin/useradd -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/userdel -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/usermod -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupadd -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupdel -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupmod -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/passwd -F perm=x -k user_mgmt

## === Privilege Escalation ===
-a always,exit -F arch=b64 -F path=/usr/bin/sudo -F perm=x -k sudo_usage
-a always,exit -F arch=b64 -F path=/usr/bin/su -F perm=x -k su_usage

## === Privilege Execution (commands run as root by non-system users) ===
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=-1 -k priv_exec

## === LUKS/Crypto ===
-a always,exit -F arch=b64 -F path=/etc/crypttab -F perm=wa -k luks

## === Login & Security Configuration ===
-a always,exit -F arch=b64 -F path=/etc/login.defs -F perm=wa -k login_config
-a always,exit -F arch=b64 -F dir=/etc/security -F perm=wa -k security_config

## === Cron ===
-a always,exit -F arch=b64 -F path=/etc/crontab -F perm=wa -k cron
-a always,exit -F arch=b64 -F dir=/etc/cron.d -F perm=wa -k cron
-a always,exit -F arch=b64 -F dir=/etc/cron.daily -F perm=wa -k cron

## === PAM ===
-a always,exit -F arch=b64 -F dir=/etc/pam.d -F perm=wa -k pam_changes

## === Immutable (MUST BE LAST RULE) ===
-e 2
```

### Rule Categories

| Category | Rules | Key | What it monitors |
|----------|-------|-----|-----------------|
| GNOME session | 1 | `gnome_session_files` | GNOME session .desktop files (GDM login) |
| Identity | 4 | `identity` | passwd, shadow, group, gshadow |
| Sudo | 2 | `sudoers` | sudoers, sudoers.d |
| SSH | 2 | `sshd_config` | sshd_config, sshd_config.d |
| Audit config | 2 | `audit_config` | /etc/audit, rules.d |
| Kernel params | 2 | `sysctl` | sysctl.conf, sysctl.d |
| Kernel modules | 3 | `modprobe` / `kernel_modules` | modprobe.d + init/delete_module syscalls |
| Bootloader | 2 | `bootloader` | grub, dracut.conf.d |
| Systemd | 1 | `systemd` | /etc/systemd/system unit files |
| Firewall | 1 | `firewall` | /etc/firewalld |
| Network | 1 | `network_config` | NM connection files |
| User management | 7 | `user_mgmt` | useradd/del/mod, groupadd/del/mod, passwd |
| Privilege escalation | 2 | `sudo_usage` / `su_usage` | sudo, su execution |
| Privilege execution | 1 | `priv_exec` | Any command run as root by a real user |
| LUKS | 1 | `luks` | crypttab changes |
| Login/security | 2 | `login_config` / `security_config` | login.defs, /etc/security/ |
| Cron | 3 | `cron` | crontab, cron.d, cron.daily |
| PAM | 1 | `pam_changes` | /etc/pam.d |
| **Total** | **38** | | + 2 config lines (`-b`, `-f`) + 1 immutable (`-e 2`) |

### Rule Syntax Explained

| Syntax | Meaning |
|--------|---------|
| `-a always,exit -F arch=b64 -F path=/file -F perm=wa -k key` | Watch file: log write + attribute changes |
| `-a always,exit -F arch=b64 -F dir=/dir -F perm=wa -k key` | Watch directory (recursive): log write + attribute changes |
| `-a always,exit -F arch=b64 -F path=/bin -F perm=x -k key` | Watch binary: log execution |
| `-a always,exit -F arch=b64 -S syscall -k key` | Syscall audit: log every invocation |
| `-a always,exit -F euid=0 -F auid>=1000` | Log commands run as root (euid=0) by real users (auid>=1000) |
| `-b 8192` | Set backlog buffer to 8192 events |
| `-f 1` | On failure: log to syslog (don't panic) |
| `-e 2` | **Immutable**: no rule changes allowed at runtime |

> **Why `-F arch=b64` on every rule?** Without specifying the architecture, the kernel evaluates the rule against every syscall on both 32-bit and 64-bit paths. Specifying `arch=b64` limits evaluation to 64-bit syscalls only — significantly better performance on x86_64 systems.

---

## 4. Immutable Mode (`-e 2`)

### What

Once `-e 2` is loaded (must be the **last rule**), the audit ruleset becomes immutable. No rules can be added, modified, or deleted until the next reboot.

### Why

An attacker with root access could disable audit rules (`auditctl -D`) to hide their activity. With immutable mode, this is impossible — even root cannot change the rules. To modify the audit rules, the attacker would need to:

1. Edit the rule file in `/etc/audit/rules.d/` (detected by AIDE — [Doc 13](13-aide.md))
2. Reboot the system (suspicious and logged)
3. Wait for the new rules to load

This creates a strong forensic trail.

### Applying rule changes

Because of immutable mode, the workflow for changing audit rules is:

```bash
# 1. Edit the rule file:
$ sudo nano /etc/audit/rules.d/99-hardening.rules

# 2. Validate syntax:
$ sudo augenrules --check

# 3. Reboot (required — augenrules --load cannot override -e 2):
$ sudo reboot

# 4. Verify after reboot:
$ sudo auditctl -s | grep "enabled"
# Expected: enabled 2
```

> **`augenrules --load` and `systemctl restart auditd` do NOT work** with immutable mode. A reboot is the only way to apply changes.

---

## 5. Real-Time Desktop Notifications (Optional)

### What

A systemd service that monitors the audit log in real-time and sends GNOME desktop notifications when critical security events are detected — identity file changes, kernel module loading, firewall modifications, and more.

### Why

auditd logs to `/var/log/audit/audit.log` — silently. Without notifications, an attacker could modify `/etc/passwd`, load a kernel module, or change firewall rules, and you wouldn't know until you manually run `ausearch`. The notification service bridges this gap: critical events trigger an immediate desktop alert.

### Architecture: Why a systemd service, not an audisp plugin

audisp plugins run inside the `auditd_t` SELinux context — which is intentionally restricted. An audisp plugin cannot:
- Write to `/run/` (rate-limit state)
- Execute `notify-send` (desktop notification)
- Access the D-Bus session bus (user communication)

A separate systemd service avoids these restrictions entirely. It tails `/var/log/audit/audit.log` and runs in `unconfined_service_t` — with full access to send desktop notifications.

### Script: `/usr/local/bin/audit-notify.sh`

```bash
#!/bin/bash
# auditd Real-Time Desktop Notification via audit.log tail
# Runs as systemd service — NOT as audisp plugin (avoids auditd_t SELinux context)
# Rate-limited: max 1 notification per key per 60 seconds

DISPLAY_USER="<YOUR-USERNAME>"
DISPLAY_UID=$(id -u "$DISPLAY_USER" 2>/dev/null || echo "1000")
RATE_DIR="/run/audit-notify"
RATE_LIMIT=60
CRITICAL_KEYS="identity|sudoers|kernel_modules|modprobe|audit_config|bootloader|sysctl|systemd|firewall|pam_changes|network_config|user_mgmt|su_usage|luks|login_config|security_config|cron"

mkdir -p "$RATE_DIR" 2>/dev/null

send_notification() {
    sudo -u "$DISPLAY_USER" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${DISPLAY_UID}/bus" \
        notify-send --urgency=critical --icon=dialog-warning "$1" "$2" 2>/dev/null
}

rate_check() {
    local now; now=$(date +%s)
    local last_file="${RATE_DIR}/${1}"
    if [ -f "$last_file" ]; then
        local last; last=$(cat "$last_file" 2>/dev/null || echo "0")
        (( now - last < RATE_LIMIT )) && return 1
    fi
    echo "$now" > "$last_file"; return 0
}

# Wait for audit log to exist
while [ ! -f /var/log/audit/audit.log ]; do sleep 1; done

# Tail the audit log (start from end, follow new entries)
tail -n 0 -F /var/log/audit/audit.log | while read -r line; do
    key=""
    if [[ "$line" =~ key=\"([^\"]+)\" ]]; then
        key="${BASH_REMATCH[1]}"
    elif [[ "$line" =~ key=([^[:space:]]+) ]]; then
        key="${BASH_REMATCH[1]}"
    fi

    [ -z "$key" ] && continue
    [[ "$key" == "(null)" ]] && continue
    [[ ! "$key" =~ ^($CRITICAL_KEYS)$ ]] && continue

    rate_check "$key" || continue

    send_notification "auditd: ${key} event" \
        "Check: sudo ausearch -k ${key} -ts recent"
done
```

> Replace `<YOUR-USERNAME>` with your actual username (`whoami`).

### How it works

The service tails `/var/log/audit/audit.log` in real-time. For each new log entry, it:

1. Extracts the audit `key` from the record (matches your audit rules in Section 3)
2. Checks if the key is in the critical list (17 of 21 unique keys, covering 14 of 18 rule categories)
3. Rate-limits: max 1 notification per key per 60 seconds (prevents notification flood during `dnf upgrade`)
4. Sends a GNOME desktop notification with the event type and investigation command

### Monitored keys (17 keys, covering 14 of 18 categories)

| Key | Triggers on | Why critical |
|-----|------------|-------------|
| `identity` | passwd/shadow/group changes | User account tampering |
| `sudoers` | sudoers/sudoers.d changes | Privilege configuration |
| `kernel_modules` | Module load/unload | Rootkit installation |
| `modprobe` | modprobe.d changes | Blacklist circumvention |
| `audit_config` | Audit rule changes | Evidence tampering |
| `bootloader` | GRUB/dracut changes | Boot chain manipulation |
| `sysctl` | Kernel parameter changes | Security policy weakening |
| `systemd` | Unit file changes | Service manipulation |
| `firewall` | firewalld changes | Network policy weakening |
| `pam_changes` | PAM config changes | Auth bypass |
| `network_config` | NM connection changes | Network redirection |
| `user_mgmt` | useradd/userdel/usermod | Unauthorized accounts |
| `su_usage` | su execution | Unusual on single-user |
| `luks` | crypttab changes | Encryption config |
| `login_config` | login.defs changes | Password policy weakening |
| `security_config` | /etc/security/ changes | faillock/limits/access tampering |
| `cron` | crontab/cron.d/cron.daily changes | Persistence via scheduled tasks |

> **Note:** Some categories use multiple keys (e.g., "Kernel modules" uses both `kernel_modules` and `modprobe`). 17 unique keys cover 14 of the 18 rule categories.

### Excluded keys (4 keys, covering 4 of 18 categories)

| Key | Why excluded |
|-----|-------------|
| `sudo_usage` | Too noisy — every `sudo` command triggers this |
| `priv_exec` | Too noisy — every root command by a real user |
| `sshd_config` | Low frequency — SSH is disabled, config rarely changes |
| `gnome_session_files` | Low frequency — only changes during GNOME/DNF updates |

### systemd Service: `/etc/systemd/system/audit-notify.service`

```ini
[Unit]
Description=Audit Event Desktop Notification Service
After=auditd.service
Requires=auditd.service

[Service]
Type=simple
ExecStart=/usr/local/bin/audit-notify.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Rate-limit state: `/etc/tmpfiles.d/audit-notify.conf`

```ini
d /run/audit-notify 0750 root root -
```

The rate-limit state is stored in `/run/` (tmpfs) — cleared on every reboot, no persistent state on disk.

### Decision

```
auditd desktop notifications
├── Enable (recommended):
│   ├── Real-time visibility of security-critical changes
│   ├── Rate-limited — won't flood during updates
│   ├── No impact on auditd operation or log files
│   └── Runs as independent systemd service (not inside auditd)
│
├── Skip if:
│   ├── Headless server (no GNOME session)
│   └── You monitor audit logs via external SIEM/log aggregation
│
└── What breaks: nothing — the notification is advisory only
```

---

## 🔧 Applying Changes

### Step 1: Verify SELinux

```bash
$ getenforce
# If not "Enforcing":
$ sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
$ sudo reboot
```

### Step 2: Fix Fedora's default audit.rules (CRITICAL)

Fedora ships `/etc/audit/rules.d/audit.rules` with `-a task,never` which **disables all syscall auditing**. Your audit rules will load without errors but never generate events. Replace it:

```bash
$ sudo tee /etc/audit/rules.d/audit.rules << 'EOF'
## Clear all rules on startup
-D
EOF
```

> **Without this step, none of your audit rules will work.** The rules load and `auditctl -l` shows them, but no events are ever generated. This is Fedora's default "performance mode" — it must be disabled for security auditing.

### Step 3: Create audit rules

```bash
$ sudo tee /etc/audit/rules.d/99-hardening.rules << 'AUDIT'
-b 8192
-f 1
-a always,exit -F arch=b64 -F dir=/usr/share/gnome-session/sessions -F perm=wa -k gnome_session_files
-a always,exit -F arch=b64 -F path=/etc/passwd -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/shadow -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/group -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/gshadow -F perm=wa -k identity
-a always,exit -F arch=b64 -F path=/etc/sudoers -F perm=wa -k sudoers
-a always,exit -F arch=b64 -F dir=/etc/sudoers.d -F perm=wa -k sudoers
-a always,exit -F arch=b64 -F path=/etc/ssh/sshd_config -F perm=wa -k sshd_config
-a always,exit -F arch=b64 -F dir=/etc/ssh/sshd_config.d -F perm=wa -k sshd_config
-a always,exit -F arch=b64 -F dir=/etc/audit -F perm=wa -k audit_config
-a always,exit -F arch=b64 -F dir=/etc/audit/rules.d -F perm=wa -k audit_config
-a always,exit -F arch=b64 -F path=/etc/sysctl.conf -F perm=wa -k sysctl
-a always,exit -F arch=b64 -F dir=/etc/sysctl.d -F perm=wa -k sysctl
-a always,exit -F arch=b64 -F dir=/etc/modprobe.d -F perm=wa -k modprobe
-a always,exit -F arch=b64 -S init_module,finit_module,delete_module -k kernel_modules
-a always,exit -F arch=b32 -S init_module,finit_module,delete_module -k kernel_modules
-a always,exit -F arch=b64 -F path=/etc/default/grub -F perm=wa -k bootloader
-a always,exit -F arch=b64 -F dir=/etc/dracut.conf.d -F perm=wa -k bootloader
-a always,exit -F arch=b64 -F dir=/etc/systemd/system -F perm=wa -k systemd
-a always,exit -F arch=b64 -F dir=/etc/firewalld -F perm=wa -k firewall
-a always,exit -F arch=b64 -F dir=/etc/NetworkManager/system-connections -F perm=wa -k network_config
-a always,exit -F arch=b64 -F path=/usr/sbin/useradd -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/userdel -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/usermod -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupadd -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupdel -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/groupmod -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/sbin/passwd -F perm=x -k user_mgmt
-a always,exit -F arch=b64 -F path=/usr/bin/sudo -F perm=x -k sudo_usage
-a always,exit -F arch=b64 -F path=/usr/bin/su -F perm=x -k su_usage
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=-1 -k priv_exec
-a always,exit -F arch=b64 -F path=/etc/crypttab -F perm=wa -k luks
-a always,exit -F arch=b64 -F path=/etc/login.defs -F perm=wa -k login_config
-a always,exit -F arch=b64 -F dir=/etc/security -F perm=wa -k security_config
-a always,exit -F arch=b64 -F path=/etc/crontab -F perm=wa -k cron
-a always,exit -F arch=b64 -F dir=/etc/cron.d -F perm=wa -k cron
-a always,exit -F arch=b64 -F dir=/etc/cron.daily -F perm=wa -k cron
-a always,exit -F arch=b64 -F dir=/etc/pam.d -F perm=wa -k pam_changes
-e 2
AUDIT
```

> **Important:** Only ONE rules file should contain your rules. Remove any other files in `/etc/audit/rules.d/` except `audit.rules` (which has only `-D`) and your `99-hardening.rules`. Duplicate rules from multiple files cause conflicts.

### Step 4: Harden SELinux booleans

```bash
$ sudo setsebool -P selinuxuser_execstack off
$ sudo setsebool -P selinuxuser_execmod off
```

### Step 5: Load rules and reboot

```bash
$ sudo augenrules --load
$ sudo reboot
```

### Step 6: Install notification service (optional)

```bash
# Create the script (use single-quoted heredoc to prevent variable expansion):
$ sudo tee /usr/local/bin/audit-notify.sh << 'NOTIFY'
#!/bin/bash
DISPLAY_USER="<YOUR-USERNAME>"
DISPLAY_UID=$(id -u "$DISPLAY_USER" 2>/dev/null || echo "1000")
RATE_DIR="/run/audit-notify"
RATE_LIMIT=60
CRITICAL_KEYS="identity|sudoers|kernel_modules|modprobe|audit_config|bootloader|sysctl|systemd|firewall|pam_changes|network_config|user_mgmt|su_usage|luks|login_config|security_config|cron"
mkdir -p "$RATE_DIR" 2>/dev/null

send_notification() {
    sudo -u "$DISPLAY_USER" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${DISPLAY_UID}/bus" \
        notify-send --urgency=critical --icon=dialog-warning "$1" "$2" 2>/dev/null
}

rate_check() {
    local now; now=$(date +%s)
    local last_file="${RATE_DIR}/${1}"
    if [ -f "$last_file" ]; then
        local last; last=$(cat "$last_file" 2>/dev/null || echo "0")
        (( now - last < RATE_LIMIT )) && return 1
    fi
    echo "$now" > "$last_file"; return 0
}

while [ ! -f /var/log/audit/audit.log ]; do sleep 1; done

tail -n 0 -F /var/log/audit/audit.log | while read -r line; do
    key=""
    if [[ "$line" =~ key=\"([^\"]+)\" ]]; then key="${BASH_REMATCH[1]}"
    elif [[ "$line" =~ key=([^[:space:]]+) ]]; then key="${BASH_REMATCH[1]}"; fi
    [ -z "$key" ] && continue
    [[ "$key" == "(null)" ]] && continue
    [[ ! "$key" =~ ^($CRITICAL_KEYS)$ ]] && continue
    rate_check "$key" || continue
    send_notification "auditd: ${key} event" \
        "Check: sudo ausearch -k ${key} -ts recent"
done
NOTIFY
$ sudo chmod 755 /usr/local/bin/audit-notify.sh

# Create rate-limit tmpfiles entry:
$ sudo tee /etc/tmpfiles.d/audit-notify.conf << 'TMP'
d /run/audit-notify 0750 root root -
TMP
$ sudo systemd-tmpfiles --create /etc/tmpfiles.d/audit-notify.conf

# Create systemd service:
$ sudo tee /etc/systemd/system/audit-notify.service << 'SVC'
[Unit]
Description=Audit Event Desktop Notification Service
After=auditd.service
Requires=auditd.service

[Service]
Type=simple
ExecStart=/usr/local/bin/audit-notify.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
SVC

$ sudo systemctl daemon-reload
$ sudo systemctl enable --now audit-notify.service
```

> **Edit** the script to replace `<YOUR-USERNAME>` with your actual username. Use a **single-quoted heredoc** (`<< 'NOTIFY'`) to prevent shell variable expansion — the `$` signs in the script must be preserved literally.

---

## ✅ Complete Verification

```bash
# 1. SELinux enforcing:
$ getenforce
# Expected: Enforcing

# 2. SELinux config:
$ grep "^SELINUX=" /etc/selinux/config
# Expected: SELINUX=enforcing

# 3. Audit enabled and immutable:
$ sudo auditctl -s | grep "enabled"
# Expected: enabled 2

# 4. Audit rule count:
$ sudo auditctl -l | wc -l
# Expected: 38

# 5. Immutable test (should fail):
$ sudo auditctl -D 2>&1
# Expected: "The audit system is in immutable mode, no rule changes allowed"

# 6. File watch test (MUST generate events):
$ sudo touch /etc/passwd
$ sudo ausearch -k identity -ts recent | grep comm=\"touch\"
# Expected: SYSCALL event with comm="touch" and key="identity"

# 7. Audit log exists:
$ sudo ls -la /var/log/audit/audit.log
# Expected: File exists with recent timestamp

# 8. Notification service active (if installed):
$ systemctl is-active audit-notify.service
# Expected: active

# 9. Test notification (triggers identity key):
$ sudo bash -c 'rm -f /run/audit-notify/identity; touch /etc/passwd'
# Expected: Desktop notification "auditd: identity event"

# 10. Rate-limit directory exists:
$ ls -ld /run/audit-notify
# Expected: drwxr-x--- root root

# 11. Fedora audit.rules fixed:
$ sudo cat /etc/audit/rules.d/audit.rules
# Expected: only "-D" — must NOT contain "-a task,never"
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| SELinux enforcing | Some apps may need policy adjustments | Full mandatory access control |
| `-e 2` immutable | Rule changes require reboot | Attacker cannot disable audit at runtime |
| 38 audit rules | Increased log volume | Comprehensive monitoring of security-critical changes |
| `ENRICHED` log format | Slightly larger logs | Human-readable without lookup tools |
| `priv_exec` rule | Logs ALL root commands by real users | Detects unauthorized privilege use |
| Desktop notifications | Requires active GNOME session | Real-time visibility of security events |
| Fedora audit.rules replaced | Must maintain custom audit.rules across updates | Syscall auditing actually works |

---

## Notes

- **`-e 2` (immutable):** Rule changes require a full reboot. `augenrules --load` and `systemctl restart auditd` cannot override immutable mode. This is by design.
- **Fedora's default `audit.rules` disables syscall auditing.** The file contains `-a task,never` which prevents ALL file watch and syscall audit rules from generating events. You MUST replace this file with one containing only `-D` (clear rules). Without this change, `auditctl -l` shows rules as loaded but they never fire — a silent failure that is extremely difficult to diagnose. Check after every major Fedora upgrade.
- **Why not `-w` (deprecated watch syntax)?** The `-w /path -p wa` format is inode-based. On Btrfs (Fedora's default filesystem), copy-on-write operations can change inodes, causing watches to silently stop tracking files. The explicit `-a always,exit -F path=` format uses path-based resolution and is reliable on all filesystems.
- **Audit log rotation:** `max_log_file=8` MB × `num_logs=5` = max 40 MB of audit logs. Adjust if you need longer retention.
- **AIDE integration:** auditd monitors changes in real-time. AIDE ([Doc 13](13-aide.md)) verifies file integrity periodically. Complementary protection layers.
- **SELinux AVCs:** `sudo ausearch -m AVC -ts recent` shows recent policy violations. Known false positives (AIDE → GDM, AIDE → EFI) are benign.
- **Searching audit logs:** Use `ausearch -k <key>` to search by rule key. Example: `sudo ausearch -k sudo_usage -ts today` shows all sudo invocations today.
- **Notification service vs audisp plugin:** audisp plugins run in the `auditd_t` SELinux context which blocks desktop notifications. The systemd service approach tails the audit log file from outside `auditd_t`, avoiding SELinux restrictions entirely.
- **Rate limiting:** The notification script limits alerts to 1 per key per 60 seconds. During `dnf upgrade`, dozens of `systemd` and `modprobe` events fire — without rate limiting, your desktop would be flooded with notifications.
- **Single-quoted heredoc for script installation:** When installing the notification script via `tee << 'NOTIFY'`, the single quotes around `NOTIFY` prevent the shell from expanding `$` variables. Without single quotes, the script on disk will have mangled variable names and fail with syntax errors.
- **Only one rules file:** Remove any extra files in `/etc/audit/rules.d/` (e.g., `50-*.rules`, `51-*.rules`) to prevent duplicate or conflicting rules. All custom rules belong in `99-hardening.rules`.

---

*Previous: [11 — DNS & NTP](11-dns-ntp.md)*
*Next: [13 — AIDE](13-aide.md) — Daily file integrity monitoring*
