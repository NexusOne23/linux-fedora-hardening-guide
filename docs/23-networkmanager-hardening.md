# 📡 NetworkManager Hardening

> Static IP, MAC randomization, disabled connectivity checks, and firewall zone binding.
> Applies to: Fedora Workstation 43+ | All hardware with Ethernet or WiFi
> No reboot required — changes apply on connection restart.

---

## Overview

NetworkManager is Fedora's default network configuration tool. By default, it uses DHCP (trusting the network), broadcasts the real MAC address, pings external servers for connectivity checks, and enables LLDP. Each of these is a privacy leak or attack vector.

### Why

| Default behavior | Risk | Hardening |
|-----------------|------|-----------|
| DHCP for IP/DNS | DHCP spoofing, rogue DNS injection | Static IP configuration |
| Real MAC address | Device tracking across networks | MAC randomization (`stable`) |
| Connectivity check | Metadata leak to fedoraproject.org | Disable connectivity check |
| LLDP listener active | Network topology data received from switches | Disable LLDP |
| No zone binding | Interface may land in wrong firewall zone | Explicit firewall zone binding |

### What you get

| Setting | Configuration |
|---------|--------------|
| IP method | Static (no DHCP) |
| MAC address | `stable` — randomized but consistent per network |
| LLDP | Disabled |
| Connectivity check | Disabled |
| IPv6 | Disabled on physical interface (→ Doc 07) |
| Firewall zone | Bound to hardened zone (→ Doc 03) |
| Permissions | Only your user can modify the connection |

---

## 1. Static IP Configuration

### Why static IP

| Aspect | DHCP | Static IP |
|--------|------|-----------|
| IP assignment | Router assigns — can change | Fixed — you control it |
| DNS injection | Router pushes DNS servers | No DNS from network |
| DHCP spoofing | Attacker can impersonate router | Not possible — no DHCP client |
| ARP hardening | Harder (IP may change) | Compatible (→ Doc 04) |
| Predictability | Network config may change | Always known |

### How

```bash
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses <YOUR-STATIC-IP>/24 \
  ipv4.gateway <YOUR-GATEWAY-IP> \
  ipv4.ignore-auto-dns yes
```

> **Find your current IP and gateway**:
> ```bash
> ip addr show <YOUR-INTERFACE>    # current IP
> ip route | grep default           # current gateway
> ```

### Decision

```
IP configuration method
├── Static IP (recommended for hardened desktops):
│   ├── No DHCP = no DHCP-based attacks
│   ├── No DNS injection from network
│   ├── Required for ARP hardening (→ Doc 04)
│   └── Prerequisites: know your network's subnet and gateway
│
├── DHCP (acceptable for laptops/mobile use):
│   ├── Automatic configuration — works on any network
│   ├── Risk: DHCP spoofing, rogue DNS, IP changes
│   ├── Mitigation: ignore-auto-dns=yes (don't accept DNS from DHCP)
│   └── Use if: you connect to many different networks
│
└── What breaks with static IP:
    ├── Must manually configure IP when changing networks
    ├── IP conflicts if another device uses the same address
    └── Not practical for WiFi on multiple networks (use DHCP there)
```

---

## 2. MAC Address Randomization

### Why

Your MAC address is a unique hardware identifier. Without randomization, every network you connect to (and every device on that network) can track your device permanently.

### Modes

| Mode | Behavior | Use case |
|------|----------|----------|
| `permanent` | Real hardware MAC | Never — allows permanent tracking |
| `random` | New random MAC on every connection | Public WiFi — maximum anonymity |
| **`stable`** | **Random but consistent per network** | **Home network — recommended** |

### How `stable` works

`stable` generates a deterministic MAC based on:
- Connection UUID (unique per saved connection)
- Machine ID (`/etc/machine-id`)

Result: your MAC is **different from the hardware MAC** (privacy) but **stays the same for each network** (DHCP lease stability, ARP hardening compatibility, no router confusion).

### How

```bash
nmcli connection modify "Wired connection 1" \
  802-3-ethernet.cloned-mac-address stable
```

### Decision

```
MAC address strategy
├── stable (recommended for home/office):
│   ├── Hides real MAC from network
│   ├── Same MAC per network — DHCP leases survive reconnects
│   ├── Compatible with ARP hardening (→ Doc 04)
│   └── Compatible with MAC-based router access lists
│
├── random (maximum privacy — public WiFi):
│   ├── New MAC every time you connect
│   ├── Maximum anonymity — no tracking between sessions
│   ├── Breaks: DHCP lease continuity, router MAC filters
│   └── Use on: untrusted networks, coffee shops, airports
│
├── permanent (not recommended):
│   ├── Uses real hardware MAC
│   ├── Allows permanent device tracking
│   └── Only if: network equipment requires the real MAC
│
└── Verify your MAC is randomized:
    ip link show <YOUR-INTERFACE>
    # Compare with: cat /sys/class/net/<YOUR-INTERFACE>/address
    # addr_assign_type: 0=permanent, 1=random, 3=set (cloned/stable)
    cat /sys/class/net/<YOUR-INTERFACE>/addr_assign_type
    # Expected: 3 (set)
```

---

## 3. LLDP (Link Layer Discovery Protocol)

### Why disable

LLDP allows your system to receive network topology information from switches — VLAN IDs, switch hostnames, port descriptions. NetworkManager listens for LLDP frames by default. While NM doesn't actively broadcast, the received data can reveal your network infrastructure to applications, and some configurations may respond to LLDP queries.

### How

```bash
nmcli connection modify "Wired connection 1" \
  connection.lldp 0
```

### Decision

```
LLDP
├── Disable (recommended):
│   ├── No network topology information leaked
│   ├── No impact on normal networking
│   └── Default on most connections, but verify
│
└── Keep enabled if:
    ├── You manage network infrastructure and need LLDP data
    └── Corporate networks that require LLDP for VLAN assignment
```

---

## 4. Connectivity Check

### What

NetworkManager periodically pings `fedoraproject.org` (HTTP request to `http://fedoraproject.org/static/hotspot.txt`) to detect captive portals and verify internet connectivity.

### Why disable

- **Metadata leak**: Your ISP (or VPN exit) sees DNS queries + HTTP requests to fedoraproject.org
- **Unnecessary with VPN**: If you use a VPN killswitch, connectivity is binary — VPN up = internet, VPN down = no internet
- **Privacy**: The request reveals you're running Fedora

### How

Create `/etc/NetworkManager/conf.d/99-privacy.conf`:

```ini
[connectivity]
enabled=false
```

> **Why a drop-in file**: Files in `conf.d/` are never overwritten by `dnf upgrade`. The main `/etc/NetworkManager/NetworkManager.conf` can be replaced during updates. Custom config belongs in `conf.d/`.

Then reload:

```bash
sudo systemctl reload NetworkManager
```

### Decision

```
Connectivity check
├── Disable (recommended):
│   ├── No metadata leak to fedoraproject.org
│   ├── No impact on actual connectivity
│   ├── GNOME may show "Limited connectivity" icon occasionally
│   └── Actual internet access works normally
│
└── Keep enabled if:
    ├── You frequently connect to captive portals (hotels, airports)
    ├── Captive portal detection requires the connectivity check
    └── Trade-off: convenience vs privacy leak
```

---

## 5. Firewall Zone Binding

### Why

Without explicit zone binding, NetworkManager assigns interfaces to the default firewall zone. If the default zone is permissive, your interface has weak firewall rules.

### How

```bash
nmcli connection modify "Wired connection 1" \
  connection.zone <YOUR-FIREWALL-ZONE>
```

Where `<YOUR-FIREWALL-ZONE>` is your hardened zone from Doc 03 (e.g., a zone with target `DROP`).

### Verify

```bash
nmcli -f connection.zone connection show "Wired connection 1"
# Expected: your hardened zone name

firewall-cmd --get-active-zones
# Expected: your interface listed under the correct zone
```

---

## 6. Connection Permissions

### Why

By default, any user on the system can see and use NetworkManager connections. On a single-user desktop this is less critical, but restricting permissions is defense-in-depth.

### How

```bash
nmcli connection modify "Wired connection 1" \
  connection.permissions "user:<YOUR-USERNAME>"
```

### What this does

| Without permissions | With permissions |
|--------------------|-----------------|
| Any local user can activate/deactivate the connection | Only `<YOUR-USERNAME>` can use it |
| Other users can see connection details (including IPs) | Connection is invisible to other users |
| Shared system risk | Single-user lockdown |

---

## 7. IPv6 on Physical Interface

### How

```bash
nmcli connection modify "Wired connection 1" \
  ipv6.method disabled
```

Full IPv6 hardening details: → Doc 07.

---

## 8. VPN Killswitch (NetworkManager Integration)

Most VPN clients create their own NetworkManager connections for the tunnel and killswitch. These are typically:

| Connection | Interface | Purpose |
|------------|-----------|---------|
| VPN tunnel | `<YOUR-VPN-INTERFACE>` | Encrypted tunnel (WireGuard/OpenVPN) |
| Killswitch | `<YOUR-VPN-KS-INTERFACE>` | Dummy interface — blackhole route for leak prevention |

### How the killswitch works

The killswitch connection has:
- **Blackhole DNS** (`dns=0.0.0.0`) — DNS queries go nowhere
- **High DNS priority** (`dns-priority=-1400`) — overrides all other DNS
- **Low route metric** — killswitch routes take priority when VPN disconnects

When VPN is active: traffic flows through the VPN tunnel.
When VPN disconnects: traffic falls back to the killswitch routes → routed to unreachable addresses → no traffic leak.

> **Note**: These connections are managed by your VPN client — don't modify them manually unless you know what you're doing. For full killswitch architecture: → Doc 06.

---

## 9. Complete Configuration (All-in-One)

```bash
# 1. Disable connectivity check:
sudo tee /etc/NetworkManager/conf.d/99-privacy.conf << 'EOF'
[connectivity]
enabled=false
EOF

# 2. Configure Ethernet connection:
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses <YOUR-STATIC-IP>/24 \
  ipv4.gateway <YOUR-GATEWAY-IP> \
  ipv4.ignore-auto-dns yes \
  ipv6.method disabled \
  connection.zone <YOUR-FIREWALL-ZONE> \
  802-3-ethernet.cloned-mac-address stable \
  connection.lldp 0 \
  connection.permissions "user:<YOUR-USERNAME>"

# 3. Restart NetworkManager:
sudo systemctl restart NetworkManager
```

---

## 10. Complete Verification

```bash
# 1. Static IP:
ip addr show <YOUR-INTERFACE> | grep "inet "
# Expected: your static IP

# 2. MAC randomization:
cat /sys/class/net/<YOUR-INTERFACE>/addr_assign_type
# Expected: 3 (set/cloned)

# 3. LLDP disabled:
nmcli -f connection.lldp connection show "Wired connection 1"
# Expected: 0

# 4. Connectivity check disabled:
cat /etc/NetworkManager/conf.d/99-privacy.conf
# Expected: enabled=false

# 5. No DHCP DNS:
resolvectl status <YOUR-INTERFACE> 2>/dev/null
# Expected: Current Scopes: none (no DNS from this interface)

# 6. IPv6 disabled:
nmcli -f ipv6.method connection show "Wired connection 1"
# Expected: disabled

# 7. Firewall zone:
nmcli -f connection.zone connection show "Wired connection 1"
# Expected: your hardened zone name

# 8. Permissions:
nmcli -f connection.permissions connection show "Wired connection 1"
# Expected: user:<YOUR-USERNAME>
```

---

## Important Notes

- **Static IP + ARP hardening**: Static IP is a prerequisite for ARP hardening (→ Doc 04). DHCP could change your IP and break static ARP entries.
- **MAC `stable` + ARP**: The `stable` MAC stays the same per network, so your router's ARP table entry remains valid. `random` would break ARP hardening.
- **Drop-in convention**: Always use `/etc/NetworkManager/conf.d/` for custom config — avoid editing the main `NetworkManager.conf` (may generate `.rpmnew` conflicts during updates).
- **Connection name**: Fedora names the default Ethernet connection "Wired connection 1" on English systems. Your name may differ — check with `nmcli connection show`.
- **VPN server route**: If you use static IP with a VPN, you may need a host route to your VPN server via the local gateway. Your VPN client typically handles this automatically. If not: `nmcli connection modify "Wired connection 1" +ipv4.routes "<VPN-SERVER-IP>/32 <YOUR-GATEWAY-IP>"`.
- **WiFi connections**: If you use WiFi, create separate connection profiles with their own hardening settings. Consider `random` MAC for untrusted WiFi networks.

---

*Previous: [22 — LUKS Encryption](22-luks-encryption.md)*
*Next: [24 — Firmware Updates](24-firmware-updates.md) — fwupd, LVFS, UEFI dbx management*
