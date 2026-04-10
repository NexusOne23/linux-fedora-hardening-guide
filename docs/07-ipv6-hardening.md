# 🌐 IPv6 Hardening

> Selective IPv6 disable with defense-in-depth: sysctl + NetworkManager + firewall coordination.
> Applies to: Fedora Workstation 43+ | All hardware
> No reboot required.

---

## Overview

If you don't use IPv6 for internet access (most home networks are IPv4-only), every IPv6 feature is pure attack surface. This document defines the **strategy** for disabling IPv6 and coordinates the implementation across three subsystems:

| Subsystem | What it controls | Configured in |
|-----------|-----------------|---------------|
| sysctl | Kernel-level IPv6 disable + RA/autoconf hardening | [Doc 02](02-sysctl-hardening.md), Section 10 |
| NetworkManager | Per-connection IPv6 method (disabled/auto/manual) | **This document** |
| firewalld | ICMPv6 type filtering (allow-host-ipv6 policy) | [Doc 03](03-firewall.md), Section 5 |

> **This document focuses on the strategy and NM configuration.** The sysctl parameters and firewall rules are implemented in their respective docs.

---

## 1. Disable Strategy

### The problem with `all.disable_ipv6 = 1`

The obvious approach — globally disabling IPv6 — breaks VPN killswitch implementations:

```
net.ipv6.conf.all.disable_ipv6 = 1   ← DO NOT SET THIS (with VPN)
```

Some VPN clients create a killswitch interface with IPv6 blackhole routes. `all.disable_ipv6 = 1` removes these routes and IPv6 addresses, breaking the killswitch's ability to catch and discard IPv6 traffic. IPv6 packets could then leak uncontrolled.

### Selective disable (recommended for VPN users)

```
all.disable_ipv6     = 0    ← NOT globally disabled (VPN killswitch needs IPv6)
default.disable_ipv6 = 1    ← New interfaces: IPv6 off by default
<YOUR-INTERFACE>             ← Physical NIC: IPv6 disabled via NM + defense-in-depth sysctl
loopback                     ← IPv6 active (::1 needed internally)
VPN killswitch interface     ← IPv6 active (blackhole route for killswitch)
```

### Global disable (simpler, for non-VPN users)

If you don't use a VPN, you can safely disable IPv6 globally:

```
all.disable_ipv6     = 1    ← All interfaces: IPv6 off
default.disable_ipv6 = 1    ← New interfaces: IPv6 off
```

### Decision

```
IPv6 disable strategy
├── VPN with killswitch:
│   ├── Set: default.disable_ipv6 = 1 (NOT all.disable_ipv6!)
│   ├── Set: NM ipv6.method = disabled on physical connection
│   ├── Set: Defense-in-depth sysctl params on physical interface
│   └── Why: VPN killswitch interface needs IPv6 for blackhole routes
│
├── No VPN:
│   ├── Set: all.disable_ipv6 = 1
│   ├── Set: default.disable_ipv6 = 1
│   └── Simpler — no VPN killswitch to protect
│
└── IPv6 required (IPv6-only ISP):
    ├── Do NOT disable IPv6
    ├── Set: privacy extensions (use_tempaddr = 2)
    ├── Set: accept_redirects = 0, accept_source_route = 0
    └── Consider: DHCPv6 instead of SLAAC for address assignment
```

---

## 2. NetworkManager: IPv6 Disabled

### What

Disable IPv6 at the NetworkManager level for your physical connection, in addition to sysctl.

### Why

Two independent mechanisms disable IPv6 on your physical interface:
1. **sysctl** (`disable_ipv6 = 1` on interface, or `default.disable_ipv6 = 1` for new interfaces)
2. **NetworkManager** (`ipv6.method = disabled` on the connection profile)

If either mechanism fails (kernel bug, config reload race condition, NM profile reset), the other still prevents IPv6 from activating. This is defense-in-depth.

### How

```bash
$ sudo nmcli con mod "Wired connection 1" ipv6.method disabled
```

> Replace `"Wired connection 1"` with your connection name (`nmcli con show`).

### Verify

```bash
$ nmcli -f ipv6.method con show "Wired connection 1"
# Expected: disabled

$ ip -6 addr show dev <YOUR-INTERFACE>
# Expected: No output (no IPv6 addresses)
```

---

## 3. Defense-in-Depth: Per-Interface Hardening

### What

Even with IPv6 disabled, all RA/autoconf/DAD parameters are explicitly set to 0 on your physical interface.

### Why

If a kernel bug, a race condition during boot, or a faulty update re-enables IPv6 on your interface, these parameters prevent the most dangerous IPv6 attacks — **before** you notice and fix the issue:

| Parameter | Value | Attack it prevents |
|-----------|-------|--------------------|
| `autoconf` | `0` | SLAAC address generation from rogue RAs |
| `accept_ra_defrtr` | `0` | Accepting a rogue default router |
| `accept_ra_pinfo` | `0` | Accepting prefix information from rogue RAs |
| `accept_ra_rtr_pref` | `0` | Accepting router preference from rogue RAs |
| `dad_transmits` | `0` | DAD transmissions (reveals machine presence on network) |
| `router_solicitations` | `0` | Router solicitations (actively asks for IPv6 routers) |

These are configured in [Doc 02](02-sysctl-hardening.md), Section 10.

> **Is this redundant?** Yes, intentionally. With `disable_ipv6 = 1`, none of these parameters matter. But if `disable_ipv6` gets reset to `0`, they become your last line of defense against rogue RA attacks.

---

## 4. Global Scope: Redirects and Source Routing

Regardless of whether IPv6 is enabled or disabled per-interface, these parameters are set globally:

| Parameter | Scope | Value | Attack prevented |
|-----------|-------|-------|-----------------|
| `accept_redirects` | `all` + `default` | `0` | ICMPv6 redirect MITM attacks |
| `accept_source_route` | `all` + `default` | `0` | IPv6 source routing (attacker-controlled path) |

These apply to **all** interfaces — including VPN and killswitch interfaces. Configured in [Doc 02](02-sysctl-hardening.md), Section 10.

---

## 5. Firewall: ICMPv6 Hardening

The `allow-host-ipv6` firewalld policy (priority -15000) controls which ICMPv6 types reach your system. Two dangerous types have been removed:

| ICMPv6 Type | Status | Why |
|-------------|--------|-----|
| `redirect` | **Removed** | Routing manipulation — attacker redirects your traffic |
| `router-advertisement` | **Removed** | Rogue RA — attacker becomes your default gateway |
| MLD types (4) | Kept | IPv6 multicast basic function |
| NDP types (2) | Kept | IPv6 address resolution (like ARP) |

This is fully documented in [Doc 03](03-firewall.md), Section 5.

> **Why this matters even with IPv6 disabled:** The `allow-host-ipv6` policy has priority -15000 — it is evaluated **before** zone rules. If `redirect` or `router-advertisement` were still allowed and IPv6 somehow re-enabled, these attacks would bypass your DROP zones entirely.

---

## 6. IPv6 Audit: What Remains Active

After applying all hardening, the following IPv6 addresses and routes remain:

### Expected IPv6 addresses

| Interface | IPv6 | Why active |
|-----------|------|-----------|
| `lo` (loopback) | `::1/128` | Internal — required by many applications |
| VPN killswitch interface | ULA global + link-local | Killswitch blackhole route (VPN only) |
| Your physical NIC | **None** | IPv6 disabled |
| VPN tunnel interface | **None** | WireGuard IPv4-only tunnel |

### Expected IPv6 routes (VPN users)

All IPv6 routes point to the killswitch dummy interface → blackhole:

```bash
$ ip -6 route show
# All routes via killswitch interface — IPv6 traffic goes nowhere
```

### Verify

```bash
# No IPv6 on physical interface:
$ ip -6 addr show dev <YOUR-INTERFACE>
# Expected: No output

# all.disable_ipv6 NOT set (VPN users):
$ sysctl net.ipv6.conf.all.disable_ipv6
# Expected: 0

# OR all.disable_ipv6 IS set (non-VPN users):
$ sysctl net.ipv6.conf.all.disable_ipv6
# Expected: 1

# NM IPv6 disabled:
$ nmcli -f ipv6.method con show "Wired connection 1"
# Expected: disabled

# Defense-in-depth params:
$ sysctl net.ipv6.conf.<YOUR-INTERFACE>.autoconf \
  net.ipv6.conf.<YOUR-INTERFACE>.dad_transmits \
  net.ipv6.conf.<YOUR-INTERFACE>.router_solicitations
# Expected: 0 0 0
```

---

## 🔧 Applying Changes

### The only configuration unique to this document:

```bash
# Disable IPv6 in NetworkManager for your physical connection:
$ sudo nmcli con mod "Wired connection 1" ipv6.method disabled
```

> All other IPv6 hardening (sysctl parameters, firewall ICMPv6 rules) is configured in [Doc 02](02-sysctl-hardening.md) and [Doc 03](03-firewall.md).

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| Selective IPv6 disable (VPN) | More complex than global disable | VPN killswitch works correctly |
| Global IPv6 disable (no VPN) | IPv6-only services unreachable | Zero IPv6 attack surface |
| Defense-in-depth params | Redundant when disable_ipv6=1 | Protection if disable_ipv6 gets reset |
| ICMPv6 redirect/RA removed | Cannot use IPv6 RA for legitimate purposes | Blocks rogue RA and redirect attacks |

---

## Notes

- **IPv6-only ISPs:** If your ISP provides only IPv6, do NOT disable IPv6. Instead, harden it: privacy extensions, no redirects, no source routing, DHCPv6 preferred over SLAAC.
- **`accept_ra` vs individual RA parameters:** Setting `autoconf=0`, `accept_ra_defrtr=0`, etc. is more thorough than just `accept_ra=0`. The individual parameters prevent specific RA features even if `accept_ra` is re-enabled.
- **Loopback `::1`:** Must remain active. Many applications (including systemd-resolved) use `::1` for internal communication. Disabling IPv6 on loopback can cause subtle breakage.
- **VPN IPv6 blackhole:** The VPN killswitch's IPv6 blackhole route ensures that even if an application tries IPv6 DNS or connections, they route to the dummy interface and silently fail. This is intentional.

---

*Previous: [06 — VPN Killswitch](06-vpn-killswitch.md)*
*Next: [08 — Service Minimization](08-service-minimization.md) — 73 disabled components (53 system + 18 user + 2 D-Bus)*
