# 🔇 LAN Isolation

> Network invisibility: MAC randomization, LLDP off, multicast/broadcast suppression, hostname protection.
> Applies to: Fedora Workstation 43+ | All hardware
> Goal: Your machine is invisible and unreachable on the local network.

---

## Overview

A hardened desktop should be **invisible** on its local network. No device should be able to discover, probe, or communicate with your machine — only the gateway (router) for internet access.

This document coordinates protections across Layers 2–7:

| Layer | Protection | Configured in |
|-------|-----------|---------------|
| L2 (ARP) | Static ARP, nftables ARP filter, sysctl | [Doc 04](04-nftables-arp-hardening.md) |
| L2 (MAC) | MAC randomization (NM `stable` mode) | **This document** |
| L2 (LLDP) | LLDP disabled | **This document** |
| L3 (IP) | No broadcast/multicast, ICMP ignored | [Doc 02](02-sysctl-hardening.md) |
| L3 (IPv6) | Disabled on physical interface | [Doc 07](07-ipv6-hardening.md) |
| L3–L4 | block-lan-out policy (RFC1918/multicast/broadcast DROP) | [Doc 03](03-firewall.md) |
| L3–L4 | strict-wan zone (DROP, no services) | [Doc 03](03-firewall.md) |
| L4 | VPN killswitch nftables (only VPN server IP allowed) | [Doc 06](06-vpn-killswitch.md) |
| L7 | IGMP reports suppressed, wsdd blocked | **This document** |
| Infra | libvirt network sockets masked | **This document** |

> This document focuses on the items marked "This document." The others are cross-referenced — see their respective docs for configuration details.

---

## 1. MAC Randomization

### What

Replace your network interface's real (hardware) MAC address with a deterministic randomized one.

### Why

Your real MAC address contains the manufacturer's OUI (Organizationally Unique Identifier) — the first 3 bytes identify the NIC vendor. This allows:
- **Device fingerprinting**: An observer can determine your hardware manufacturer
- **Persistent tracking**: The same MAC always identifies the same physical machine
- **Asset correlation**: The MAC can be correlated across networks if you move between them

With `stable` MAC randomization, NetworkManager generates a deterministic MAC per connection profile. It's consistent (your router always sees the same MAC), but it's not your real hardware MAC.

### How to check

```bash
# Current MAC vs real hardware MAC:
$ ip link show <YOUR-INTERFACE>
# Look for: link/ether XX:XX:XX:XX:XX:XX ... permaddr YY:YY:YY:YY:YY:YY
# link/ether = active MAC (should be randomized)
# permaddr = real hardware MAC (never sent on the network)

# A randomized MAC has the locally-administered bit set (second hex digit is 2, 6, A, or E)
# Example: 06:xx:xx:xx:xx:xx — the "0" + "6" means locally administered
```

### Decision

```
MAC randomization mode
├── stable (recommended):
│   ├── Deterministic per connection — router always sees the same MAC
│   ├── Not your real hardware MAC — vendor OUI is hidden
│   └── DHCP leases remain consistent (no new lease on every connect)
│
├── random:
│   ├── New MAC on every connection — maximum privacy
│   ├── Router assigns new DHCP lease each time (IP may change)
│   └── Use if: You frequently connect to untrusted networks (laptops/Wi-Fi)
│
└── preserve (Fedora default):
    ├── Uses real hardware MAC — vendor OUI visible
    └── NOT recommended for hardened systems
```

### How

```bash
# Find your connection name:
$ nmcli con show
# Note the NAME of your wired/wireless connection

# Set stable MAC randomization:
$ sudo nmcli con mod "Wired connection 1" 802-3-ethernet.cloned-mac-address stable
```

> Replace `"Wired connection 1"` with your actual connection name. For Wi-Fi, use `802-11-wireless.cloned-mac-address` instead.

### Verify

```bash
$ nmcli -f 802-3-ethernet.cloned-mac-address con show "Wired connection 1"
# Expected: stable

$ ip link show <YOUR-INTERFACE>
# The "link/ether" MAC should differ from "permaddr" (the real hardware MAC)
```

---

## 2. LLDP Disabled

### What

Disable the Link Layer Discovery Protocol on your network connection.

### Why

NetworkManager listens for LLDP frames from switches and other network devices by default. While NM doesn't actively broadcast LLDP, the received data reveals your network topology (VLAN IDs, switch hostnames, port descriptions) to applications. On a hardened desktop, LLDP serves no purpose — disable the listener.

### How

```bash
$ sudo nmcli con mod "Wired connection 1" connection.lldp disable
```

### Verify

```bash
$ nmcli -f connection.lldp con show "Wired connection 1"
# Expected: disable
```

### What breaks

Nothing. LLDP is a passive discovery protocol used in managed enterprise networks. Desktop systems never need it.

---

## 3. IGMP/Multicast Suppression

### What

Prevent your machine from announcing its multicast group memberships via IGMP.

### Why

IGMP (Internet Group Management Protocol) reports tell your router which multicast groups your machine has joined. On a LAN, any device can capture these reports and learn that your machine exists and what services it's interested in. The sysctl parameter `igmp_link_local_mcast_reports = 0` ([Doc 02](02-sysctl-hardening.md), Section 9) suppresses these reports.

### Multicast groups that remain

Some multicast groups are kernel defaults or created by desktop dependencies (gvfs). They cannot be permanently removed, but they are **triple-blocked**:

| Group | Source | Why harmless |
|-------|--------|-------------|
| `224.0.0.1` (all-hosts) | Kernel default | Outbound blocked by `block-lan-out` (224.0.0.0/4 DROP) |
| `239.255.255.250` (SSDP) | gvfs/wsdd dependency | Outbound blocked by `block-lan-out` (multicast + wsdd UDP 3702) |
| IPv6 multicast (`ff02::1`, `ff01::1`) | Kernel default | IPv6 disabled on your interface |

### wsdd (Web Services Discovery)

wsdd broadcasts to UDP 3702 — it is a gvfs dependency and cannot be uninstalled without breaking the file manager. It is triple-blocked:

1. `block-lan-out`: drops UDP 3702 outbound
2. `block-lan-out`: drops all multicast (224.0.0.0/4) outbound
3. VPN killswitch: only VPN server IP is allowed on your physical NIC ([Doc 06](06-vpn-killswitch.md))

---

## 4. Broadcast Suppression

### What

Prevent your machine from sending or responding to broadcast traffic.

### Why

Broadcast traffic (destination `255.255.255.255`) reaches every device on the LAN. A hardened desktop should never broadcast — it reveals presence and activity.

### How it's blocked

| Layer | Mechanism | Configured in |
|-------|-----------|---------------|
| Outbound | `block-lan-out` drops `255.255.255.255/32` | [Doc 03](03-firewall.md) |
| Inbound | `strict-wan` zone (DROP target) | [Doc 03](03-firewall.md) |
| ICMP | `icmp_echo_ignore_broadcasts = 1` (sysctl) | [Doc 02](02-sysctl-hardening.md) |

> No additional configuration needed here — this section documents how broadcast is handled across the other configs.

---

## 5. ICMP Invisibility

### What

Your machine ignores all ICMP echo requests (pings).

### Why

ICMP echo is the most basic network discovery method. `nmap -sn` (ping sweep) and simple `ping` commands use it to find live hosts. With `icmp_echo_ignore_all = 1` ([Doc 02](02-sysctl-hardening.md), Section 6), your machine is invisible to ping sweeps.

> **This does NOT affect outbound connections.** You can still ping other hosts — only incoming pings are ignored.

---

## 6. libvirt Network Sockets (Conditional)

> **Skip this section** if you don't have libvirt/QEMU installed.

### What

Mask libvirt's network management sockets to prevent automatic activation of virtual network daemons.

### Why

libvirt installs three network management daemons (`virtnetworkd`, `virtnwfilterd`, `virtinterfaced`) that can create virtual network bridges on your system. Even if the services are `disabled`, systemd socket activation would start them on demand. Masking the sockets prevents this entirely.

### How

```bash
$ sudo systemctl mask \
    virtnetworkd.socket virtnetworkd-ro.socket virtnetworkd-admin.socket \
    virtnwfilterd.socket virtnwfilterd-ro.socket virtnwfilterd-admin.socket \
    virtinterfaced.socket virtinterfaced-ro.socket virtinterfaced-admin.socket
```

This masks **9 sockets** total (3 daemons × 3 sockets each: main, read-only, admin).

### Decision

```
libvirt network sockets
├── Mask if:     You have libvirt installed but don't need automatic virtual networking
├── Skip if:     No libvirt installed — these sockets don't exist
├── Undo if:     You need VM networking — unmask the sockets
└── What breaks: VMs cannot create virtual network bridges via socket activation
                 (manual bridge creation still possible)
```

### Verify

```bash
$ systemctl is-enabled virtnetworkd.socket virtnwfilterd.socket virtinterfaced.socket 2>/dev/null
# Expected: masked masked masked
```

---

## 7. Hostname Protection

### What

Ensure your hostname doesn't leak identifying information to the network.

### Why

Hostnames can leak via DHCP requests, mDNS announcements, LLDP frames, and other protocols. A hardened system should either use a generic hostname or prevent hostname transmission entirely.

### Decision

```
Hostname strategy
├── Static IP (recommended):
│   ├── No DHCP requests sent — hostname never transmitted
│   └── Transient hostname "fedora" is generic and non-identifying
│
├── DHCP with privacy:
│   ├── Set: dhcp-send-hostname=false in NM connection
│   └── This prevents hostname from being sent in DHCP requests
│
└── Custom hostname:
    ├── Avoid personal names, machine descriptions, or serial numbers
    └── Generic names like "fedora", "workstation", or "desktop" are safe
```

### Check your hostname

```bash
$ hostnamectl
# Static hostname should be "(unset)" or generic
# Transient hostname should be generic (e.g., "fedora")
```

### If using DHCP

```bash
# Prevent hostname from being sent in DHCP requests:
$ sudo nmcli con mod "Wired connection 1" ipv4.dhcp-send-hostname false
```

> With a static IP configuration ([Doc 23](23-networkmanager-hardening.md)), no DHCP requests are sent, making hostname leakage via DHCP impossible.

---

## 8. Complete Picture: LAN Visibility

### What an attacker on your LAN sees

| Discovery method | Result |
|-----------------|--------|
| `nmap -sn <YOUR-SUBNET>` (ping sweep) | Invisible — ICMP ignored, ARP only to gateway |
| `arp-scan -l` | No ARP reply — `arp_ignore=1`, nftables ARP filter |
| LLDP listener | No LLDP frames sent |
| SSDP/UPnP discovery | No response — wsdd blocked, multicast DROP |
| mDNS/Avahi | No response — Avahi masked, mDNS never started |
| IPv6 NDP | No response — IPv6 disabled on physical interface |
| Port scan (`nmap -sT/-sU`) | No open ports — `strict-wan` DROP, no services |
| ARP spoofing attempt | Static ARP + nftables ARP filter + sysctl triple defense |

### What your machine sends to the LAN

| Traffic type | Status |
|-------------|--------|
| ARP requests | **Blocked** — static entry + nftables output DROP |
| ARP replies | Only to gateway (nftables allows only `arp operation reply`) |
| ICMP | **Blocked** — `icmp_echo_ignore_all=1` |
| Broadcast (255.255.255.255) | **Blocked** — `block-lan-out` DROP |
| Multicast (224.0.0.0/4) | **Blocked** — `block-lan-out` DROP |
| IGMP reports | **Blocked** — `igmp_link_local_mcast_reports=0` |
| wsdd (UDP 3702) | **Blocked** — `block-lan-out` DROP |
| LLDP frames | **Disabled** — NM `connection.lldp=disable` |
| IPv6 (any) | **Disabled** — `disable_ipv6`, `autoconf=0`, no RA |
| LAN traffic (RFC 1918) | **Blocked** — `block-lan-out` + VPN killswitch |

**Only allowed traffic on your physical interface:** Encrypted tunnel to your VPN server (if using VPN) or regular internet traffic to your gateway.

---

## 🔧 Applying Changes

The configurations unique to this document:

### Step 1: MAC randomization and LLDP

```bash
# Find your connection name:
$ nmcli con show

# Apply settings (replace connection name):
$ sudo nmcli con mod "Wired connection 1" 802-3-ethernet.cloned-mac-address stable
$ sudo nmcli con mod "Wired connection 1" connection.lldp disable

# Reconnect to apply MAC change:
$ sudo nmcli con down "Wired connection 1" && sudo nmcli con up "Wired connection 1"
```

### Step 2: libvirt sockets (if applicable)

```bash
$ sudo systemctl mask \
    virtnetworkd.socket virtnetworkd-ro.socket virtnetworkd-admin.socket \
    virtnwfilterd.socket virtnwfilterd-ro.socket virtnwfilterd-admin.socket \
    virtinterfaced.socket virtinterfaced-ro.socket virtinterfaced-admin.socket
```

### Step 3: Hostname (if using DHCP)

```bash
$ sudo nmcli con mod "Wired connection 1" ipv4.dhcp-send-hostname false
```

> Other LAN isolation measures (sysctl, firewall, ARP) are configured in their respective docs — see the overview table above.

---

## ✅ Complete Verification

```bash
# 1. MAC randomized:
$ ip link show <YOUR-INTERFACE> | grep -E "link/ether|permaddr"
# Expected: link/ether differs from permaddr (locally-administered bit set)

# 2. LLDP disabled:
$ nmcli -f connection.lldp con show "Wired connection 1"
# Expected: disable

# 3. IGMP reports suppressed:
$ sysctl net.ipv4.igmp_link_local_mcast_reports
# Expected: 0

# 4. No external TCP listeners:
$ sudo ss -tlnp | grep -v "127\.\|::1"
# Expected: No listeners on 0.0.0.0

# 5. libvirt sockets masked (if applicable):
$ systemctl is-enabled virtnetworkd.socket virtnwfilterd.socket virtinterfaced.socket 2>/dev/null
# Expected: masked masked masked

# 6. Ping test from another LAN device:
# $ ping <YOUR-STATIC-IP>    → no response
# $ nmap -sn <YOUR-STATIC-IP> → "Host seems down"
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| `stable` MAC | Not maximum randomness (same MAC per connection) | Consistent DHCP lease, hidden vendor OUI |
| LLDP disabled | No network topology discovery | No information leaked to LAN |
| IGMP suppressed | Router doesn't know your multicast subscriptions | Invisible to multicast enumeration |
| libvirt sockets masked | No automatic VM networking | Prevents unintended bridge creation |
| Full LAN isolation | Cannot communicate with any LAN device | Maximum invisibility |

---

*Previous: [04 — ARP Hardening](04-nftables-arp-hardening.md)*
*Next: [06 — VPN Killswitch](06-vpn-killswitch.md) — Quadruple killswitch ensuring zero traffic leaks*
