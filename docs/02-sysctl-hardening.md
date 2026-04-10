# ⚙️ Sysctl Hardening

> 64 kernel runtime parameters for memory, process, network, and filesystem security.
> Applies to: Fedora Workstation 43+ | All x86_64 hardware
> No reboot required — changes apply immediately via `sysctl -p`.

---

## Overview

Sysctl parameters control kernel behavior at runtime. Unlike boot parameters ([Doc 01](01-kernel-boot-params.md)), these can be changed without rebooting — but they must be persisted in config files to survive reboots.

This document configures **64 parameters** across three files:
- `/etc/sysctl.d/99-hardening.conf` — 61 parameters (main hardening config)
- `/etc/sysctl.d/99-audit-fixes.conf` — 2 parameters (kernel stability limits)
- `/etc/sysctl.d/99-userns.conf` — 1 parameter (user namespace limit)

### Category Breakdown

| Category | Parameters | Section |
|----------|-----------|---------|
| Kernel Security | 14 | [1](#1-kernel-security-14-parameters) |
| Kernel Stability | 2 | [2](#2-kernel-stability-2-parameters) |
| BPF JIT Hardening | 1 | [3](#3-bpf-jit-hardening-1-parameter) |
| Filesystem Protection | 5 | [4](#4-filesystem-protection-5-parameters) |
| Network: IPv4 Routing | 12 | [5](#5-network-ipv4-routing-12-parameters) |
| Network: ICMP | 3 | [6](#6-network-icmp-3-parameters) |
| Network: TCP | 2 | [7](#7-network-tcp-2-parameters) |
| Network: ARP | 12 | [8](#8-network-arp-12-parameters) |
| Network: IGMP/Multicast | 1 | [9](#9-network-igmpmulticast-1-parameter) |
| Network: IPv6 | 11 | [10](#10-network-ipv6-11-parameters) |
| User Namespaces | 1 | [11](#11-user-namespaces--limited-not-disabled) |

> **Related:** Boot-level kernel hardening is in [Doc 01](01-kernel-boot-params.md). ARP hardening at the nftables layer is in [Doc 04](04-nftables-arp-hardening.md). IPv6 hardening is in [Doc 07](07-ipv6-hardening.md).

---

## 1. Kernel Security (14 parameters)

### What

Parameters that restrict access to kernel internals, disable dangerous subsystems, and harden memory layout.

### Why

An unprivileged attacker (or compromised application) should learn as little as possible about the kernel's internal state. These parameters hide kernel pointers, restrict debugging interfaces, disable legacy attack surfaces, and maximize memory randomization.

### Parameters

| Parameter | Value | What it does | Threat mitigated |
|-----------|-------|-------------|------------------|
| `kernel.kptr_restrict` | `2` | Hides kernel pointers in `/proc/kallsyms` (even from root) | Kernel ASLR bypass via address disclosure |
| `kernel.yama.ptrace_scope` | `2` | Only processes with `CAP_SYS_PTRACE` can use ptrace | Process memory inspection, credential theft |
| `kernel.sysrq` | `0` | Disables all SysRq key combinations | Physical keyboard attacks (instant reboot, memory dump) |
| `kernel.kexec_load_disabled` | `1` | Prevents loading a new kernel at runtime | Rootkit loading unsigned kernel via kexec |
| `kernel.dmesg_restrict` | `1` | Restricts `dmesg` to `CAP_SYSLOG` | Kernel information leaks (addresses, driver info) |
| `kernel.unprivileged_bpf_disabled` | `2` | Permanently disables BPF for unprivileged users | BPF-based kernel exploits |
| `kernel.perf_event_paranoid` | `3` | Disables perf events for unprivileged users (KSPP) | Kernel profiling side channels |
| `kernel.randomize_va_space` | `2` | Full ASLR (stack + heap + libraries + VDSO) | Memory layout prediction for exploits |
| `dev.tty.ldisc_autoload` | `0` | Disables automatic loading of TTY line disciplines | Kernel module auto-load exploitation |
| `dev.tty.legacy_tiocsti` | `0` | Blocks TIOCSTI ioctl (character injection into other terminals) | Privilege escalation via terminal injection |
| `vm.unprivileged_userfaultfd` | `0` | Restricts userfaultfd to `CAP_SYS_PTRACE` | Use-after-free exploit timing control |
| `vm.mmap_min_addr` | `65536` | Minimum address for mmap allocations | NULL pointer dereference kernel exploits |
| `kernel.io_uring_disabled` | `2` | Disables io_uring for unprivileged users | Dozens of io_uring CVEs since 2020 |
| `vm.mmap_rnd_bits` | `32` | Maximum ASLR entropy for mmap (default: 28) | Reduced ASLR brute-force resistance |

### Decision

```
Kernel Security parameters
├── Set ALL:     Recommended for every desktop and server
├── Notable decisions:
│
├── kernel.sysrq = 0
│   ├── Set 0 if:   Desktop with physical access control (recommended)
│   ├── Set 176 if: You need emergency sync + reboot via keyboard (Alt+SysRq+S, U, B)
│   └── Note:       Ctrl+Alt+Del works regardless of sysrq — it's handled by systemd
│
├── kernel.perf_event_paranoid = 3
│   ├── Set 3 if:   Not doing kernel/application profiling (recommended)
│   ├── Set 2 if:   You need `perf` without sudo for development
│   └── What breaks: `perf`, `flamegraph`, any profiling tool (requires sudo with 3)
│
├── kernel.io_uring_disabled = 2
│   ├── Set 2:      Always — no desktop application uses io_uring
│   ├── Note:       Google disabled io_uring in ChromeOS and Android
│   └── What breaks: Nothing on desktop (io_uring is a server/database feature)
│
└── vm.mmap_rnd_bits = 32
    ├── Set 32:     Maximum ASLR entropy on x86_64 (KSPP recommendation)
    ├── Default:    28 (4 fewer bits = 16× easier to brute-force)
    └── What breaks: Nothing
```

### Config

```bash
# === Kernel Security ===
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 2
kernel.sysrq = 0
kernel.kexec_load_disabled = 1
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 2
kernel.perf_event_paranoid = 3
kernel.randomize_va_space = 2
dev.tty.ldisc_autoload = 0
dev.tty.legacy_tiocsti = 0
vm.unprivileged_userfaultfd = 0
vm.mmap_min_addr = 65536
kernel.io_uring_disabled = 2
vm.mmap_rnd_bits = 32
```

---

## 2. Kernel Stability (2 parameters)

### What

Limits on kernel oops and warning events before the kernel panics. These are stored in a **separate file** (`/etc/sysctl.d/99-audit-fixes.conf`).

### Why

Kernel exploits often trigger hundreds of oops events in rapid succession (exploit loops). An `oops_limit` forces a panic (clean reboot) after a threshold, stopping the exploit. However, setting the limit too low causes unnecessary reboots from harmless single oops events — especially with proprietary GPU drivers that occasionally trigger warnings.

### Parameters

| Parameter | Value | What it does | Threat mitigated |
|-----------|-------|-------------|------------------|
| `kernel.oops_limit` | `100` | Kernel panics after 100 oops events | Exploit loops that generate repeated oops |
| `kernel.warn_limit` | `0` | Disabled (no limit on warnings) | — |

### Decision

```
kernel.oops_limit
├── Set 100:   Recommended — stops exploit loops, tolerates occasional oops
├── Set 1:     Too aggressive for desktop — proprietary GPU drivers may cause single oops
└── Set 0:     Disabled (no limit) — not recommended, allows unlimited exploit attempts

kernel.warn_limit
├── Set 0:     Recommended (disabled) — warnings are informational, not exploitable
├── Set 1:     Too aggressive — NVIDIA drivers can produce harmless WARN_ON() calls
└── Note:      This is in 99-audit-fixes.conf, NOT in 99-hardening.conf
```

### Config (separate file)

```bash
# File: /etc/sysctl.d/99-audit-fixes.conf
kernel.oops_limit = 100
kernel.warn_limit = 0
```

---

## 3. BPF JIT Hardening (1 parameter)

### What

Hardens the BPF Just-In-Time compiler output.

### Why

Even when unprivileged BPF is disabled (Section 1), privileged BPF programs still use the JIT compiler. Without hardening, JIT-compiled BPF code is predictable — enabling JIT spray attacks where an attacker fills the kernel's executable memory with gadgets. `bpf_jit_harden = 2` applies constant blinding and makes JIT output unreadable.

### Parameter

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `net.core.bpf_jit_harden` | `2` | Hardens BPF JIT for all users (constant blinding, no readable JIT code) |

### Config

```bash
net.core.bpf_jit_harden = 2
```

> Complementary to `unprivileged_bpf_disabled = 2` — belt and suspenders.

---

## 4. Filesystem Protection (5 parameters)

### What

Parameters that prevent file-based attacks in shared directories.

### Why

Shared directories (like `/tmp`) with the sticky bit are classic attack surfaces. Without these protections, an attacker can:
- Create symlinks pointing to sensitive files (TOCTOU attacks)
- Create hardlinks to SUID binaries for offline exploitation
- Open regular files or FIFOs created by other users in sticky directories
- Extract sensitive data from core dumps of SUID programs

### Parameters

| Parameter | Value | What it does | Threat mitigated |
|-----------|-------|-------------|------------------|
| `fs.suid_dumpable` | `0` | No core dumps from SUID/SGID programs | Credential/memory extraction from privileged crashes |
| `fs.protected_regular` | `2` | Files in sticky dirs only openable by owner | File squatting in /tmp |
| `fs.protected_fifos` | `2` | FIFOs in sticky dirs only openable by owner | FIFO-based data interception |
| `fs.protected_hardlinks` | `1` | Hardlinks only to files you own | Hardlink-based SUID exploitation |
| `fs.protected_symlinks` | `1` | Symlinks in sticky dirs only followable by owner | TOCTOU symlink attacks in /tmp |

### Decision

```
Filesystem protection
├── Set ALL:     Always — these are universally safe
├── Note:        Most are Fedora defaults, explicitly set as defense-in-depth
└── What breaks: Nothing — these only restrict malicious usage patterns
```

### Config

```bash
# === Filesystem Protection ===
fs.suid_dumpable = 0
fs.protected_regular = 2
fs.protected_fifos = 2
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

---

## 5. Network: IPv4 Routing (12 parameters)

### What

Parameters that prevent IPv4 routing manipulation attacks.

### Why

On a network, attackers can redirect your traffic by sending forged ICMP redirect messages, injecting source routes into packets, or spoofing IP addresses. These parameters disable all of these vectors:

- **ICMP redirects**: Allow a device to tell your system "send traffic for X via me instead" — used in MITM attacks.
- **Source routing**: Allows a packet's sender to dictate its path through the network — used for IP spoofing.
- **Reverse path filtering**: Drops packets with impossible source addresses — prevents spoofed traffic.
- **Martian logging**: Logs packets with addresses that shouldn't exist on your network — detects scanning/spoofing.

### Parameters

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `conf.all.accept_redirects` | `0` | Ignore ICMP redirect messages |
| `conf.default.accept_redirects` | `0` | *(same, for new interfaces)* |
| `conf.all.secure_redirects` | `0` | Ignore "secure" ICMP redirects (from default gateway) |
| `conf.default.secure_redirects` | `0` | *(same, for new interfaces)* |
| `conf.all.send_redirects` | `0` | Never send ICMP redirects |
| `conf.default.send_redirects` | `0` | *(same, for new interfaces)* |
| `conf.all.accept_source_route` | `0` | Ignore source-routed packets |
| `conf.default.accept_source_route` | `0` | *(same, for new interfaces)* |
| `conf.all.log_martians` | `1` | Log packets with impossible source addresses |
| `conf.default.log_martians` | `1` | *(same, for new interfaces)* |
| `conf.all.rp_filter` | `2` | Reverse path filtering — loose mode |
| `conf.default.rp_filter` | `2` | *(same, for new interfaces)* |

### Decision

```
rp_filter (Reverse Path Filtering)
├── Set 2 (loose) if:  You use a WireGuard-based VPN
│   └── Why: WireGuard VPN clients use policy routing with fwmark. Strict mode (1)
│            drops VPN return packets because the source IP isn't routable via the
│            expected interface. Loose mode (2) only checks that the source IP is
│            routable at all — VPN-compatible.
├── Set 1 (strict) if: No VPN — maximum anti-spoofing protection
└── What breaks:       rp_filter=1 breaks most WireGuard VPN connections

All other routing parameters
├── Set ALL to 0/1 as shown — no exceptions needed
└── What breaks: Nothing — a desktop should never route, redirect, or accept source routes
```

### Config

```bash
# === IPv4: Routing Protection ===
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
```

> **No VPN?** Change `rp_filter` to `1` for strict anti-spoofing.

---

## 6. Network: ICMP (3 parameters)

### What

Parameters that control how your system responds to ICMP (ping) requests.

### Why

ICMP echo responses reveal your system's existence and activity on the network. Broadcast pings (Smurf attacks) can amplify into denial-of-service floods. Bogus ICMP error responses waste system resources and can be used for fingerprinting.

### Parameters

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `icmp_echo_ignore_all` | `1` | Ignores all ICMP echo (ping) requests — system is invisible to ping |
| `icmp_echo_ignore_broadcasts` | `1` | Ignores broadcast/multicast pings — prevents Smurf amplification |
| `icmp_ignore_bogus_error_responses` | `1` | Ignores ICMP error responses that violate RFC 1122 |

### Decision

```
icmp_echo_ignore_all
├── Set 1 if:    Maximum network invisibility (recommended for hardened desktop)
├── Set 0 if:    You need to be pingable (e.g., for network monitoring or troubleshooting)
├── Note:        Does NOT affect outbound connections — you can still reach everything
└── What breaks: Other devices cannot ping you (may confuse network admins/monitoring)

icmp_echo_ignore_broadcasts / icmp_ignore_bogus_error_responses
├── Set to 1:    Always — no reason to respond to broadcast pings or accept bogus errors
└── What breaks: Nothing
```

### Config

```bash
# === IPv4: ICMP ===
net.ipv4.icmp_echo_ignore_all = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
```

---

## 7. Network: TCP (2 parameters)

### What

Parameters that harden the TCP stack against flood and sequence attacks.

### Why

- **SYN floods**: An attacker sends thousands of SYN packets without completing the handshake, exhausting your connection table. SYN cookies allow the kernel to handle this without allocating state until the handshake completes.
- **TIME-WAIT assassination** (RFC 1337): An attacker sends a RST to a socket in TIME-WAIT state, prematurely closing it. This can cause connection reuse issues and data corruption.

### Parameters

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `tcp_syncookies` | `1` | Enables SYN cookies (stateless SYN flood protection) |
| `tcp_rfc1337` | `1` | Drops RST packets for sockets in TIME-WAIT state |

### Decision

```
TCP hardening
├── Set BOTH:    Always — universally safe, zero performance impact
└── What breaks: Nothing
```

### Config

```bash
# === IPv4: TCP ===
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
```

---

## 8. Network: ARP (12 parameters)

### What

Twelve ARP parameters across three scopes that harden Layer 2 address resolution.

### Why

ARP (Address Resolution Protocol) maps IP addresses to MAC addresses on local networks. It has **zero authentication** — any device on your LAN can claim to be your gateway, intercepting all your traffic (ARP spoofing/poisoning). These parameters restrict how your system responds to and sends ARP messages.

> **ARP spoofing is trivial.** Tools like `arpspoof` or `ettercap` can redirect traffic on any LAN in seconds. These sysctl parameters are your first defense layer — combined with nftables ARP filtering in [Doc 04](04-nftables-arp-hardening.md).

### Why Triple Scope?

Each parameter is set for three scopes:

| Scope | Applies to | Why needed |
|-------|-----------|------------|
| `conf.all.*` | All currently existing interfaces | Covers interfaces already present at sysctl load time |
| `conf.default.*` | Newly created interfaces (inherited) | Covers VPN tunnels, bridges, or other interfaces created after boot |
| `conf.<YOUR-INTERFACE>.*` | Your specific network interface | Explicit override — ensures your primary NIC is hardened even if `all` behaves unexpectedly |

All three are needed for complete coverage. Without `default`, a VPN tunnel created after boot would be unprotected. Without the explicit interface, some kernel versions may not apply `all` consistently.

> **Find your interface name:** `ip link show` — look for your Ethernet or Wi-Fi interface (e.g., `enp3s0`, `wlp4s0`, `ens18`).

### Parameters (4 settings × 3 scopes = 12)

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `arp_ignore` | `1` | Only respond to ARP if the target IP is configured on the receiving interface |
| `arp_announce` | `2` | Always use the best local source address in ARP — prevents IP leaks |
| `drop_gratuitous_arp` | `1` | Drop unsolicited ARP announcements — belt-and-suspenders with nftables |
| `drop_unicast_in_l2_multicast` | `1` | Drop unicast IP packets in L2 multicast frames — anti-firewall-bypass |

### Config

```bash
# === IPv4: ARP (Triple Scope: all / default / <YOUR-INTERFACE>) ===
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.<YOUR-INTERFACE>.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.<YOUR-INTERFACE>.arp_announce = 2
net.ipv4.conf.all.drop_gratuitous_arp = 1
net.ipv4.conf.default.drop_gratuitous_arp = 1
net.ipv4.conf.<YOUR-INTERFACE>.drop_gratuitous_arp = 1
net.ipv4.conf.all.drop_unicast_in_l2_multicast = 1
net.ipv4.conf.default.drop_unicast_in_l2_multicast = 1
net.ipv4.conf.<YOUR-INTERFACE>.drop_unicast_in_l2_multicast = 1
```

> Replace `<YOUR-INTERFACE>` with your actual interface name (e.g., `enp3s0`).

### What breaks

Nothing for normal desktop usage. Gratuitous ARP drop may briefly delay detection of a changed gateway MAC — but your static ARP entry ([Doc 04](04-nftables-arp-hardening.md)) handles this.

---

## 9. Network: IGMP/Multicast (1 parameter)

### What

Suppresses IGMP membership reports for link-local multicast groups.

### Why

IGMP reports tell your router which multicast groups your machine has joined. On a LAN, this reveals your presence and activity to any device capturing multicast traffic. Disabling link-local reports makes your machine invisible to multicast group enumeration.

### Parameter

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `igmp_link_local_mcast_reports` | `0` | Suppresses IGMP reports for 224.0.0.x groups |

### Config

```bash
# === IPv4: IGMP/Multicast ===
net.ipv4.igmp_link_local_mcast_reports = 0
```

---

## 10. Network: IPv6 (11 parameters)

### What

Parameters that disable or harden IPv6 on your system.

### Why

If you don't use IPv6 (most home networks are IPv4-only), every IPv6 feature is pure attack surface:
- **Router Advertisements (RA)**: A rogue device can advertise itself as your IPv6 router, redirecting traffic.
- **SLAAC autoconf**: Automatically configures an IPv6 address from RA — leaks your MAC address in the interface identifier.
- **DAD (Duplicate Address Detection)**: Transmits on the network during address setup — reveals your presence.
- **Router solicitations**: Actively asks for IPv6 routers — announces your existence.

### Decision

```
IPv6 disable strategy
├── NO VPN (or VPN without killswitch):
│   ├── Set: net.ipv6.conf.all.disable_ipv6 = 1
│   └── This disables IPv6 on ALL interfaces — simplest approach
│
├── WITH VPN (WireGuard-based with killswitch):
│   ├── Set: net.ipv6.conf.default.disable_ipv6 = 1 (NOT all.disable_ipv6!)
│   ├── Why: Some VPN killswitch implementations create interfaces with IPv6
│   │        blackhole routes. all.disable_ipv6=1 breaks these routes.
│   └── Then: Explicitly harden your physical interface (see below)
│
└── IPv6 REQUIRED (e.g., IPv6-only ISP):
    ├── Do NOT disable IPv6
    ├── Still set: accept_redirects=0, accept_source_route=0
    └── Consider: privacy extensions (use_tempaddr=2)
```

### Parameters

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `conf.default.disable_ipv6` | `1` | Disables IPv6 on newly created interfaces |
| `conf.<YOUR-INTERFACE>.autoconf` | `0` | No SLAAC auto-configuration on your NIC |
| `conf.<YOUR-INTERFACE>.accept_ra_defrtr` | `0` | Ignore default router from RA |
| `conf.<YOUR-INTERFACE>.accept_ra_pinfo` | `0` | Ignore prefix information from RA |
| `conf.<YOUR-INTERFACE>.accept_ra_rtr_pref` | `0` | Ignore router preference from RA |
| `conf.<YOUR-INTERFACE>.dad_transmits` | `0` | No Duplicate Address Detection (zero transmissions) |
| `conf.<YOUR-INTERFACE>.router_solicitations` | `0` | No router solicitations (zero transmissions) |
| `conf.all.accept_redirects` | `0` | Ignore IPv6 ICMP redirects |
| `conf.default.accept_redirects` | `0` | *(same, for new interfaces)* |
| `conf.all.accept_source_route` | `0` | Ignore IPv6 source-routed packets |
| `conf.default.accept_source_route` | `0` | *(same, for new interfaces)* |

### Config (VPN users — use `default` not `all`)

```bash
# === IPv6 ===
# Disable for new interfaces (NOT all — VPN killswitch may need IPv6)
net.ipv6.conf.default.disable_ipv6 = 1

# Harden your physical interface explicitly
net.ipv6.conf.<YOUR-INTERFACE>.autoconf = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_defrtr = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_pinfo = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_rtr_pref = 0
net.ipv6.conf.<YOUR-INTERFACE>.dad_transmits = 0
net.ipv6.conf.<YOUR-INTERFACE>.router_solicitations = 0

# Global: disable redirects and source routing
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
```

### Config (no VPN — simpler)

If you don't use a VPN, you can disable IPv6 globally:

```bash
# === IPv6 (no VPN) ===
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Still set redirects/source_route as defense-in-depth
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
```

> Replace `<YOUR-INTERFACE>` with your interface name (`ip link show`).

---

## 11. User Namespaces — Limited, Not Disabled

### What

Limit the maximum number of unprivileged user namespaces to prevent kernel exploit abuse while keeping desktop sandboxing functional.

### Why

`user.max_user_namespaces = 0` is frequently recommended in hardening guides. **Do not set it to 0** — it breaks Flatpak and browser sandboxing. But the Fedora default (~255,000) is far too permissive. Nearly every Linux kernel privilege-escalation exploit from 2023–2026 requires `CLONE_NEWUSER` to create unprivileged user namespaces. Limiting the count to 256 allows sandboxing while preventing the mass namespace creation that exploits rely on.

### Parameter

| Parameter | Value | What it does | Threat mitigated |
|-----------|-------|-------------|------------------|
| `user.max_user_namespaces` | `256` | Limits unprivileged user namespace creation | Kernel LPE exploits requiring mass CLONE_NEWUSER |

### Decision

```
User Namespaces (user.max_user_namespaces)
├── Set 256 (recommended):
│   ├── Sufficient for Flatpak (bubblewrap), Firefox, and Chromium sandboxes
│   ├── Blocks exploit techniques that create hundreds/thousands of namespaces
│   └── Stored in separate file: /etc/sysctl.d/99-userns.conf
│
├── Set 0 (DO NOT):
│   ├── Flatpak apps fail to launch
│   ├── Firefox loses Seccomp sandbox (runs UNSANDBOXED — worse security)
│   └── Chromium refuses to start entirely
│
├── Leave default (~255,000) (NOT recommended):
│   └── Effectively unlimited — no protection against namespace-based exploits
│
├── What breaks at 256: Nothing in normal desktop use
│   ├── If a specific app fails: increase to 512 or 1024
│   └── Test: launch Flatpak apps + Firefox + Chromium after applying
│
└── Additional mitigations (complement, don't replace):
    ├── ptrace_scope = 2 (restrict debugging)
    ├── unprivileged_bpf_disabled = 2 (disable BPF)
    └── io_uring_disabled = 2 (disable io_uring)
```

### Config (separate file)

```bash
# File: /etc/sysctl.d/99-userns.conf
user.max_user_namespaces = 256
```

> **Separate file** because this value may need adjustment if new software requires more namespaces. Keeping it out of `99-hardening.conf` makes it easy to find and modify.

---

## 🔧 Applying Changes

### Step 1: Find your interface name

```bash
$ ip link show
# Look for your primary NIC (e.g., enp3s0, ens18, wlp4s0)
# Ignore lo (loopback) and any VPN/virtual interfaces
```

### Step 2: Create the main config file

Replace `<YOUR-INTERFACE>` with your actual interface name throughout:

```bash
$ sudo tee /etc/sysctl.d/99-hardening.conf << 'SYSCTL'
# === Kernel Security ===
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 2
kernel.sysrq = 0
kernel.kexec_load_disabled = 1
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 2
kernel.perf_event_paranoid = 3
kernel.randomize_va_space = 2
dev.tty.ldisc_autoload = 0
dev.tty.legacy_tiocsti = 0
vm.unprivileged_userfaultfd = 0
vm.mmap_min_addr = 65536
kernel.io_uring_disabled = 2
vm.mmap_rnd_bits = 32
net.core.bpf_jit_harden = 2

# === Filesystem ===
fs.suid_dumpable = 0
fs.protected_regular = 2
fs.protected_fifos = 2
fs.protected_hardlinks = 1
fs.protected_symlinks = 1

# === IPv4: Routing Protection ===
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2

# === IPv4: ICMP ===
net.ipv4.icmp_echo_ignore_all = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# === IPv4: TCP ===
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1

# === IPv4: ARP (Triple Scope: all / default / <YOUR-INTERFACE>) ===
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.<YOUR-INTERFACE>.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.<YOUR-INTERFACE>.arp_announce = 2
net.ipv4.conf.all.drop_gratuitous_arp = 1
net.ipv4.conf.default.drop_gratuitous_arp = 1
net.ipv4.conf.<YOUR-INTERFACE>.drop_gratuitous_arp = 1
net.ipv4.conf.all.drop_unicast_in_l2_multicast = 1
net.ipv4.conf.default.drop_unicast_in_l2_multicast = 1
net.ipv4.conf.<YOUR-INTERFACE>.drop_unicast_in_l2_multicast = 1

# === IPv4: IGMP/Multicast ===
net.ipv4.igmp_link_local_mcast_reports = 0

# === IPv6 ===
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.<YOUR-INTERFACE>.autoconf = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_defrtr = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_pinfo = 0
net.ipv6.conf.<YOUR-INTERFACE>.accept_ra_rtr_pref = 0
net.ipv6.conf.<YOUR-INTERFACE>.dad_transmits = 0
net.ipv6.conf.<YOUR-INTERFACE>.router_solicitations = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
SYSCTL
```

> **Important:** Edit the file after creation to replace all `<YOUR-INTERFACE>` with your actual interface name. The `sudo tee` heredoc cannot use variables inside single-quoted delimiters.

### Step 3: Create the stability config

```bash
$ sudo tee /etc/sysctl.d/99-audit-fixes.conf << 'AUDIT'
kernel.oops_limit = 100
kernel.warn_limit = 0
AUDIT
```

### Step 4: Create the user namespace config

```bash
$ sudo tee /etc/sysctl.d/99-userns.conf << 'USERNS'
user.max_user_namespaces = 256
USERNS
```

### Step 5: Apply all configs

```bash
$ sudo sysctl -p /etc/sysctl.d/99-hardening.conf
$ sudo sysctl -p /etc/sysctl.d/99-audit-fixes.conf
$ sudo sysctl -p /etc/sysctl.d/99-userns.conf
```

> No reboot required. Changes take effect immediately.

---

## ✅ Complete Verification

```bash
# 1. Kernel Security:
$ sysctl kernel.kptr_restrict kernel.yama.ptrace_scope kernel.sysrq \
  kernel.kexec_load_disabled kernel.dmesg_restrict kernel.unprivileged_bpf_disabled \
  kernel.perf_event_paranoid kernel.randomize_va_space dev.tty.ldisc_autoload \
  dev.tty.legacy_tiocsti vm.unprivileged_userfaultfd vm.mmap_min_addr \
  kernel.io_uring_disabled vm.mmap_rnd_bits net.core.bpf_jit_harden
# Expected: 2 2 0 1 1 2 3 2 0 0 0 65536 2 32 2

# 2. Kernel Stability:
$ sysctl kernel.oops_limit kernel.warn_limit
# Expected: 100 0

# 3. Filesystem:
$ sysctl fs.suid_dumpable fs.protected_regular fs.protected_fifos \
  fs.protected_hardlinks fs.protected_symlinks
# Expected: 0 2 2 1 1

# 4. ICMP:
$ sysctl net.ipv4.icmp_echo_ignore_all
# Expected: 1

# 5. ARP (triple scope — replace <YOUR-INTERFACE>):
$ sysctl net.ipv4.conf.all.arp_ignore net.ipv4.conf.default.arp_ignore \
  net.ipv4.conf.<YOUR-INTERFACE>.arp_ignore
# Expected: 1 1 1

$ sysctl net.ipv4.conf.all.drop_gratuitous_arp net.ipv4.conf.default.drop_gratuitous_arp \
  net.ipv4.conf.<YOUR-INTERFACE>.drop_gratuitous_arp
# Expected: 1 1 1

# 6. IPv6 on your interface:
$ sysctl net.ipv6.conf.<YOUR-INTERFACE>.autoconf \
  net.ipv6.conf.<YOUR-INTERFACE>.dad_transmits \
  net.ipv6.conf.<YOUR-INTERFACE>.router_solicitations
# Expected: 0 0 0

# 7. io_uring disabled:
$ sysctl kernel.io_uring_disabled
# Expected: 2

# 8. ASLR entropy:
$ sysctl vm.mmap_rnd_bits
# Expected: 32

# 9. Total parameter count:
$ grep -c "^[a-z]" /etc/sysctl.d/99-hardening.conf
# Expected: 61

$ grep -c "^[a-z]" /etc/sysctl.d/99-audit-fixes.conf
# Expected: 2

$ grep -c "^[a-z]" /etc/sysctl.d/99-userns.conf
# Expected: 1

# Total: 64 (61 + 2 + 1)
```

---

## ⚠️ Known Tradeoffs

| Parameter | Cost | Benefit |
|-----------|------|---------|
| `kptr_restrict = 2` | Kernel debugging harder (even for root) | Prevents kernel ASLR bypass |
| `perf_event_paranoid = 3` | `perf`/flamegraph requires sudo | Blocks profiling side channels |
| `sysrq = 0` | No emergency keyboard shortcuts | Prevents physical keyboard attacks |
| `icmp_echo_ignore_all = 1` | System invisible to ping | Network stealth |
| `rp_filter = 2` (loose) | Weaker than strict (1) against IP spoofing | Required for WireGuard VPN compatibility |
| `default.disable_ipv6 = 1` (not `all`) | IPv6 not globally disabled | VPN killswitch compatibility |
| `oops_limit = 100` | Not as aggressive as 1 | Tolerates occasional GPU driver oops |
| User namespaces limited to 256 | Kernel exploit building block (but 256 allows sandboxing) | Blocks mass-namespace exploit techniques |
| `io_uring_disabled = 2` | None for desktop | Eliminates dozens of kernel CVE vectors |
| `mmap_rnd_bits = 32` | None | Maximum ASLR entropy (16× harder than default 28) |

---

## Notes

- **`kexec_load_disabled = 1`**: This is a one-way toggle — once set to 1, it cannot be changed back to 0 without rebooting. The sysctl file re-applies it on every boot (kernel default is 0).
- **`unprivileged_bpf_disabled = 2`**: Value `2` means "permanently disabled" — cannot be re-enabled without reboot. This is stronger than `1` (which could be toggled back by root).
- **Many of these are already Fedora defaults** (symlink protection, ASLR, etc.). Setting them explicitly ensures they remain active regardless of future distribution changes — defense-in-depth.
- **VPN users:** Check the `rp_filter` and `disable_ipv6` decision trees carefully. Wrong values will break your VPN connection.
- **No VPN?** Consider using `rp_filter = 1` (strict) and `all.disable_ipv6 = 1` for stronger protection.

---

*Previous: [01 — Kernel Boot Parameters](01-kernel-boot-params.md)*
*Next: [03 — Firewall](03-firewall.md) — firewalld with nftables backend, DROP default policy*
