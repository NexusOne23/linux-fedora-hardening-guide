# 🔑 SSH Hardening

> Key-only authentication, root denied, service disabled by default.
> Applies to: Fedora Workstation 43+ | OpenSSH
> No reboot required.

---

## Overview

On a hardened desktop, the SSH server should be **disabled by default**. You don't need remote access to a workstation most of the time. However, the configuration is hardened — so if you ever temporarily enable SSH, the security settings apply immediately.

### Strategy

```
SSH on a hardened desktop
├── Service disabled:     sshd not running — zero network exposure
├── Config hardened:      If enabled, only key-based auth, no root login
├── Firewall blocked:     Even if sshd runs, firewall drops inbound SSH
└── Temporary use only:   Enable → do your work → disable again
```

> **Servers are different.** If you run SSH as a persistent service (e.g., on a server), you need additional hardening (fail2ban, port change, etc.) that is beyond the scope of this desktop guide.

---

## 1. Configuration

### What

Six parameters appended to `/etc/ssh/sshd_config` that restrict authentication and access.

### Why

OpenSSH's defaults are reasonable but not hardened:
- Root login is allowed (with key) — should be forbidden entirely
- Password authentication is enabled — vulnerable to brute-force
- 6 auth attempts per connection — too many
- 2-minute login grace time — too generous
- All users can log in — should be restricted to your user only

### File: `/etc/ssh/sshd_config`

Append at the end:

```bash
# === Hardening ===
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers <YOUR-USERNAME>
```

### Parameters

| Parameter | Value | Default | Why |
|-----------|-------|---------|-----|
| `PermitRootLogin` | `no` | `prohibit-password` | Root login completely forbidden — not even with a key |
| `PasswordAuthentication` | `no` | `yes` | Key-only authentication — brute-force impossible |
| `X11Forwarding` | `no` | `no` | No X11 forwarding (Wayland desktop, X11 not needed) |
| `MaxAuthTries` | `3` | `6` | Max 3 authentication attempts per connection |
| `LoginGraceTime` | `30` | `120` | 30 seconds to authenticate, then connection dropped |
| `AllowUsers` | `<YOUR-USERNAME>` | (all users) | Only your user can log in via SSH |

### Secure defaults (already correct, no change needed)

| Parameter | Default | Why secure |
|-----------|---------|-----------|
| `PubkeyAuthentication` | `yes` | Key authentication active |
| `PermitEmptyPasswords` | `no` | Empty passwords forbidden |
| `StrictModes` | `yes` | Checks permissions on `~/.ssh` |
| `HostbasedAuthentication` | `no` | Host-based auth disabled |
| `PermitTunnel` | `no` | No SSH tunneling |
| `UsePAM` | `yes` | PAM integration active (Fedora default) |

### Decision

```
SSH key generation
├── Generate if:  You plan to ever use SSH (even temporarily)
│   └── ssh-keygen -t ed25519
│       (Ed25519 is the strongest and fastest option)
├── Skip if:      You will never use SSH
└── Note:         Generate the key BEFORE disabling password auth
```

---

## 2. Firewall Protection

Even if sshd is accidentally enabled, multiple firewall layers prevent external access:

| Layer | Protection |
|-------|-----------|
| `strict-wan` zone | Target DROP, no SSH service — port 22 unreachable from LAN |
| `drop` zone | Target DROP — no SSH via VPN interfaces |
| VPN killswitch nftables | Only VPN server IP allowed on physical NIC — no inbound SSH |
| `block-lan-out` | RFC1918 DROP outbound — cannot initiate SSH to LAN devices |

**Result:** SSH is only reachable via `localhost` — not from any network interface. To allow temporary SSH access from the LAN, you would need to explicitly add the SSH service to your zone.

---

## 3. Temporary SSH Access

If you need SSH temporarily (e.g., for file transfer or remote debugging):

```bash
# Enable and start:
$ sudo systemctl start sshd

# When done — disable again:
$ sudo systemctl stop sshd

# Verify it's stopped:
$ systemctl is-active sshd
# Expected: inactive
```

> **Do not `enable` sshd** unless you need it to survive reboots. `start` runs it once; `enable` makes it persistent.

### If you need LAN access temporarily

```bash
# Open SSH in strict-wan zone:
$ sudo firewall-cmd --zone=strict-wan --add-service=ssh

# When done — remove:
$ sudo firewall-cmd --zone=strict-wan --remove-service=ssh

# Note: Without --permanent, this resets on firewall reload or reboot
```

---

## 🔧 Applying Changes

### Step 1: Harden SSH config

```bash
$ sudo tee -a /etc/ssh/sshd_config << 'SSH'

# === Hardening ===
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers <YOUR-USERNAME>
SSH
```

> **Edit** the file to replace `<YOUR-USERNAME>` with your actual username (`whoami`).

### Step 2: Generate SSH key (if needed)

```bash
$ ssh-keygen -t ed25519
# Accept default path (~/.ssh/id_ed25519)
# Set a strong passphrase
```

### Step 3: Ensure service is disabled

```bash
$ sudo systemctl disable sshd
$ sudo systemctl stop sshd
```

### Step 4: Verify no drop-in configs interfere

```bash
$ ls /etc/ssh/sshd_config.d/
# Should be empty — no additional configs that could override your settings
```

---

## ✅ Complete Verification

```bash
# 1. Service disabled:
$ systemctl is-enabled sshd
# Expected: disabled

# 2. Service not running:
$ systemctl is-active sshd
# Expected: inactive

# 3. Config syntax valid (without starting the service):
$ sudo sshd -t
# Expected: No output (= no errors)

# 4. No port 22 listening:
$ sudo ss -tlnp | grep ":22 "
# Expected: No output

# 5. Config values correct:
$ sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|maxauthtries|allowusers"
# Expected:
#   permitrootlogin no
#   passwordauthentication no
#   maxauthtries 3
#   allowusers <your-username>
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| sshd disabled | No remote access by default | Zero network exposure |
| Key-only auth | Must manage SSH keys | Brute-force impossible |
| `AllowUsers` | Only listed users can SSH in | Prevents unauthorized user access |
| `MaxAuthTries 3` | Fewer retry attempts | Faster lockout of attackers |
| `LoginGraceTime 30` | Must authenticate within 30s | Reduces connection slot exhaustion |

---

## Notes

- **`PasswordAuthentication no`** means you MUST have an SSH key set up before enabling sshd. Without a key, you will be locked out of SSH (local console login still works).
- **`AllowUsers`** is more restrictive than `AllowGroups` — explicitly lists permitted users. If you add system users later, remember to add them here.
- **Drop-in configs** in `/etc/ssh/sshd_config.d/` can override your settings. Fedora ships this directory empty, but packages or tools might add files there. Check periodically.
- **Ed25519** is recommended over RSA for new keys. It's faster, has a fixed key size (256-bit), and is considered more secure against implementation errors.
- **Crypto policy:** Fedora's system-wide crypto policy (`DEFAULT`) applies to SSH and all TLS connections. Do **not** change it to `FUTURE` — it raises the minimum RSA key size to 3072 bits, which breaks TLS to any server with RSA-2048 certificates (pypi.org, Docker Hub, many CDNs) and rejects RSA-2048 SSH keys. `DEFAULT` already disables SHA-1 signatures, which is sufficient.

---

*Previous: [08 — Service Minimization](08-service-minimization.md)*
*Next: [10 — PAM & Login Security](10-pam-login-security.md) — Yescrypt hashing, SUID audit, core dumps disabled, umask 027*
