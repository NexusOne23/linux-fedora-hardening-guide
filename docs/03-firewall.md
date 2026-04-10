# 🔥 Firewall

> firewalld with nftables backend: DROP-default policy, custom zones, outbound filtering, and IPv6 hardening.
> Applies to: Fedora Workstation 43+ | All hardware
> No reboot required — changes apply via `firewall-cmd --reload`.

---

## Overview

Fedora uses `firewalld` as the firewall management layer with `nftables` as the kernel backend. This document configures:

- **Default zone `drop`** — all unsolicited traffic silently dropped
- **Custom zone `strict-wan`** — dedicated zone for your physical network interface with no services, no forwarding
- **Policy `allow-host-ipv6`** — hardened ICMPv6 handling (Fedora default with dangerous types removed)
- **Policy `block-lan-out`** — blocks outbound LAN traffic, broadcasts, multicast, and known noisy services
- **Inactive zone cleanup** — removes services from unused Fedora default zones
- **VM policies** (conditional) — forces VM traffic through VPN

### Architecture

```
                        ┌─────────────────────────────────────┐
Internet ──► <YOUR-INTERFACE> ──►│ strict-wan (DROP)               │
                                │   ├─ allow-host-ipv6 (prio -15000) │
                                │   └─ block-lan-out   (prio  100)   │
                                └────────────────────────────────────┘
                                          │
VPN Tunnel ──► <YOUR-VPN-INTERFACE> ──► drop (DROP, default zone)
```

> **Related:** ARP-level filtering is in [Doc 04](04-nftables-arp-hardening.md). VPN killswitch (additional nftables tables) is in [Doc 06](06-vpn-killswitch.md). LAN isolation is in [Doc 05](05-lan-isolation.md).

---

## 1. Core Configuration: `firewalld.conf`

### What

The main firewalld configuration file that controls backend, defaults, and reload behavior.

### Why

Fedora's default firewalld configuration uses the `FedoraWorkstation` zone with SSH and mDNS open — too permissive for a hardened system. The reload policy defaults to `ACCEPT`, meaning during a firewall reload your system is completely unprotected for a fraction of a second.

### File: `/etc/firewalld/firewalld.conf`

```ini
DefaultZone=drop
CleanupOnExit=yes
CleanupModulesOnExit=no
IPv6_rpfilter=loose
IndividualCalls=no
LogDenied=all
FirewallBackend=nftables
FlushAllOnReload=yes
ReloadPolicy=INPUT:DROP,FORWARD:DROP,OUTPUT:DROP
RFC3964_IPv4=yes
StrictForwardPorts=no
NftablesFlowtable=off
NftablesCounters=no
NftablesTableOwner=yes
```

### Key Parameters

| Parameter | Value | Why |
|-----------|-------|-----|
| `DefaultZone` | `drop` | Unknown interfaces get the most restrictive zone — no accidental exposure |
| `LogDenied` | `all` | Log all dropped packets (INPUT, FORWARD, unicast, multicast) for auditing |
| `FirewallBackend` | `nftables` | Modern kernel packet filter (replaces legacy iptables) |
| `ReloadPolicy` | `INPUT:DROP,FORWARD:DROP,OUTPUT:DROP` | During reload, ALL traffic is dropped — no unprotected window |
| `RFC3964_IPv4` | `yes` | Blocks IPv4-in-IPv6 tunnel attacks (6to4/6rd encapsulation) |
| `NftablesTableOwner` | `yes` | Only firewalld can modify its nftables table — prevents tampering |
| `FlushAllOnReload` | `yes` | Clean slate on reload — no stale rules accumulate |
| `CleanupModulesOnExit` | `no` | Netfilter modules stay loaded on exit — prevents brief gap in protection |

### Decision

```
IPv6_rpfilter
├── Set loose if:  You use a WireGuard-based VPN (asymmetric routing)
├── Set strict if: No VPN — maximum anti-spoofing
└── Note:          Must match your sysctl rp_filter setting (Doc 02, Section 5)

ReloadPolicy
├── Set DROP/DROP/DROP:  Always — the default ACCEPT is a security gap
└── What breaks:         Brief connectivity interruption during firewall reload (milliseconds)
```

---

## 2. Zone: `drop` (Default)

### What

The default zone for all interfaces that are not explicitly assigned elsewhere.

### Why

Every new interface (USB tethering, VM bridges, unknown adapters) automatically lands in the most restrictive zone. No services, no ports, no forwarding. Nothing gets through unless explicitly allowed.

### Configuration

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone target="DROP">
  <short>Drop</short>
  <description>Unsolicited incoming network packets are dropped.</description>
  <forward/>
</zone>
```

| Property | Value | Why |
|----------|-------|-----|
| Target | `DROP` | Silently discard everything not explicitly allowed |
| Services | None | No services exposed |
| Ports | None | No ports open |
| Forward | Yes (`<forward/>`) | Required for VPN forwarding — VPN interfaces land in this zone |

### Decision

```
drop zone forward
├── Enable if:   You use a VPN (VPN interfaces need packet forwarding)
├── Disable if:  No VPN and no VMs — remove <forward/> tag
└── Note:        Forward only applies between interfaces in the same zone
```

> **Why `drop` instead of `reject`?** `DROP` gives no response — the attacker learns nothing. `REJECT` sends back an ICMP unreachable, confirming your existence and giving timing information.

---

## 3. Zone: `strict-wan` (Physical Interface)

### What

A custom zone for your physical network interface (Ethernet or Wi-Fi) with maximum restriction and no forwarding.

### Why

Your physical interface is your system's only entry point from the network. It needs its own zone (separate from `drop`) because:

1. **No forwarding** — `drop` zone has forwarding enabled for VPN; your physical NIC should never forward packets
2. **Policy binding** — the `block-lan-out` policy is bound to `strict-wan`, not to `drop`, keeping outbound filtering separate from VPN zone behavior
3. **Explicit assignment** — your NIC is assigned by name, not by "everything else" logic

### Configuration

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone target="DROP">
</zone>
```

| Property | Value |
|----------|-------|
| Target | `DROP` |
| Services | None |
| Ports | None |
| Forward | **No** (no `<forward/>` tag) |

### Decision

```
Separate zone for physical interface
├── Create if:     You use a VPN (need different forwarding behavior per zone)
├── Skip if:       No VPN — you can assign your NIC directly to the drop zone
└── What it adds:  Forwarding control + separate policy binding point
```

---

## 4. Inactive Zone Cleanup

### What

Remove all services from Fedora's pre-installed zones that you don't use.

### Why

Fedora ships with zones like `FedoraWorkstation` (SSH + mDNS + samba-client + DHCPv6-client open) and `home` (similar). If you accidentally assign an interface to one of these zones, those services are immediately exposed. Removing all services makes them harmless.

### Decision

```
Zone cleanup strategy
├── Remove services:  Yes — from all unused zones
├── Delete zones:     No — Fedora may recreate them during updates
└── Result:           Even accidental zone assignment exposes nothing
```

### Zones to Clean

| Zone | Services to remove |
|------|-------------------|
| `FedoraWorkstation` | ssh, mdns, samba-client, dhcpv6-client |
| `FedoraServer` | ssh, cockpit, dhcpv6-client |
| `home` / `work` / `internal` | ssh, mdns, samba-client, dhcpv6-client |
| `public` / `dmz` / `external` | ssh (and any others present) |

> **Leave `libvirt` and `nm-shared` alone** — they are managed by their respective daemons and only active when those services create interfaces.

---

## 5. Policy: `allow-host-ipv6` (Hardened)

### What

A firewalld policy that controls which ICMPv6 types reach your system. Fedora ships this with 8 types — we remove 2 dangerous ones.

### Why

IPv6 depends on ICMPv6 for basic functionality (neighbor discovery, multicast). But two ICMPv6 types are attack vectors:

- **`redirect`**: An attacker sends an ICMPv6 redirect telling your system to route traffic through them (MITM)
- **`router-advertisement`**: A rogue device advertises itself as your IPv6 router, hijacking your default gateway

These must be blocked. The remaining 6 types are safe and required for minimal IPv6 operation.

### Configuration

```xml
<?xml version="1.0" encoding="utf-8"?>
<policy priority="-15000" target="CONTINUE">
  <short>Allow host IPv6</short>
  <ingress-zone name="ANY"/>
  <egress-zone name="HOST"/>
  <!-- 6 safe ICMPv6 types retained -->
  <rule family="ipv6"><icmp-type name="mld-listener-done"/><accept/></rule>
  <rule family="ipv6"><icmp-type name="mld-listener-query"/><accept/></rule>
  <rule family="ipv6"><icmp-type name="mld-listener-report"/><accept/></rule>
  <rule family="ipv6"><icmp-type name="mld2-listener-report"/><accept/></rule>
  <rule family="ipv6"><icmp-type name="neighbour-advertisement"/><accept/></rule>
  <rule family="ipv6"><icmp-type name="neighbour-solicitation"/><accept/></rule>
  <!-- REMOVED: redirect, router-advertisement -->
</policy>
```

| Property | Value |
|----------|-------|
| Priority | `-15000` (evaluated before zone rules) |
| Target | `CONTINUE` (non-matching packets pass to zone rules) |
| Direction | ANY → HOST (inbound from all zones to the host) |

### Allowed ICMPv6 Types

| Type | Purpose | Why allowed |
|------|---------|-------------|
| `mld-listener-done` | MLD Leave Group | IPv6 multicast basic function |
| `mld-listener-query` | MLD Query | IPv6 multicast basic function |
| `mld-listener-report` | MLDv1 Report | IPv6 multicast basic function |
| `mld2-listener-report` | MLDv2 Report | IPv6 multicast basic function |
| `neighbour-advertisement` | NDP Neighbor Advertisement | IPv6 address resolution (like ARP) |
| `neighbour-solicitation` | NDP Neighbor Solicitation | IPv6 address resolution (like ARP) |

### Removed ICMPv6 Types

| Type | Why removed |
|------|-------------|
| `redirect` | Allows routing manipulation — attacker could redirect your traffic |
| `router-advertisement` | Allows rogue RA attacks — attacker could become your default gateway |

> **Critical:** This policy has priority `-15000` — it is evaluated **before** zone rules. Without removing `redirect` and `router-advertisement`, these attacks would bypass your DROP zones entirely.

---

## 6. Policy: `block-lan-out` (Outbound Filtering)

### What

A policy that blocks outbound traffic to LAN addresses, multicast, broadcast, and known noisy services.

### Why

A hardened desktop should never initiate traffic to the local network beyond what's needed for its gateway. Without this policy:

- Compromised software could scan or attack LAN devices
- wsdd (Web Services Discovery — a gvfs dependency) broadcasts to UDP 3702
- Multicast/broadcast traffic reveals your presence on the network

### Configuration

```xml
<?xml version="1.0" encoding="utf-8"?>
<policy priority="100" target="CONTINUE">
  <ingress-zone name="HOST"/>
  <egress-zone name="strict-wan"/>
  <rule family="ipv4"><port port="3702" protocol="udp"/><drop/></rule>
  <rule family="ipv4"><destination address="10.0.0.0/8"/><drop/></rule>
  <rule family="ipv4"><destination address="172.16.0.0/12"/><drop/></rule>
  <rule family="ipv4"><destination address="192.168.0.0/16"/><drop/></rule>
  <rule family="ipv4"><destination address="224.0.0.0/4"/><drop/></rule>
  <rule family="ipv4"><destination address="255.255.255.255/32"/><drop/></rule>
</policy>
```

| Property | Value |
|----------|-------|
| Priority | `100` (evaluated after zone rules) |
| Target | `CONTINUE` |
| Direction | HOST → strict-wan (outbound from host via your physical NIC) |

### Drop Rules

| Rule | What it blocks | Why |
|------|---------------|-----|
| UDP 3702 | wsdd (Web Services Discovery) | gvfs dependency — cannot be uninstalled, must be firewalled |
| `10.0.0.0/8` | RFC 1918 Class A | No outbound LAN traffic to private networks |
| `172.16.0.0/12` | RFC 1918 Class B | No outbound LAN traffic to private networks |
| `192.168.0.0/16` | RFC 1918 Class C | No outbound LAN traffic to private networks |
| `224.0.0.0/4` | Multicast | No IGMP/multicast into the LAN |
| `255.255.255.255/32` | Broadcast | No broadcast into the LAN |

### Decision

```
block-lan-out
├── Use if:       Always recommended — prevents LAN leakage from compromised software
├── Adjust if:    You need to reach LAN devices (printers, NAS, etc.)
│   └── Add specific allow rules BEFORE the RFC1918 drop rules
├── wsdd rule:    Always block — wsdd serves no purpose on a hardened desktop
└── What breaks:  Cannot reach any device on your local network (this is intentional)
```

> **Important:** `block-lan-out` is a **policy**, not a zone. `firewall-cmd --zone=block-lan-out` returns `INVALID_ZONE` — this is correct. Query it with `firewall-cmd --policy=block-lan-out`.

### If you need LAN access

If you need to reach specific LAN devices (e.g., a printer at `192.168.1.100`), add a specific allow rule with a negative priority so it is evaluated before the drop rules:

```bash
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule priority="-1" family="ipv4" destination address="192.168.1.100/32" accept'
```

---

## 7. VM Policies (Conditional — libvirt users only)

> **Skip this section** if you don't use virtual machines (libvirt/QEMU/KVM).

### What

Firewall policies that force VM traffic through the VPN tunnel and prevent direct LAN access from VMs.

### Why

Without these policies, a VM could bypass your VPN and access the LAN directly — leaking your real IP or attacking local devices.

### Policies

| Policy | Direction | Target | Purpose |
|--------|----------|--------|---------|
| `block-vm-direct` | libvirt → strict-wan | **DROP** | VMs cannot reach LAN directly |
| `vm-vpn-only` | libvirt → drop | ACCEPT | VM traffic routes through VPN tunnel |

### Decision

```
VM network policies
├── Create if:     You use libvirt/QEMU VMs AND a VPN
├── Skip if:       No VMs — these policies have no effect without libvirt
├── Skip if:       No VPN — vm-vpn-only has no VPN interface to route through
└── What it does:  All VM internet traffic must go through the VPN tunnel
```

> Fedora's default libvirt policies (`libvirt-routed-in`, `libvirt-routed-out`, `libvirt-to-host`) are created automatically when libvirt is installed — no manual creation needed.

---

## 8. Additional nftables Tables

Outside of firewalld, additional nftables tables provide specialized filtering:

| Table | Family | Purpose | Documented in |
|-------|--------|---------|---------------|
| ARP hardening | `arp` | Layer 2 ARP filtering (gateway MAC only) | [Doc 04](04-nftables-arp-hardening.md) |
| VPN killswitch | `ip` | Only VPN server IP allowed on physical NIC | [Doc 06](06-vpn-killswitch.md) |
| IPv6 killswitch6 | `ip6` | All IPv6 dropped on physical NIC (user-managed) | [Doc 06](06-vpn-killswitch.md) |

> These tables are managed independently of firewalld and are documented in their respective docs.

---

## 🔧 Applying Changes

### Step 1: Configure firewalld.conf

```bash
$ sudo sed -i 's/^DefaultZone=.*/DefaultZone=drop/' /etc/firewalld/firewalld.conf
$ sudo sed -i 's/^LogDenied=.*/LogDenied=all/' /etc/firewalld/firewalld.conf
$ sudo sed -i 's/^ReloadPolicy=.*/ReloadPolicy=INPUT:DROP,FORWARD:DROP,OUTPUT:DROP/' /etc/firewalld/firewalld.conf
```

### Step 2: Create `strict-wan` zone and assign your interface

```bash
$ sudo firewall-cmd --permanent --new-zone=strict-wan
$ sudo firewall-cmd --permanent --zone=strict-wan --set-target=DROP
$ sudo firewall-cmd --permanent --zone=strict-wan --add-interface=<YOUR-INTERFACE>
```

### Step 3: Clean up inactive zones

```bash
$ for zone in FedoraServer FedoraWorkstation dmz external home internal public work; do
    for svc in $(sudo firewall-cmd --permanent --zone=$zone --list-services); do
        sudo firewall-cmd --permanent --zone=$zone --remove-service=$svc
    done
done
```

### Step 4: Harden `allow-host-ipv6` policy

```bash
# Remove dangerous ICMPv6 types (suppress errors if already absent):
$ sudo firewall-cmd --permanent --policy=allow-host-ipv6 \
    --remove-rich-rule='rule family="ipv6" icmp-type name="redirect" accept' 2>/dev/null
$ sudo firewall-cmd --permanent --policy=allow-host-ipv6 \
    --remove-rich-rule='rule family="ipv6" icmp-type name="router-advertisement" accept' 2>/dev/null
```

### Step 5: Create `block-lan-out` policy

```bash
$ sudo firewall-cmd --permanent --new-policy=block-lan-out
$ sudo firewall-cmd --permanent --policy=block-lan-out --set-priority=100
$ sudo firewall-cmd --permanent --policy=block-lan-out --set-target=CONTINUE
$ sudo firewall-cmd --permanent --policy=block-lan-out --add-ingress-zone=HOST
$ sudo firewall-cmd --permanent --policy=block-lan-out --add-egress-zone=strict-wan

# Add drop rules:
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" port port="3702" protocol="udp" drop'
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" destination address="10.0.0.0/8" drop'
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" destination address="172.16.0.0/12" drop'
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" destination address="192.168.0.0/16" drop'
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" destination address="224.0.0.0/4" drop'
$ sudo firewall-cmd --permanent --policy=block-lan-out \
    --add-rich-rule='rule family="ipv4" destination address="255.255.255.255/32" drop'
```

### Step 6 (Optional): VM policies

Only if you use libvirt/QEMU with a VPN:

```bash
# Block VMs from reaching LAN directly:
$ sudo firewall-cmd --permanent --new-policy=block-vm-direct
$ sudo firewall-cmd --permanent --policy=block-vm-direct --set-priority=-1
$ sudo firewall-cmd --permanent --policy=block-vm-direct --set-target=DROP
$ sudo firewall-cmd --permanent --policy=block-vm-direct --add-ingress-zone=libvirt
$ sudo firewall-cmd --permanent --policy=block-vm-direct --add-egress-zone=strict-wan

# Route VM traffic through VPN:
$ sudo firewall-cmd --permanent --new-policy=vm-vpn-only
$ sudo firewall-cmd --permanent --policy=vm-vpn-only --set-priority=-1
$ sudo firewall-cmd --permanent --policy=vm-vpn-only --set-target=ACCEPT
$ sudo firewall-cmd --permanent --policy=vm-vpn-only --add-ingress-zone=libvirt
$ sudo firewall-cmd --permanent --policy=vm-vpn-only --add-egress-zone=drop
```

### Step 7: Reload

```bash
$ sudo firewall-cmd --reload
```

---

## ✅ Complete Verification

```bash
# 1. Default zone:
$ firewall-cmd --get-default-zone
# Expected: drop

# 2. Active zones:
$ firewall-cmd --get-active-zones
# Expected: strict-wan with your interface listed

# 3. strict-wan has no services:
$ sudo firewall-cmd --zone=strict-wan --list-all
# Expected: target: DROP, services: (empty), ports: (empty)

# 4. block-lan-out policy active:
$ sudo firewall-cmd --policy=block-lan-out --list-all
# Expected: 6 rich rules (wsdd, 3× RFC1918, multicast, broadcast)

# 5. allow-host-ipv6 hardened:
$ sudo firewall-cmd --policy=allow-host-ipv6 --list-all
# Expected: 6 rich rules (4× MLD + 2× NDP)
# Must NOT contain: redirect, router-advertisement

# 6. Inactive zones cleaned:
$ for zone in FedoraServer FedoraWorkstation dmz external home internal public work; do
    svcs=$(sudo firewall-cmd --permanent --zone=$zone --list-services)
    [ -z "$svcs" ] && echo "OK: $zone clean" || echo "WARNING: $zone has services: $svcs"
done
# Expected: All OK

# 7. LogDenied active:
$ grep "^LogDenied=" /etc/firewalld/firewalld.conf
# Expected: LogDenied=all

# 8. nftables rules for block-lan-out:
$ sudo nft list chain inet firewalld filter_OUT_policy_block-lan-out_deny
# Expected: 6 rules visible

# 9. No TCP services listening externally:
$ sudo ss -tlnp | grep -v "127\.\|::1"
# Expected: No listeners on 0.0.0.0 or ::
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| `DefaultZone=drop` | New interfaces have no connectivity until assigned | Zero accidental exposure |
| `ReloadPolicy=DROP` | Brief (~ms) connectivity interruption during reload | No unprotected window |
| `block-lan-out` RFC1918 drop | Cannot reach LAN devices (printers, NAS, etc.) | Compromised software can't attack LAN |
| `allow-host-ipv6` hardened | No IPv6 redirect or RA processing | Prevents RA/redirect MITM attacks |
| `strict-wan` no forwarding | Cannot route packets through your NIC | Prevents your system from being used as a router |
| `LogDenied=all` | Increased journal size from dropped packet logs | Full visibility into blocked traffic |

---

## Notes

- **`block-lan-out` is a policy, not a zone.** `firewall-cmd --zone=block-lan-out` returns `INVALID_ZONE` — this is correct behavior, not an error. Use `--policy=block-lan-out`.
- **The `external` zone** retains `masquerade: yes` (Fedora default). This is harmless — no interface is assigned to it.
- **Three protection layers** for outbound LAN traffic: (1) `block-lan-out` policy, (2) VPN killswitch nftables table ([Doc 06](06-vpn-killswitch.md)), (3) VPN policy routing. Depth matters.
- **Packet flow for inbound**: packet → established/related ACCEPT → loopback ACCEPT → invalid LOG+DROP → allow-host-ipv6 ICMPv6 → zone policy (DROP).
- **Packet flow for outbound**: packet → established/related ACCEPT → loopback ACCEPT → RFC3964 REJECT → zone policy → block-lan-out rules.

---

*Previous: [02 — Sysctl Hardening](02-sysctl-hardening.md)*
*Next: [04 — ARP Hardening](04-nftables-arp-hardening.md) — Layer 2 ARP filtering with nftables*
