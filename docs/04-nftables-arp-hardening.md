# 🛡️ ARP Hardening

> Layer 2 protection: static ARP, nftables ARP filtering, and NetworkManager persistence.
> Applies to: Fedora Workstation 43+ | All hardware with Ethernet or Wi-Fi
> No reboot required — changes apply immediately.

---

## Overview

ARP (Address Resolution Protocol) operates at **Layer 2** — below IP, below firewalld, below everything you normally think of as a firewall. Without explicit ARP hardening, an attacker on your LAN can redirect all your traffic through their machine in seconds.

This document implements three defense layers:

1. **Static ARP entry** — your gateway's MAC is permanent and cannot be overwritten by ARP replies
2. **nftables `arp` table** — only ARP frames from your gateway's MAC are accepted, everything else is dropped
3. **sysctl ARP parameters** — already configured in [Doc 02](02-sysctl-hardening.md), Section 8

### Why Three Layers?

| Layer | Protects against | How |
|-------|-----------------|-----|
| Static ARP (PERMANENT) | ARP cache poisoning | Kernel ignores all ARP updates for gateway IP |
| nftables ARP filter | ARP spoofing from any MAC | Only gateway's real MAC passes the filter |
| sysctl (arp_ignore, drop_gratuitous_arp) | ARP probing, gratuitous ARP | Kernel-level response restriction |

> **Related:** Sysctl ARP parameters are in [Doc 02](02-sysctl-hardening.md). Firewall (Layer 3+) is in [Doc 03](03-firewall.md). LAN isolation is in [Doc 05](05-lan-isolation.md).

---

## 1. Why firewalld Is Not Enough

### What happens at Layer 2

firewalld operates on the `inet` family (IPv4/IPv6). ARP frames are processed **before** the IP stack and never reach firewalld rules. An attacker can:

- **ARP spoofing**: Send fake ARP replies that overwrite your gateway's MAC in the ARP table → all your traffic goes through the attacker (MITM)
- **ARP flooding**: Fill your ARP table with fake entries → denial of service
- **Gratuitous ARP**: Send unsolicited ARP replies that silently change MAC associations

All of this happens at Layer 2 — completely invisible to IP-based firewalls.

### How practical is this?

Tools like `arpspoof` or `ettercap` can execute ARP spoofing attacks on any LAN in **under 10 seconds**. Anyone on your local network (including compromised IoT devices) can do this. This is not a theoretical attack.

---

## 2. Static ARP Entry

### What

A permanent ARP entry that maps your gateway's IP to its real MAC address. The kernel will never update this entry, regardless of what ARP replies it receives.

### Why

A normal ARP entry (`REACHABLE` or `STALE`) can be overwritten by incoming ARP replies — this is exactly what ARP spoofing exploits. A `PERMANENT` entry is immutable — the kernel ignores all ARP updates for this IP.

With a static entry, the kernel also doesn't need to send ARP requests to resolve the gateway's MAC. This is why the nftables output chain can safely block ARP requests.

### How to find your gateway's IP and MAC

```bash
# Your gateway IP:
$ ip route | grep default
default via 192.168.1.1 dev enp3s0 proto static metric 100
#              ^^^^^^^^^^^^ this is your gateway IP

# Your gateway MAC:
$ ip neigh show dev <YOUR-INTERFACE> | grep <YOUR-GATEWAY-IP>
192.168.1.1 lladdr aa:bb:cc:dd:ee:ff REACHABLE
#                  ^^^^^^^^^^^^^^^^^^ this is your gateway MAC
```

> **Important:** Run these commands **on a trusted network** before applying the static entry. If an attacker is already spoofing, you would capture the wrong MAC.

### Verify after setting

```bash
$ ip neigh show dev <YOUR-INTERFACE>
# Expected: <YOUR-GATEWAY-IP> lladdr <YOUR-GATEWAY-MAC> PERMANENT
```

---

## 3. nftables ARP Filter

### What

An nftables table using the `arp` family that filters all ARP frames on your physical interface. Only frames from your gateway's MAC are accepted — everything else is silently dropped.

### Why

The static ARP entry prevents cache poisoning, but an attacker can still flood your system with ARP frames (resource consumption, potential kernel bugs). The nftables filter drops all ARP frames from unknown MACs before they reach the kernel's ARP processing.

### File: `/etc/nftables/arp-hardening.nft`

```nft
#!/usr/sbin/nft -f

# ARP Hardening: Only gateway MAC accepted, everything else DROP
destroy table arp arp_hardening

table arp arp_hardening {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        iifname "<YOUR-INTERFACE>" ether saddr <YOUR-GATEWAY-MAC> accept
    }

    chain output {
        type filter hook output priority filter; policy drop;
        oifname "lo" accept
        oifname "<YOUR-INTERFACE>" arp operation reply accept
    }
}
```

> Set permissions: `chmod 600 /etc/nftables/arp-hardening.nft`

### Rule Logic

**INPUT chain (received ARP frames):**

| Rule | Action | Why |
|------|--------|-----|
| `iifname "lo"` | ACCEPT | Loopback — internal, no network |
| `iifname "<YOUR-INTERFACE>" ether saddr <YOUR-GATEWAY-MAC>` | ACCEPT | Only your gateway's real MAC is trusted |
| Default policy | DROP | All other ARP frames silently discarded |

**OUTPUT chain (sent ARP frames):**

| Rule | Action | Why |
|------|--------|-----|
| `oifname "lo"` | ACCEPT | Loopback |
| `oifname "<YOUR-INTERFACE>" arp operation reply` | ACCEPT | ARP replies to gateway (when it asks) |
| Default policy | DROP | No ARP requests sent (static entry makes them unnecessary) |

### Decision

```
libvirt / VM bridge (virbr0)
├── Add rules if:    You use libvirt/QEMU VMs
│   ├── INPUT:  iifname "virbr0" accept
│   ├── OUTPUT: oifname "virbr0" accept
│   └── Place these BEFORE the interface-specific rules
├── Skip if:         No VMs — virbr0 doesn't exist
└── Why needed:      The VM bridge needs ARP for its internal DHCP/gateway function

Why "destroy table" instead of "flush"?
├── destroy: Deletes and recreates the table — clean slate, no stale rules
├── flush:   Only empties chains — doesn't remove chains or change structure
└── destroy: No error if table doesn't exist (unlike "delete table")
```

---

## 4. Persistence

ARP hardening must survive reboots, reconnections, and firewalld reloads. Two persistence mechanisms handle this:

### 4.1 NetworkManager Dispatcher Script

Triggers on interface up (boot, reconnect, resume from suspend).

#### File: `/etc/NetworkManager/dispatcher.d/90-arp-hardening`

```bash
#!/bin/bash
# ARP Hardening — Static ARP + nftables arp filter
# Triggered by NetworkManager on interface up

IFACE="$1"
ACTION="$2"

GATEWAY_IP="<YOUR-GATEWAY-IP>"
GATEWAY_MAC="<YOUR-GATEWAY-MAC>"
NFT_FILE="/etc/nftables/arp-hardening.nft"

apply_arp_hardening() {
    # Static ARP entry (prevents spoofing + eliminates ARP requests)
    ip neigh replace "$GATEWAY_IP" lladdr "$GATEWAY_MAC" dev <YOUR-INTERFACE> nud permanent

    # Load nftables ARP filter
    if [ -f "$NFT_FILE" ]; then
        nft -f "$NFT_FILE"
    fi
}

case "$ACTION" in
    up)
        if [ "$IFACE" = "<YOUR-INTERFACE>" ]; then
            apply_arp_hardening
        fi
        ;;
esac
```

> Set permissions: `chmod 700 /etc/NetworkManager/dispatcher.d/90-arp-hardening`

### 4.2 firewalld Reload Drop-In

#### Why needed

firewalld with `NftablesTableOwner=yes` deletes **all non-firewalld nftables tables** on restart/reload. Without this drop-in, a simple `firewall-cmd --reload` would wipe your ARP hardening.

#### File: `/etc/systemd/system/firewalld.service.d/arp-hardening-firewalld-reload.conf`

```ini
[Service]
ExecStartPost=/usr/sbin/nft -f /etc/nftables/arp-hardening.nft
```

After creating this file:

```bash
$ sudo systemctl daemon-reload
```

---

## 5. Protection Flow

```
ARP frame received on <YOUR-INTERFACE>
  │
  ├─ nftables arp input: source MAC ≠ <YOUR-GATEWAY-MAC>? → DROP
  │
  ├─ sysctl drop_gratuitous_arp=1: gratuitous ARP? → DROP
  │
  ├─ sysctl arp_ignore=1: target IP not on this interface? → no response
  │
  └─ Kernel ARP table: <YOUR-GATEWAY-IP> = PERMANENT → update ignored
```

All four checks must pass for an ARP frame to be processed. An attacker must spoof the exact gateway MAC **and** pass all sysctl filters **and** somehow override a PERMANENT ARP entry. This is effectively impossible.

---

## 🔧 Applying Changes

### Step 1: Find your gateway IP and MAC

```bash
$ ip route | grep default
# Note: the IP after "via" is your gateway IP

$ ip neigh show dev <YOUR-INTERFACE>
# Note: the MAC (lladdr) for your gateway IP
```

### Step 2: Create the nftables ARP rule file

```bash
$ sudo mkdir -p /etc/nftables
$ sudo tee /etc/nftables/arp-hardening.nft << 'NFT'
#!/usr/sbin/nft -f

destroy table arp arp_hardening

table arp arp_hardening {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        iifname "<YOUR-INTERFACE>" ether saddr <YOUR-GATEWAY-MAC> accept
    }

    chain output {
        type filter hook output priority filter; policy drop;
        oifname "lo" accept
        oifname "<YOUR-INTERFACE>" arp operation reply accept
    }
}
NFT
$ sudo chmod 600 /etc/nftables/arp-hardening.nft
```

> **Edit the file** to replace `<YOUR-INTERFACE>` and `<YOUR-GATEWAY-MAC>` with your actual values.

### Step 3: Create the NM dispatcher script

```bash
$ sudo tee /etc/NetworkManager/dispatcher.d/90-arp-hardening << 'DISPATCHER'
#!/bin/bash
IFACE="$1"
ACTION="$2"
GATEWAY_IP="<YOUR-GATEWAY-IP>"
GATEWAY_MAC="<YOUR-GATEWAY-MAC>"
NFT_FILE="/etc/nftables/arp-hardening.nft"

apply_arp_hardening() {
    ip neigh replace "$GATEWAY_IP" lladdr "$GATEWAY_MAC" dev <YOUR-INTERFACE> nud permanent
    if [ -f "$NFT_FILE" ]; then
        nft -f "$NFT_FILE"
    fi
}

case "$ACTION" in
    up)
        if [ "$IFACE" = "<YOUR-INTERFACE>" ]; then
            apply_arp_hardening
        fi
        ;;
esac
DISPATCHER
$ sudo chmod 700 /etc/NetworkManager/dispatcher.d/90-arp-hardening
```

> **Edit the file** to replace all `<YOUR-...>` placeholders.

### Step 4: Create the firewalld drop-in

```bash
$ sudo mkdir -p /etc/systemd/system/firewalld.service.d
$ sudo tee /etc/systemd/system/firewalld.service.d/arp-hardening-firewalld-reload.conf << 'DROPIN'
[Service]
ExecStartPost=/usr/sbin/nft -f /etc/nftables/arp-hardening.nft
DROPIN
$ sudo systemctl daemon-reload
```

### Step 5: Apply immediately

```bash
# Set static ARP entry:
$ sudo ip neigh replace <YOUR-GATEWAY-IP> lladdr <YOUR-GATEWAY-MAC> dev <YOUR-INTERFACE> nud permanent

# Load nftables ARP filter:
$ sudo nft -f /etc/nftables/arp-hardening.nft
```

---

## ✅ Complete Verification

```bash
# 1. Static ARP entry is PERMANENT:
$ ip neigh show dev <YOUR-INTERFACE>
# Expected: <YOUR-GATEWAY-IP> lladdr <YOUR-GATEWAY-MAC> PERMANENT

# 2. nftables ARP table loaded:
$ sudo nft list table arp arp_hardening
# Expected: input/output chains with policy drop and your rules

# 3. Dispatcher script exists and is executable:
$ ls -la /etc/NetworkManager/dispatcher.d/90-arp-hardening
# Expected: -rwx------ 1 root root

# 4. firewalld drop-in exists:
$ systemctl cat firewalld | grep ExecStartPost
# Expected: ExecStartPost=/usr/sbin/nft -f /etc/nftables/arp-hardening.nft

# 5. ARP filter rules active:
$ sudo nft list ruleset | grep -A5 "table arp"
# Expected: arp_hardening table with input/output chains, policy drop
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| Static ARP | Must update 3 files if router changes | Gateway MAC cannot be spoofed |
| nftables ARP DROP policy | No ARP from any device except gateway | Complete Layer 2 isolation |
| No ARP requests (output DROP) | Cannot discover new LAN devices via ARP | Minimal ARP footprint on network |
| Gateway-MAC-only filter | Cannot communicate with LAN devices (printers, NAS) | Maximum LAN isolation |

---

## Notes

- **Router replacement:** If you change your router, update three locations: (1) `arp-hardening.nft`, (2) `90-arp-hardening` dispatcher, (3) manually run `ip neigh replace` with the new MAC.
- **No LAN device access:** This configuration only allows ARP with your gateway. Direct LAN communication (printers, NAS, other computers) is blocked at Layer 2. This is intentional for maximum isolation.
- **VPN interfaces:** WireGuard tunnels (and similar Layer 3 VPNs) don't use ARP — they encapsulate at IP level. ARP rules only affect your physical interface.
- **`destroy table` is idempotent:** Can be executed repeatedly without errors, unlike `delete table` which fails if the table doesn't exist.
- **SELinux context:** The dispatcher script must have the `NetworkManager_dispatcher_script_t` context. Files created in `/etc/NetworkManager/dispatcher.d/` get this automatically.

---

*Previous: [03 — Firewall](03-firewall.md)*
*Next: [05 — LAN Isolation](05-lan-isolation.md) — MAC randomization, LLDP off, multicast suppression*
