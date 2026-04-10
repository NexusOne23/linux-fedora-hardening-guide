# 🔒 VPN Killswitch

> Quadruple killswitch ensuring zero traffic leaks: nftables IPv4/IPv6 + policy routing + firewall policy.
> Applies to: Fedora Workstation 43+ | WireGuard-based VPN providers
> Conditional: Skip this document if you don't use a VPN.

---

## Overview

A VPN killswitch ensures that if the VPN tunnel drops, **no traffic leaves your system unencrypted**. A single killswitch layer can fail — this guide implements four independent layers that must all fail simultaneously for a leak to occur.

### Four Killswitch Layers

| Layer | Mechanism | Managed by | What happens on VPN failure |
|-------|-----------|-----------|----------------------------|
| 1. nftables killswitch (IPv4) | Only VPN server IP allowed on physical NIC | VPN client (automatic) | All non-VPN IPv4 traffic dropped |
| 2. nftables killswitch6 (IPv6) | All IPv6 traffic dropped on physical NIC | User (manual) | No IPv6 traffic bypasses VPN |
| 3. Policy routing (fwmark) | Non-VPN traffic routed to tunnel interface | VPN client (automatic) | Traffic routes to dummy interface → blackholed |
| 4. block-lan-out policy | RFC1918 + multicast + broadcast DROP on output | User (manual, [Doc 03](03-firewall.md)) | No LAN traffic even if layers 1-3 fail |

> **Key insight:** Layers 1 and 3 are managed by your VPN client software. Layers 2 and 4 are configured by you. This document explains how all four work together and what you need to configure for compatibility.

---

## 1. VPN Interfaces

When a WireGuard-based VPN with killswitch is active, two additional interfaces appear:

### Tunnel Interface

| Property | Value |
|----------|-------|
| Type | WireGuard tunnel (Layer 3, no ARP) |
| IP | VPN-assigned tunnel IP (e.g., `10.x.x.x/32`) |
| MTU | 1420 (WireGuard standard) |
| Firewall zone | `drop` (no incoming connections) |
| Purpose | Encrypted tunnel — all internet traffic goes through this |

### Killswitch Interface

| Property | Value |
|----------|-------|
| Type | Virtual dummy interface |
| IPv4 | CGNAT range address (not internet-routable) |
| IPv6 | ULA address (used for IPv6 blackhole route) |
| Firewall zone | `drop` |
| Purpose | Default route trap — catches traffic when tunnel is down |

### Decision

```
VPN interfaces in firewall zones
├── Both interfaces MUST be in the drop zone (no incoming connections)
├── If DefaultZone=drop (Doc 03): automatic — VPN interfaces inherit drop
├── If DefaultZone is NOT drop: manually assign VPN interfaces to drop zone
└── Verify with: firewall-cmd --get-active-zones
```

### Why the killswitch interface has IPv6

Some VPN clients create an IPv6 blackhole route via the killswitch interface. If any IPv6 traffic is attempted, it routes to the dummy interface and is silently discarded. This is why `net.ipv6.conf.all.disable_ipv6` must NOT be set to `1` — it would break this blackhole route ([Doc 02](02-sysctl-hardening.md), Section 10).

---

## 2. Layer 1: nftables Killswitch Table

### What

An nftables table (managed by your VPN client) that restricts outbound traffic on your physical interface to **only** the VPN server's IP address.

### How it works

```nft
table ip killswitch {
    chain output {
        type filter hook output priority filter; policy accept;
        oifname "<YOUR-INTERFACE>" ip daddr <YOUR-SUBNET> drop
        oifname "<YOUR-INTERFACE>" ip daddr != <YOUR-VPN-SERVER-IP> drop
    }
}
```

| Rule | Effect |
|------|--------|
| LAN subnet → DROP | No direct LAN communication on physical NIC |
| Destination ≠ VPN server → DROP | Only VPN server IP can be reached via physical NIC |

### What this means

On your physical interface, the **only** allowed destination is your VPN server. DNS, web traffic, email — everything must go through the WireGuard tunnel. If the tunnel is down, nothing gets out.

> **You don't create this table manually.** Your VPN client creates and manages it when the killswitch is enabled. The VPN server IP updates automatically on server changes.

---

## 3. Layer 2: IPv6 Killswitch Table (User-Managed)

### What

An nftables table that drops **all** IPv6 traffic on your physical interface. This is the IPv6 counterpart to the IPv4 killswitch (Layer 1).

### Why

WireGuard creates a dual-stack kernel socket by default — visible as `[::]:PORT` in `ss -6 -ulnp`. This socket binds on IPv6 even when `disable_ipv6=1` is set on your interfaces. Since `net.ipv6.conf.all.disable_ipv6` cannot be set to `1` (it breaks the killswitch interface's IPv6 blackhole route), an explicit IPv6 firewall rule is needed.

Without this layer, an attacker on your LAN could theoretically reach the WireGuard socket via IPv6 if your physical interface somehow acquires an IPv6 address (rogue Router Advertisement).

### Config

```nft
table ip6 killswitch6 {
    chain input {
        type filter hook input priority -1; policy accept;
        iifname "<YOUR-INTERFACE>" drop
    }
    chain output {
        type filter hook output priority -1; policy accept;
        oifname "<YOUR-INTERFACE>" drop
    }
}
```

> Replace `<YOUR-INTERFACE>` with your physical network interface name.

### How to apply

```bash
# Create the nftables file:
$ sudo tee /etc/nftables/killswitch6.nft << 'EOF'
table ip6 killswitch6 {
    chain input {
        type filter hook input priority -1; policy accept;
        iifname "<YOUR-INTERFACE>" drop
    }
    chain output {
        type filter hook output priority -1; policy accept;
        oifname "<YOUR-INTERFACE>" drop
    }
}
EOF

# Add to nftables config for persistence:
$ echo 'include "/etc/nftables/killswitch6.nft"' | sudo tee -a /etc/sysconfig/nftables.conf

# Load immediately:
$ sudo nft -f /etc/nftables/killswitch6.nft
```

### Decision

```
IPv6 killswitch
├── Apply if:    You use a WireGuard-based VPN (recommended)
│   ├── WireGuard always creates a dual-stack socket
│   ├── all.disable_ipv6=1 is NOT an option (breaks VPN killswitch)
│   └── This rule is your defense-in-depth for IPv6
│
├── Skip if:     You don't use a VPN
│   └── Set all.disable_ipv6=1 instead (Doc 02, Section 10 — simpler)
│
└── What breaks:  Nothing — your system doesn't use IPv6 for internet access
    IPv6 on loopback and VPN interfaces is unaffected (rules only match <YOUR-INTERFACE>)
```

### Verify

```bash
$ sudo nft list table ip6 killswitch6
# Expected: table with input/output chains dropping traffic on <YOUR-INTERFACE>
```

> **Unlike the IPv4 killswitch, this table is NOT managed by your VPN client.** You create and maintain it. It does not change when you switch VPN servers.

---

## 4. Layer 3: Policy Routing (fwmark)

### What

Policy routing rules (managed by your VPN client) that ensure all non-VPN traffic is routed through the tunnel interface. If the tunnel is down, traffic falls through to the killswitch dummy interface and is blackholed.

### How it works (simplified)

```
1. VPN traffic gets a firewall mark (fwmark) set by the VPN client
2. IP rules:
   ├── Traffic WITH fwmark → normal routing → physical NIC → VPN server
   └── Traffic WITHOUT fwmark → VPN routing table → tunnel interface
3. VPN routing table has only: default dev <YOUR-VPN-INTERFACE>
4. If tunnel is down: fallback default route → killswitch dummy interface → blackholed
```

### Routing table structure

| Evaluation order | Rule | Purpose |
|-----------------|------|---------|
| 1st | `lookup main suppress_prefixlength 0` | Use main table for subnet routes, but ignore default route |
| 2nd | `not fwmark <VPN-MARK> lookup <VPN-TABLE>` | Non-VPN traffic → VPN routing table (tunnel only) |
| 3rd | `lookup main` | Normal routing (only reached by VPN-marked traffic) |

### Failsafe: killswitch default route

```
default via <KS-IP> dev <YOUR-VPN-KS-INTERFACE> metric 98
```

If the tunnel interface goes down, this default route (with higher metric) catches all traffic and sends it to the dummy killswitch interface — where it goes nowhere.

> **You don't configure policy routing manually.** Your VPN client sets up all rules, routes, and fwmarks automatically.

---

## 5. Layer 4: block-lan-out Policy (User-Managed)

This is the **only killswitch layer you configure yourself**. It is fully documented in [Doc 03 — Firewall](03-firewall.md), Section 6.

```
block-lan-out (priority 100, HOST → strict-wan):
  UDP 3702 → DROP (wsdd)
  10.0.0.0/8 → DROP
  172.16.0.0/12 → DROP
  192.168.0.0/16 → DROP
  224.0.0.0/4 → DROP (multicast)
  255.255.255.255/32 → DROP (broadcast)
```

Even if the VPN killswitch table and policy routing both fail, `block-lan-out` drops all RFC1918 traffic, multicast, and broadcast on your physical interface. This is your safety net.

---

## 6. Packet Flow: VPN Active

```
Application sends packet (e.g., to 1.1.1.1)
  │
  ├─ Kernel routing lookup:
  │   ├─ Suppress main default route
  │   ├─ No fwmark → VPN routing table → default dev <VPN-INTERFACE>
  │   └─ Packet goes to tunnel interface
  │
  ├─ WireGuard encrypts packet
  │   └─ Encapsulated UDP to <YOUR-VPN-SERVER-IP>
  │
  ├─ Kernel routing for encrypted packet:
  │   ├─ fwmark set by VPN → normal routing
  │   ├─ Main table: <YOUR-VPN-SERVER-IP> via <YOUR-GATEWAY-IP> dev <YOUR-INTERFACE>
  │   └─ Encrypted packet goes to physical NIC
  │
  └─ nftables killswitch: destination == <YOUR-VPN-SERVER-IP> → ACCEPT
```

**Result:** Only encrypted VPN traffic reaches your physical interface.

---

## 7. Packet Flow: VPN Failure

```
Application sends packet
  │
  ├─ Tunnel interface down → VPN routing table unusable
  │
  ├─ Fallback: default via killswitch dummy interface (higher metric)
  │   └─ Packet sent to dummy interface → silently discarded
  │
  ├─ Even if packet somehow reaches physical NIC:
  │   ├─ nftables killswitch: destination ≠ VPN server → DROP
  │   └─ block-lan-out: RFC1918/multicast/broadcast → DROP
  │
  └─ Result: ZERO packets leave the system unencrypted
```

**Four independent failures would need to occur simultaneously** for a leak: IPv4 killswitch removal + IPv6 killswitch removal + policy routing bypass + block-lan-out policy deletion. This is effectively impossible under normal operation.

---

## 8. Configuration Dependencies

These settings must be correct for the VPN killswitch to work. All are configured in other documents:

| Setting | Required value | Why | Configured in |
|---------|---------------|-----|---------------|
| `sysctl rp_filter` | `2` (loose) | Strict mode drops VPN return packets | [Doc 02](02-sysctl-hardening.md) |
| `sysctl all.disable_ipv6` | `0` (NOT disabled globally) | Killswitch interface needs IPv6 blackhole | [Doc 02](02-sysctl-hardening.md) |
| `firewalld IPv6_rpfilter` | `loose` | Same as rp_filter — VPN compatible | [Doc 03](03-firewall.md) |
| `firewalld DefaultZone` | `drop` | VPN interfaces inherit DROP (no exposure) | [Doc 03](03-firewall.md) |
| `drop zone forward` | `yes` | VPN forwarding between interfaces | [Doc 03](03-firewall.md) |
| `strict-wan zone forward` | `no` | No forwarding on physical NIC | [Doc 03](03-firewall.md) |

> **If any of these are wrong, your VPN connection will break.** Verify each one using the respective doc's verification commands.

---

## 🔧 Applying Changes

Most killswitch configuration is automatic. Here's what you need to do:

### Step 1: Enable killswitch in your VPN client

Each VPN provider has different UI/settings:
- Look for "Kill Switch", "Always-on VPN", or "Block connections without VPN"
- Ensure WireGuard is selected as the protocol (not OpenVPN)

### Step 2: Verify firewall zones

```bash
$ firewall-cmd --get-active-zones
# Your VPN interfaces should be in the "drop" zone
# Your physical interface should be in "strict-wan"
```

If VPN interfaces are not in `drop`, assign them:

```bash
$ sudo firewall-cmd --permanent --zone=drop --add-interface=<YOUR-VPN-INTERFACE>
$ sudo firewall-cmd --permanent --zone=drop --add-interface=<YOUR-VPN-KS-INTERFACE>
$ sudo firewall-cmd --reload
```

### Step 3: Verify sysctl compatibility

```bash
$ sysctl net.ipv4.conf.all.rp_filter
# Must be: 2 (loose), NOT 1

$ sysctl net.ipv6.conf.all.disable_ipv6
# Must be: 0 (NOT globally disabled)
```

### Step 4: Ensure block-lan-out exists

See [Doc 03](03-firewall.md) for creating the block-lan-out policy. This is your fourth killswitch layer.

---

## ✅ Complete Verification

```bash
# 1. VPN connection active:
$ nmcli con show --active | grep -i vpn
# Expected: Your VPN connection listed as active

# 2. Killswitch nftables table:
$ sudo nft list table ip killswitch
# Expected: output chain with LAN DROP + only VPN server IP allowed

# 3. Policy routing rules:
$ ip rule show
# Expected: fwmark-based rule pointing to VPN routing table

# 4. VPN routing table:
$ ip route show table all | grep "dev <YOUR-VPN-INTERFACE>"
# Expected: default dev <YOUR-VPN-INTERFACE>

# 5. Killswitch default route:
$ ip route show default
# Expected: Two default routes — one via VPN (lower metric), one via killswitch (higher metric)

# 6. Firewall zones:
$ firewall-cmd --get-active-zones
# Expected: VPN interfaces in "drop", physical interface in "strict-wan"

# 7. Leak test:
$ curl https://ifconfig.me
# Expected: Shows your VPN server's IP, NOT your real IP

# 8. IPv6 killswitch table (user-managed):
$ sudo nft list table ip6 killswitch6
# Expected: table with input/output chains dropping traffic on <YOUR-INTERFACE>

# 9. block-lan-out active:
$ sudo firewall-cmd --policy=block-lan-out --list-all
# Expected: 6 rich rules (wsdd + RFC1918 + multicast + broadcast)
```

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| Quadruple killswitch | No internet without VPN | Zero possibility of IP/DNS leaks |
| `rp_filter = 2` (loose) | Slightly weaker anti-spoofing than strict | Required for WireGuard policy routing |
| `all.disable_ipv6 ≠ 1` | IPv6 not globally disabled | Killswitch IPv6 blackhole route works |
| VPN client manages nftables | Must trust VPN client software | Automatic server IP updates on reconnect |
| block-lan-out (manual layer) | Cannot reach LAN devices | Independent safety net if VPN client fails |

---

## Notes

- **No internet without VPN:** With the killswitch enabled, your system has zero internet access when the VPN is disconnected. This is intentional — not a bug.
- **VPN server IP changes:** When you switch VPN servers, your client automatically updates the killswitch nftables table with the new server IP. No manual action needed.
- **WireGuard kernel socket:** `ss` may show a UDP socket with `uid=65534` (nobody). This is normal — WireGuard runs in the kernel, not userspace.
- **`firewall-cmd --zone=block-lan-out` returns INVALID_ZONE:** This is correct — `block-lan-out` is a policy, not a zone. Use `--policy=block-lan-out` ([Doc 03](03-firewall.md)).
- **Connmark/fwmark:** The firewall mark value is generated by your VPN client and used for policy routing. It is not personal or identifying — it's a local routing identifier.
- **NftablesTableOwner:** firewalld with `NftablesTableOwner=yes` ([Doc 03](03-firewall.md)) will delete the killswitch table on reload. The VPN client recreates it automatically on reconnect.

---

*Previous: [05 — LAN Isolation](05-lan-isolation.md)*
*Next: [07 — IPv6 Hardening](07-ipv6-hardening.md) — Selective IPv6 disable with defense-in-depth*
