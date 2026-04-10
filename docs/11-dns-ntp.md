# ⏱️ DNS & NTP

> DNSSEC, DNS-over-HTTPS (Quad9), LLMNR/mDNS disabled, chrony with NTS (4 servers).
> Applies to: Fedora Workstation 43+ | All hardware
> No reboot required — restart services after config changes.

---

## Overview

This document secures two critical infrastructure services:

- **DNS**: systemd-resolved with DNSSEC, LLMNR/mDNS disabled, Firefox uses independent DoH
- **NTP**: chrony with Network Time Security (NTS) — authenticated time only, no unauthenticated fallback

### Why these matter

- **DNS without DNSSEC/DoH**: An attacker can forge DNS responses, redirecting you to malicious sites
- **NTP without NTS**: An attacker can manipulate your system clock, bypassing TLS certificate expiry checks, HSTS, and Kerberos

---

## 1. DNS: systemd-resolved

### 1.1 Configuration

#### File: `/etc/systemd/resolved.conf`

```ini
[Resolve]
DNSSEC=allow-downgrade
LLMNR=no
MulticastDNS=no
```

| Parameter | Value | Why |
|-----------|-------|-----|
| `DNSSEC` | `allow-downgrade` | Validate DNSSEC when available, don't enforce (see detailed explanation below) |
| `LLMNR` | `no` | Link-Local Multicast Name Resolution disabled — LAN discovery risk |
| `MulticastDNS` | `no` | mDNS disabled — no Avahi/Bonjour traffic |

### 1.2 DNSSEC: Why `allow-downgrade` — A Conscious Decision

> **TL;DR:** `DNSSEC=yes` breaks ALL DNS through a VPN because VPN resolvers strip RRSIG records (RFC 4035 by design). Use `allow-downgrade` + Firefox DoH (Quad9) for real DNSSEC protection.

This section explains a critical DNS architecture issue that affects **all VPN users**, not just a specific provider.

**Key insight:** VPN encryption and DNSSEC protect at **different layers** — they are not interchangeable:
- **VPN** = transport encryption (network level) — protects the pipe
- **DNSSEC** = response authenticity (DNS protocol level) — protects the content

A DNSSEC downgrade attack happens at the DNS protocol level (via manipulated server responses), not at the network level. VPN encryption does not protect against it.

#### The three DNSSEC modes

| Mode | Behavior | Risk |
|------|----------|------|
| `allow-downgrade` | Validates DNSSEC when available; falls back to unsigned if server doesn't offer DNSSEC | Vulnerable to downgrade attacks (attacker strips DNSSEC) |
| `yes` | Enforces DNSSEC validation — domains with broken/missing DNSSEC fail to resolve | Safer, but breaks domains with misconfigured DNSSEC |
| `no` | No DNSSEC validation | No protection |

#### Why `allow-downgrade` is the correct choice

**1. VPN DNS resolvers strip DNSSEC records (RFC 4035 by design)**

Recursive DNS resolvers (including those operated by VPN providers) validate DNSSEC **server-side** and then strip the RRSIG (signature) records before returning the response to clients. This is correct behavior per RFC 4035 — the resolver validates, the client trusts the resolver.

You can verify this with any VPN DNS resolver:

```bash
$ dig @<YOUR-VPN-DNS> example.com SOA +dnssec +short
# If no RRSIG records appear → the resolver strips them (expected behavior)
```

With `DNSSEC=yes`, systemd-resolved would expect RRSIG records in every response. Since the VPN resolver strips them, **all DNS queries would fail** — no internet.

> **This is NOT a VPN-specific bug.** It's DNS architecture: recursive resolvers validate and strip RRSIG. The same behavior occurs with any VPN provider's DNS.

**2. systemd-resolved's DNSSEC implementation is unreliable**

Fedora builds systemd with `-Ddefault-dnssec=no` and treats DNSSEC bugs as WONTFIX. Multiple open issues (systemd/systemd#38401, systemd/systemd#32570, systemd/systemd#36681) document fundamental design problems in resolved's DNSSEC validation.

**3. Firefox DoH provides real DNSSEC protection**

Firefox with Quad9 DoH (TRR Mode 3) gets DNSSEC validation **server-side from Quad9** — independently of systemd-resolved. Browser DNS (the majority of security-sensitive queries) is protected regardless of the `DNSSEC` setting in resolved.

**4. `DNSSEC=yes` breaks domains with misconfigured DNSSEC**

Broken DNSSEC configurations are more common than expected. `DNSSEC=yes` would cause hard failures for these domains.

#### Decision

```
DNSSEC setting
├── VPN users:
│   ├── Set: allow-downgrade (MUST — VPN resolver strips RRSIG)
│   ├── DNSSEC=yes WILL BREAK all DNS queries through VPN
│   └── Real DNSSEC protection via Firefox DoH (Quad9, TRR Mode 3)
│
├── Non-VPN users (direct ISP DNS):
│   ├── Set: allow-downgrade (recommended — safest option)
│   ├── Set: yes (if your DNS resolver returns RRSIG records — test with dig)
│   └── Note: systemd-resolved DNSSEC is unreliable regardless
│
└── Regardless of setting: Use Firefox DoH for browser DNSSEC
```

### 1.3 DNS Routing (VPN users)

With a VPN active, DNS should route exclusively through the tunnel:

| Interface | DNS | Why |
|-----------|-----|-----|
| VPN tunnel | VPN provider's DNS | All system DNS goes through encrypted tunnel |
| Physical NIC | None | No DNS on physical interface — prevents DNS leaks |
| Killswitch interface | None | No DNS on killswitch dummy |

Verify:

```bash
$ resolvectl status
# Expected: Only VPN interface shows a DNS server
# Physical interface should show "Current Scopes: none"
```

**DNS path:** Application → `127.0.0.53` (resolved stub) → VPN tunnel → VPN DNS resolver → internet

> Your VPN client typically configures this automatically. If not, set DNS via NetworkManager: `sudo nmcli con mod "<VPN-CONNECTION>" ipv4.dns <YOUR-VPN-DNS>`

### 1.4 Listener

```bash
$ ss -ulnp | grep ":53 "
# Expected: Only 127.0.0.53 and 127.0.0.54 — loopback only, not reachable from network
```

### 1.5 Firefox: Independent DoH

Firefox bypasses systemd-resolved entirely and uses its own DNS-over-HTTPS:

| Setting | Value | Why |
|---------|-------|-----|
| `network.trr.mode` | `3` | DoH-only — no fallback to system DNS |
| `network.trr.uri` | `https://dns.quad9.net/dns-query` | Quad9 — DNSSEC-validating, privacy-respecting |
| `network.trr.bootstrapAddr` | `9.9.9.9` | Bootstrap IP — no system DNS needed to resolve DoH hostname |
| `network.dns.disableIPv6` | `true` | Matches system-level IPv6 disable |

**Two independent DNS resolvers:**
1. systemd-resolved (VPN DNS) — for system queries (CLI tools, package managers, services)
2. Quad9 DoH (Firefox) — for all browser queries (DNSSEC validated server-side)

> Firefox DoH configuration is detailed in [Doc 16 — Firefox Hardening](16-firefox-browser.md).

---

## 2. NTP: chrony with NTS

### 2.1 What is NTS?

**Network Time Security** (RFC 8915) is the authenticated version of NTP. Without NTS, an attacker can forge NTP responses and manipulate your system clock. This enables:

- **TLS bypass**: Make expired certificates appear valid
- **HSTS bypass**: Reset HSTS timers to downgrade HTTPS to HTTP
- **Kerberos attacks**: Invalidate or extend Kerberos tickets
- **Log manipulation**: Falsify timestamps in security logs
- **TOTP bypass**: Desynchronize time-based one-time passwords

NTS prevents all of this by adding TLS authentication to NTP communication.

### 2.2 Configuration

#### File: `/etc/chrony.conf`

```bash
# NTS-authenticated NTP servers (4 sources for redundancy)
server time.cloudflare.com iburst nts
server ntppool1.time.nl iburst nts ipv4
server ptbtime1.ptb.de iburst nts ipv4
server nts.netnod.se iburst nts ipv4

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
ntsdumpdir /var/lib/chrony
leapseclist /usr/share/zoneinfo/leap-seconds.list
logdir /var/log/chrony

# Command interface: Unix socket only (no network)
bindcmdaddress /var/run/chrony/chronyd.sock
cmdport 0
```

### 2.3 NTS Servers

| Server | Location | Stratum | NTS |
|--------|----------|---------|-----|
| `time.cloudflare.com` | Global (Anycast) | 3 | Yes |
| `ntppool1.time.nl` | Netherlands | 1 | Yes |
| `ptbtime1.ptb.de` | Germany (PTB, national metrology institute) | 1 | Yes |
| `nts.netnod.se` | Sweden | 1 | Yes |

**Why 4 servers?** chrony needs at least 3 sources to detect a rogue server (Byzantine fault tolerance). 4 provides redundancy if one is temporarily unreachable.

**Why no `pool.ntp.org`?** NTP pool servers don't support NTS. Including them would create an unauthenticated fallback — an attacker could block the NTS servers and force your system to use the unprotected pool.

### 2.4 Command Interface Hardening

```bash
bindcmdaddress /var/run/chrony/chronyd.sock
cmdport 0
```

| Parameter | Value | Why |
|-----------|-------|-----|
| `bindcmdaddress` | Unix socket path | `chronyc` communicates via Unix socket only, not over network |
| `cmdport` | `0` | UDP port 323 is NOT opened |

**Why is `cmdport 0` needed?**

`bindcmdaddress 127.0.0.1` alone is insufficient — chrony opens separate sockets per protocol family. Without `cmdport 0`, an IPv6 command socket remains open. `cmdport 0` disables the UDP listener entirely — only the Unix socket remains.

### 2.5 Other Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `driftfile` | `/var/lib/chrony/drift` | Store hardware clock frequency drift |
| `makestep 1.0 3` | 1s threshold, 3 updates | Immediately correct if > 1s off in first 3 updates |
| `rtcsync` | Active | Periodically sync hardware clock |
| `ntsdumpdir` | `/var/lib/chrony` | Persist NTS cookies/keys across restarts |
| `leapseclist` | `/usr/share/zoneinfo/leap-seconds.list` | Leap second table |
| `ipv4` flag | On 3 of 4 servers | Force IPv4 (matches system-level IPv6 disable) |

### Decision

```
NTP server selection
├── Use ONLY NTS servers — no unauthenticated fallback
├── Minimum 3 servers (Byzantine fault tolerance)
├── Recommended: 4 servers from different organizations/countries
├── Do NOT include pool.ntp.org — no NTS support, creates downgrade path
└── ipv4 flag: Add to all servers except Cloudflare (which handles IPv4 automatically)

Command interface
├── bindcmdaddress: Unix socket (not 127.0.0.1)
├── cmdport: 0 (no UDP listener)
└── Why: Prevents any network access to chrony command interface
```

---

## 🔧 Applying Changes

### Step 1: Configure systemd-resolved

```bash
$ sudo tee /etc/systemd/resolved.conf << 'RESOLVED'
[Resolve]
DNSSEC=allow-downgrade
LLMNR=no
MulticastDNS=no
RESOLVED

$ sudo systemctl restart systemd-resolved
```

### Step 2: Configure chrony

```bash
$ sudo tee /etc/chrony.conf << 'CHRONY'
server time.cloudflare.com iburst nts
server ntppool1.time.nl iburst nts ipv4
server ptbtime1.ptb.de iburst nts ipv4
server nts.netnod.se iburst nts ipv4

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
ntsdumpdir /var/lib/chrony
leapseclist /usr/share/zoneinfo/leap-seconds.list
logdir /var/log/chrony

bindcmdaddress /var/run/chrony/chronyd.sock
cmdport 0
CHRONY

$ sudo systemctl restart chronyd
```

### Step 3: Configure Firefox DoH

In Firefox `about:config` (or via `user-overrides.js` — see [Doc 16](16-firefox-browser.md)):

```
network.trr.mode = 3
network.trr.uri = https://dns.quad9.net/dns-query
network.trr.bootstrapAddr = 9.9.9.9
network.dns.disableIPv6 = true
```

---

## ✅ Complete Verification

```bash
# 1. DNSSEC mode:
$ grep "DNSSEC" /etc/systemd/resolved.conf
# Expected: DNSSEC=allow-downgrade

# 2. LLMNR and mDNS disabled:
$ resolvectl status | head -10
# Expected: -LLMNR -mDNS

# 3. DNS routing (VPN users — only VPN interface has DNS):
$ resolvectl status
# Expected: Only VPN interface shows DNS server; physical NIC shows "Scopes: none"

# 4. Chrony NTS sources active:
$ chronyc -N sources
# Expected: 4 NTS servers, at least one marked ^* (selected) or ^+ (usable)

# 5. NTS authentication verified:
$ chronyc -N authdata
# Expected: All servers show "NTS" in Mode column

# 6. No UDP 323 (chrony command port):
$ sudo ss -ulnp | grep 323
# Expected: No output

# 7. DNS listener on loopback only:
$ ss -ulnp | grep ":53 "
# Expected: Only 127.0.0.53 and 127.0.0.54

# 8. DNS leak test (in browser):
# Visit https://www.dnsleaktest.com
# Expected: Shows VPN DNS, NOT your ISP's DNS
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| `DNSSEC=allow-downgrade` | Vulnerable to protocol-level downgrade | Compatible with VPN DNS resolvers |
| `DNSSEC=yes` NOT used | No client-side DNSSEC enforcement | Avoids breaking DNS with VPN + misconfigured domains |
| Firefox DoH Mode 3 | System DNS resolver bypassed for browser | Independent DNSSEC validation via Quad9 |
| LLMNR/mDNS disabled | No automatic LAN service discovery via DNS | No LAN name resolution information leaks |
| NTS-only (no pool.ntp.org) | Fewer NTP servers available | No unauthenticated time source in the mix |
| `cmdport 0` | Cannot use `chronyc` remotely | No network exposure for chrony management |

---

## Notes

- **DNSSEC + VPN = `allow-downgrade` is mandatory.** `DNSSEC=yes` breaks ALL DNS when using a VPN — this is not a bug, it's DNS architecture (RFC 4035). The VPN resolver validates DNSSEC server-side and strips RRSIG records from responses.
- **This applies to ALL VPN providers**, not just one specific provider. We verified this by querying VPN DNS resolvers directly with `dig +dnssec` — no RRSIG records returned.
- **Firefox DoH is your real DNSSEC defense.** Quad9 validates DNSSEC server-side. With TRR Mode 3 (DoH-only, no fallback), all browser DNS is DNSSEC-protected regardless of the systemd-resolved setting.
- **`chronyc -N authdata`** shows NTS authentication status per server. All servers should show "NTS" mode. If any show "NTP" instead, their NTS handshake failed.
- **Why `ipv4` on 3 servers?** With IPv6 disabled on the system, DNS resolution for NTS server hostnames must use IPv4. Cloudflare's anycast handles this automatically; the others need the explicit flag.
- **`bindcmdaddress 127.0.0.1` is NOT sufficient** for IPv6 disable — chrony opens an IPv6 command socket separately. `cmdport 0` is the complete fix.

---

*Previous: [10 — PAM & Login Security](10-pam-login-security.md)*
*Next: [12 — SELinux & auditd](12-selinux-auditd.md) — SELinux enforcing, 38 audit rules, immutable mode*
