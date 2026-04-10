# 🔒 LUKS2 Full Disk Encryption

> Full disk encryption with LUKS2, Argon2id, and hardened mount options.
> Applies to: Fedora Workstation 43+ | All hardware with NVMe/SSD/HDD
> No reboot required for mount option changes (remount). LUKS setup at install time.

---

## Overview

LUKS2 encrypts your entire root and home filesystem. Without the passphrase, the disk contents are indistinguishable from random data. Only `/boot` and `/boot/efi` remain unencrypted (required for GRUB/Shim to load the kernel — Secure Boot protects their integrity).

### Why

| Threat | Protection |
|--------|-----------|
| Stolen/lost device | Disk contents unreadable without passphrase |
| Evil maid (boot tampering) | Secure Boot protects boot chain; LUKS protects data at rest |
| Cold boot (RAM extraction) | Encryption keys derived from passphrase at boot — not stored on disk |
| Forensic disk imaging | Image is encrypted — useless without passphrase |
| Second-hand sale | Disk wipe is cryptographic — change passphrase or destroy header |

### What you get

| Component | Configuration |
|-----------|--------------|
| Encryption | LUKS2, AES-XTS-plain64, 512-bit key |
| Key derivation | Argon2id (memory-hard — resists GPU brute-force) |
| Filesystem | Btrfs with root + home subvolumes |
| `/tmp` | tmpfs with `noexec` — blocks malware execution |
| `/dev/shm` | tmpfs with `noexec` — blocks exploit pivot |
| `/home` | `nosuid,nodev` — blocks privilege escalation |
| Swap | zram (RAM-only — nothing written to disk) |

---

## 1. Disk Layout

A typical Fedora LUKS2 installation:

```
<YOUR-DISK> (system disk)
├─ <part1>   600M  vfat FAT32  → /boot/efi (EFI System Partition)
├─ <part2>   1G    ext4        → /boot
└─ <part3>   rest  LUKS2
   └─ luks-<YOUR-LUKS-UUID>  btrfs
      ├─ subvol=root         → /
      ├─ subvol=home         → /home
      ├─ .snapshots          → Snapper root snapshots (→ Doc 20)
      └─ home/.snapshots     → Snapper home snapshots (→ Doc 20)

zram0 (compressed swap in RAM)
```

| Partition | Filesystem | Mount | Encrypted |
|-----------|-----------|-------|-----------|
| EFI System Partition | vfat FAT32 | `/boot/efi` | No (EFI requirement) |
| Boot partition | ext4 | `/boot` | No (GRUB requirement) |
| Root partition | LUKS2 → Btrfs | `/` and `/home` | **Yes** |
| zram | swap | [SWAP] | N/A (RAM-only — gone on poweroff) |

> **Why /boot is unencrypted**: GRUB must read the kernel and initramfs before LUKS can be unlocked. Secure Boot verifies the integrity of `/boot` contents via signatures — encryption is not needed for integrity, and Secure Boot provides it.

---

## 2. LUKS2 Encryption Parameters

### Verify your LUKS setup

```bash
# Find your LUKS partition:
lsblk -f | grep crypto_LUKS
# Note the device path (e.g., /dev/nvme0n1p3 or /dev/sda3)

# Dump LUKS header:
sudo cryptsetup luksDump /dev/<YOUR-LUKS-PARTITION>
```

### Recommended parameters

| Parameter | Recommended | Why |
|-----------|-------------|-----|
| **Version** | LUKS2 | Supports Argon2id, authenticated encryption, larger headers |
| **Cipher** | `aes-xts-plain64` | AES-256 in XTS mode — industry standard for disk encryption |
| **Key size** | 512 bits | XTS uses two 256-bit keys (one for encryption, one for tweak) |
| **PBKDF** | `argon2id` | Memory-hard KDF — resists GPU/ASIC brute-force |
| **Memory** | 1048576 KB (1 GB) | Each brute-force attempt needs 1 GB RAM |
| **Threads** | 4 | Parallel threads for key derivation |

### Why Argon2id matters

| KDF | Protection |
|-----|-----------|
| PBKDF2 | CPU-intensive only — vulnerable to GPU/ASIC brute-force |
| **Argon2id** | **Memory-hard (1 GB/attempt) + CPU-intensive** — GPU/ASIC attacks impractical |

Each brute-force attempt requires 1 GB RAM and 4 CPU threads. An attacker with a GPU cluster cannot simply run millions of attempts in parallel because each attempt requires 1 GB of memory.

### Decision

```
LUKS2 key derivation
├── Argon2id (recommended — Fedora 38+ default):
│   ├── Memory: 1048576 (1 GB) — higher = more brute-force resistant
│   ├── Threads: 4 — match your available CPU cores
│   ├── Time cost: auto-tuned by cryptsetup (targets ~2 second unlock)
│   └── Best protection against all brute-force hardware
│
├── PBKDF2 (legacy — older LUKS1 or early LUKS2):
│   ├── Only CPU-intensive — GPUs can parallelize attacks efficiently
│   ├── Upgrade: sudo cryptsetup luksConvertKey /dev/<YOUR-LUKS-PARTITION> \
│   │     --pbkdf argon2id --pbkdf-memory 1048576 --pbkdf-parallel 4
│   └── WARNING: ensure you have a backup passphrase before converting
│
└── Passphrase strength:
    ├── Argon2id makes each guess expensive — but doesn't fix weak passwords
    ├── Minimum: 20+ characters, mix of words/numbers/symbols
    ├── Best: random passphrase from a password manager
    └── The entire security of your disk rests on this passphrase
```

---

## 3. TRIM/Discard

### What

TRIM tells the SSD which blocks are no longer in use, allowing the SSD to optimize writes internally. Without TRIM, SSD performance degrades over time.

### The trade-off

| Aspect | With Discard | Without Discard |
|--------|-------------|----------------|
| SSD performance | Optimal — SSD can garbage-collect | Degrades over time |
| SSD lifespan | Extended — fewer write amplification | Reduced |
| Security | Minor leak — attacker can see which sectors are used vs free | Maximum — encrypted volume looks uniformly random |

### Decision

```
TRIM/Discard on LUKS
├── Enable discard (recommended for most users):
│   ├── SSD performance and lifespan benefit is significant
│   ├── Information leak is minimal (attacker sees used/free ratio, not content)
│   ├── Full disk encryption already hides all file content
│   └── How: add "discard" to /etc/crypttab options
│
└── Disable discard (maximum security):
    ├── Only if: you face state-level adversaries who might analyze
    │   disk usage patterns (journalists, activists in hostile states)
    ├── Cost: SSD performance degradation over months/years
    └── How: remove "discard" from /etc/crypttab
```

### Configuration

In `/etc/crypttab`:

```
luks-<YOUR-LUKS-UUID> UUID=<YOUR-LUKS-UUID> none discard
```

| Field | Value | Meaning |
|-------|-------|---------|
| Name | `luks-<YOUR-LUKS-UUID>` | Device mapper name |
| UUID | `<YOUR-LUKS-UUID>` | LUKS partition UUID (from `blkid`) |
| Key file | `none` | No key file — passphrase prompt at boot |
| Options | `discard` | Pass TRIM commands through to SSD |

---

## 4. Mount Hardening

### 4.1 /tmp — noexec

Malware commonly writes to `/tmp` and executes from there. `noexec` blocks this.

**Fedora default** (systemd tmpfiles): `/tmp` is already a tmpfs. Add `noexec`:

In `/etc/fstab`:

```
tmpfs /tmp tmpfs defaults,nosuid,nodev,noexec,size=4G 0 0
```

| Option | Why |
|--------|-----|
| `nosuid` | No SUID binaries in /tmp |
| `nodev` | No device nodes in /tmp |
| `noexec` | **No executable files** — blocks malware execution from /tmp |
| `size=4G` | Limit tmpfs to 4 GB (adjust based on your RAM) |

### 4.2 /dev/shm — noexec

`/dev/shm` is the classic fallback when `/tmp` has `noexec`. Both must be hardened.

In `/etc/fstab`:

```
tmpfs /dev/shm tmpfs defaults,nosuid,nodev,noexec 0 0
```

> **Compatibility**: Chromium-based apps (Signal, VS Code) use `/dev/shm` for shared memory (`mmap` with `PROT_READ|PROT_WRITE`), not for code execution. `noexec` only blocks direct binary execution and `mmap` with `PROT_EXEC` — shared memory works normally. Flatpak apps have their own `/dev/shm` inside the sandbox.

### 4.3 /home — nosuid,nodev

No legitimate program creates SUID binaries or device nodes in `/home`.

In `/etc/fstab`, add `nosuid,nodev` to your `/home` mount:

```
UUID=<YOUR-BTRFS-UUID> /home btrfs nosuid,nodev,subvol=home,compress=zstd:1 0 0
```

| Option | Why |
|--------|-----|
| `nosuid` | Blocks privilege escalation via crafted SUID binaries in home |
| `nodev` | Blocks fake device nodes as attack vectors |

> **Not noexec**: `/home` cannot have `noexec` — you need to run programs from your home directory (scripts, AppImages, development builds).

### Verify mount options

```bash
# /tmp noexec:
findmnt /tmp -o OPTIONS | grep noexec
# Expected: noexec present

# /dev/shm noexec:
findmnt /dev/shm -o OPTIONS | grep noexec
# Expected: noexec present

# /home nosuid,nodev:
findmnt /home -o OPTIONS | grep -o "nosuid,nodev"
# Expected: nosuid,nodev
```

### Apply without reboot

```bash
sudo mount -o remount /tmp
sudo mount -o remount /dev/shm
sudo mount -o remount /home
```

---

## 5. Btrfs Configuration

### Mount options

```bash
findmnt -t btrfs -o TARGET,OPTIONS
```

| Option | What it does |
|--------|-------------|
| `compress=zstd:1` | Transparent compression (Zstandard level 1 — fast) |
| `ssd` | SSD optimizations enabled |
| `discard=async` | Asynchronous TRIM (more performant than synchronous) |
| `space_cache=v2` | Free-space cache v2 (faster allocation) |
| `seclabel` | SELinux labels supported on filesystem |
| `relatime` | Update access time only on modification (performance) |

### Subvolumes

```bash
sudo btrfs subvolume list /
# Expected: root, home, .snapshots, home/.snapshots
```

| Subvolume | Mount | Purpose |
|-----------|-------|---------|
| `root` | `/` | System root filesystem |
| `home` | `/home` | User home directories |
| `.snapshots` | (under /) | Snapper root snapshots (→ Doc 20) |
| `home/.snapshots` | (under /home) | Snapper home snapshots (→ Doc 20) |

> **Fedora naming**: Fedora uses `root` and `home` as subvolume names (not `@` and `@home` like Ubuntu/Arch).

---

## 6. Swap: zram (RAM-Only)

```bash
swapon --show
# Expected: only zram0 listed (TYPE=partition, no disk swap)
```

| Aspect | Detail |
|--------|--------|
| Type | Compressed RAM swap (no disk swap) |
| Encryption | N/A — exists only in RAM, gone on poweroff |
| Persistence | None — all swap content lost on shutdown |
| Package | `zram-generator` (Fedora default) |

### Why no disk swap

Disk swap writes RAM contents (passwords, keys, session tokens) to an unencrypted area on disk. With zram:
- **Nothing is written to disk** — all swap is compressed in RAM
- **Poweroff = gone** — no forensic recovery of swap content
- **No hibernation trade-off** — hibernation requires disk swap (not configured on a hardened system)

### Decision

```
Swap strategy
├── zram only (recommended — Fedora default):
│   ├── No RAM content written to disk
│   ├── Compressed — effective size is 2-3x physical allocation
│   └── No hibernation support (acceptable trade-off)
│
├── Encrypted disk swap (if you need hibernation):
│   ├── Add a LUKS-encrypted swap partition
│   ├── Required for suspend-to-disk (hibernation)
│   └── Configuration: beyond scope of this doc
│
└── No swap at all (not recommended):
    ├── OOM killer activates sooner under memory pressure
    └── Some applications expect swap to exist
```

---

## 7. USB Write-Through

### Why

Linux uses write-back caching for USB drives — the copy dialog shows "done" while GBs of data are still in RAM cache. Removing the drive before the cache flushes causes data loss.

### Fix: udev rule

Create `/etc/udev/rules.d/99-usb-write-through.rules`:

```
ACTION=="add|change", KERNEL=="sd[a-z]", SUBSYSTEM=="block", ATTRS{removable}=="1", RUN+="/bin/sh -c 'echo write through > /sys/block/%k/queue/write_cache'"
```

| Parameter | Why |
|-----------|-----|
| `ATTRS{removable}=="1"` | Only removable media (USB drives) — not internal disks |
| `write through` | Data written directly to USB — no RAM cache |

### Trade-off

| Aspect | Write-back (default) | Write-through (fix) |
|--------|---------------------|---------------------|
| Copy speed (displayed) | Fast (RAM speed) | Slower (actual USB speed) |
| "Safely remove" | Minutes (cache flush) | Instant |
| Data integrity | Risk if unplugged early | Safe |
| Internal NVMe/SSD | Not affected | Not affected |

### Verify

```bash
# Plug in a USB drive, then:
cat /sys/block/sda/queue/write_cache
# Expected: write through
```

---

## 8. LUKS Header Backup

### Why this is critical

The LUKS header contains the encrypted master key and keyslot metadata. If the header is corrupted (disk failure, accidental overwrite), **all data is permanently lost** — even if you know the passphrase.

### How to backup

```bash
# Backup LUKS header to a file:
sudo cryptsetup luksHeaderBackup /dev/<YOUR-LUKS-PARTITION> \
  --header-backup-file luks-header-backup.img

# Verify the backup:
sudo cryptsetup luksDump luks-header-backup.img
# Expected: same header info as the live partition
```

### Where to store the backup

```
LUKS header backup storage
├── Encrypted USB drive (recommended):
│   ├── Store on a separate, encrypted USB stick
│   ├── Keep in a physically secure location (safe, bank vault)
│   └── Not the same USB stick you use daily
│
├── Password manager:
│   ├── Some password managers support file attachments
│   ├── Header backup is small (~16 MB)
│   └── Ensure the password manager itself is backed up
│
└── NEVER store:
    ├── On the same encrypted disk (defeats the purpose)
    ├── In cloud storage unencrypted (header = access to your disk)
    └── On an unencrypted USB drive (physical theft risk)
```

### Recovery key (additional keyslot)

Add a separate recovery passphrase to a second LUKS keyslot:

```bash
# Add recovery key to slot 1:
sudo cryptsetup luksAddKey /dev/<YOUR-LUKS-PARTITION>
# Enter existing passphrase, then enter new recovery passphrase

# Verify both keyslots exist:
sudo cryptsetup luksDump /dev/<YOUR-LUKS-PARTITION> | grep "Keyslot"
# Expected: Keyslot 0: ENABLED, Keyslot 1: ENABLED
```

> **Store the recovery passphrase separately** from your daily passphrase — different location, different medium. If you forget your daily passphrase, the recovery key is your only way in.

---

## 9. Complete Verification

```bash
# 1. LUKS2 active:
sudo cryptsetup status luks-<YOUR-LUKS-UUID>
# Expected: active, cipher aes-xts-plain64, keysize 512

# 2. Argon2id as PBKDF:
sudo cryptsetup luksDump /dev/<YOUR-LUKS-PARTITION> | grep -i pbkdf
# Expected: argon2id

# 3. Btrfs subvolumes:
sudo btrfs subvolume list / | grep -E "root|home"
# Expected: root and home subvolumes listed

# 4. /tmp noexec:
findmnt /tmp -o OPTIONS -n | grep -o noexec
# Expected: noexec

# 5. /dev/shm noexec:
findmnt /dev/shm -o OPTIONS -n | grep -o noexec
# Expected: noexec

# 6. /home nosuid,nodev:
findmnt /home -o OPTIONS -n | grep -o "nosuid"
# Expected: nosuid

# 7. No disk swap:
swapon --show
# Expected: only zram (no /dev/nvme* or /dev/sd*)

# 8. Discard active:
sudo cryptsetup status luks-<YOUR-LUKS-UUID> | grep -i flags
# Expected: discards

# 9. SELinux labels on Btrfs:
findmnt / -o OPTIONS -n | grep -o seclabel
# Expected: seclabel

# 10. LUKS header backup exists (check your backup location):
ls -la luks-header-backup.img
# Expected: file exists, ~16 MB
```

---

## Important Notes

- **Boot partitions are unencrypted**: `/boot` and `/boot/efi` must be unencrypted for GRUB/Shim. Secure Boot protects their integrity via signature verification — encryption is not needed when integrity is guaranteed.
- **Passphrase is everything**: Argon2id makes each brute-force guess expensive, but a weak passphrase is still a weak passphrase. Use 20+ characters.
- **LUKS header backup is essential**: Without a header backup, disk corruption = total data loss. Back up the header to a separate, secure location immediately after installation.
- **TRIM/Discard trade-off**: Enabled for SSD performance. The minimal information leak (attacker sees used/free sector ratio) is acceptable for most threat models.
- **zram = no disk swap**: RAM contents are never written to disk. Hibernation is not supported (acceptable on a desktop — use suspend-to-RAM instead).
- **Single LUKS partition**: Root and home share one LUKS partition (as Btrfs subvolumes). One passphrase decrypts everything. Separate LUKS partitions with different passphrases are possible but unnecessary for single-user desktops.
- **noexec on /tmp and /dev/shm**: These two tmpfs mounts are the most common malware execution targets. Both must have `noexec` — hardening only one leaves the other as a fallback.
- **USB write-through**: Only affects removable media. Internal NVMe/SSD drives are unaffected and continue to use write-back caching for performance.

---

*Previous: [21 — Kernel Module Blacklisting](21-kernel-module-blacklisting.md)*
*Next: [23 — NetworkManager Hardening](23-networkmanager-hardening.md) — Static IP, MAC randomization, connectivity check disabled*
