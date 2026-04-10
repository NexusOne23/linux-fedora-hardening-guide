# 🦊 Firefox Hardening

> arkenfox user.js, fingerprinting protection (FPP), CRLite, DoH, uBlock Origin, WebRTC off, ECH.
> Applies to: Fedora Workstation 43+ | Firefox (system package)
> Changes apply after Firefox restart.

---

## Overview

Firefox hardened with **arkenfox user.js** — a community-maintained privacy/security template with ~1330 settings. Combined with a single extension (uBlock Origin) and custom overrides for stability.

### Security Architecture

```
Firefox Content Processes
  │
  ├─ Seccomp BPF Sandbox (syscall filter)
  ├─ Namespaces (PID, Net, User)
  ├─ arkenfox user.js (~1330 privacy/security settings)
  ├─ uBlock Origin (300k+ filters: ads, trackers, malware, phishing)
  ├─ CRLite mode 2 (local certificate revocation — no OCSP leak)
  ├─ Total Cookie Protection (isolated cookie jars per domain)
  ├─ FPP (Fingerprinting Protection — 20+ active targets)
  ├─ WebRTC disabled (no IP leak via STUN)
  ├─ ECH / Encrypted Client Hello (SNI encryption)
  ├─ DoH via Quad9 (DNS encryption, bootstrap IP direct)
  └─ VPN tunnel (if configured — see Doc 06)
```

---

## 1. arkenfox user.js

### What

A pre-configured `user.js` template that sets ~1330 Firefox preferences for privacy and security.

### Why

Firefox's defaults prioritize compatibility over privacy. arkenfox changes this:

| Area | What arkenfox does |
|------|-------------------|
| Telemetry | Completely disabled (Studies, Crash Reports, Normandy, etc.) |
| Fingerprinting | Fingerprinting Protection (FPP) or Resist Fingerprinting (RFP) |
| Tracking | Enhanced Tracking Protection on Strict (Total Cookie Protection) |
| Cookies | Third-party cookies blocked, isolated cookie jars per domain |
| Referrer | Restricted to same-origin |
| DOM APIs | Many privacy-invasive APIs disabled |
| Caching | Disk cache disabled — memory only |
| HTTPS | HTTPS-Only Mode enabled |
| TLS | 0-RTT disabled, safe negotiation enforced, cert pinning strict |
| CRLite | Mode 2 — local certificate revocation checking |
| Safe Browsing | Google Safe Browsing remote downloads disabled (privacy tradeoff) |

### Source

```
arkenfox user.js: https://github.com/arkenfox/user.js
```

---

## 2. Recommended user-overrides.js

arkenfox is deliberately strict — some settings cause website breakage. The `user-overrides.js` file customizes arkenfox for daily use. Place it in your Firefox profile directory alongside `user.js`.

### Profile path on Fedora

```bash
# Find your profile directory:
$ ls ~/.config/mozilla/firefox/
# Look for: xxxxxxxx.default-release
# Full path: ~/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release/
```

> **Fedora 43+ uses XDG path** (`~/.config/mozilla/firefox/`), not the legacy `~/.mozilla/firefox/`.

### File: `user-overrides.js`

```javascript
/*** USER OVERRIDES — arkenfox customization for daily use ***/

/*** [SECTION: SECURITY] ***/

/* OCSP hard-fail disabled — CRLite mode 2 handles revocation locally.
 * OCSP servers are dying (Let's Encrypt ended OCSP May 2025, DV certs since FF142).
 * Hard-fail caused page load failures when OCSP servers were unresponsive. */
user_pref("security.OCSP.require", false);

/* DNS-over-HTTPS via Quad9 (Switzerland, no-log, malware blocking)
 * TRR Mode 3 = DoH-only, no fallback to system DNS
 * Bootstrap IP: resolves DoH hostname without system DNS (defense-in-depth) */
user_pref("network.trr.mode", 3);
user_pref("network.trr.uri", "https://dns.quad9.net/dns-query");
user_pref("network.trr.custom_uri", "https://dns.quad9.net/dns-query");
user_pref("network.trr.bootstrapAddr", "9.9.9.9");

/* IPv6 DNS disabled — matches system-level IPv6 disable */
user_pref("network.dns.disableIPv6", true);

/* WebRTC DISABLED — prevents IP leak via STUN even with VPN */
user_pref("media.peerconnection.enabled", false);

/* HTTPS-Only Mode */
user_pref("dom.security.https_only_mode", true);

/* Encrypted Client Hello (ECH) — encrypt SNI where supported */
user_pref("network.dns.echconfig.enabled", true);
user_pref("network.dns.http3_echconfig.enabled", true);

/* Password manager, payment data, addresses — permanently disabled */
user_pref("signon.rememberSignons", false);
user_pref("extensions.formautofill.creditCards.enabled", false);
user_pref("extensions.formautofill.addresses.enabled", false);

/*** [SECTION: FINGERPRINTING] ***/

/* FPP instead of RFP — granular protection with less breakage.
 *
 * RFP (Resist Fingerprinting) is arkenfox's default — maximum protection
 * but causes significant website breakage (dark mode broken, timezone UTC,
 * window resizing, canvas errors).
 *
 * FPP (Fingerprinting Protection) provides the same protections but lets
 * you selectively disable the ones that cause breakage.
 *
 * Disabled targets (cause breakage):
 * - CSSPrefersColorScheme: allow real dark/light theme
 * - JSDateTimeUTC: allow real timezone
 * - RoundWindowSize: allow real window size
 * - CanvasRandomization: canvas noise causes image processing errors
 * - EfficientCanvasRandomization: same issue
 * - WebGLRandomization: WebGL noise causes graphics errors on upload
 * - Navigator*: real browser identity (VPN + uBlock protect sufficiently)
 * - KeyboardEvents: real keyboard layout
 * - SiteSpecificZoom: allow per-site zoom
 *
 * ACTIVE targets include: Font restriction, AudioContext, JSMath,
 * Screen spoofing, HW concurrency, MediaDevices, Touch points, etc. */
user_pref("privacy.resistFingerprinting", false);
user_pref("privacy.fingerprintingProtection", true);
user_pref("privacy.fingerprintingProtection.overrides", "+AllTargets,-CSSPrefersColorScheme,-JSDateTimeUTC,-RoundWindowSize,-CanvasRandomization,-EfficientCanvasRandomization,-WebGLRandomization,-NavigatorPlatform,-NavigatorOscpu,-NavigatorUserAgent,-HttpUserAgent,-NavigatorAppVersion,-KeyboardEvents,-SiteSpecificZoom");

/* Keep cookies on shutdown — stay logged in
 * Only v2 pref needed (v1 clearOnShutdown.cookies deprecated since FF128) */
user_pref("privacy.clearOnShutdown_v2.cookiesAndStorage", false);

/* WebGL enabled (needed by some sites — FPP protects separately) */
user_pref("webgl.disabled", false);

/* DRM enabled — for streaming services (Netflix, Spotify, Disney+) */
user_pref("media.eme.enabled", true);

/*** [SECTION: PERFORMANCE] ***/

/* Hardware video decoding (VA-API) — adjust for your GPU:
 * NVIDIA: requires libva-nvidia-driver
 * AMD/Intel: works out of the box with mesa */
user_pref("media.ffmpeg.vaapi.enabled", true);
user_pref("media.hardware-video-decoding.force-enabled", true);
user_pref("gfx.webrender.all", true);

/*** [SECTION: USABILITY] ***/

user_pref("general.smoothScroll", true);
user_pref("browser.formfill.enable", true);
user_pref("browser.startup.page", 1);
user_pref("browser.startup.homepage", "about:home");
user_pref("browser.newtabpage.enabled", true);
```

### Override Explanations

| Override | arkenfox default | Our value | Why |
|----------|-----------------|-----------|-----|
| `security.OCSP.require` | true | **false** | CRLite mode 2 handles revocation locally. OCSP servers are dying out. Hard-fail caused double-load issues. |
| `network.trr.mode` | 0 | **3** | DoH-only — no fallback to system DNS (maximum DNS leak prevention) |
| `network.trr.bootstrapAddr` | (empty) | **9.9.9.9** | Resolves DoH hostname directly — no system DNS needed |
| `network.dns.disableIPv6` | false | **true** | Matches system-level IPv6 disable |
| `media.peerconnection.enabled` | true | **false** | WebRTC completely off — prevents IP leak via STUN |
| `privacy.resistFingerprinting` | true | **false** | RFP disabled in favor of granular FPP |
| `privacy.fingerprintingProtection` | false | **true** | FPP — same protections as RFP, less breakage |
| `privacy.clearOnShutdown_v2.cookiesAndStorage` | true | **false** | Stay logged in across sessions |
| `webgl.disabled` | true | **false** | Some sites require WebGL — FPP protects separately |
| `media.eme.enabled` | false | **true** | DRM for streaming services |
| `signon.rememberSignons` | (commented) | **false** | Password manager disabled — use a dedicated password manager |

### Decision

```
RFP vs FPP (Fingerprinting Protection)
├── Use RFP if:    Maximum fingerprinting resistance, can tolerate breakage
│   └── Set: privacy.resistFingerprinting = true (arkenfox default)
│
├── Use FPP if:    Strong protection with less breakage (recommended)
│   ├── Set: privacy.resistFingerprinting = false
│   ├── Set: privacy.fingerprintingProtection = true
│   └── Set: overrides string (see above — disable breakage-causing targets)
│
└── Breakage examples with RFP:
    ├── Dark mode forced to light on many sites
    ├── Timezone shows UTC instead of local
    ├── Window size snapped to fixed dimensions
    └── Canvas/WebGL errors on image processing sites
```

---

## 3. Profile Isolation

### What

Use separate Firefox profiles to isolate different browsing contexts.

### Why

A dedicated profile for a specific service (e.g., streaming, social media) prevents:
- Cookie leakage between contexts
- Browsing history correlation
- Fingerprint linking across services

### How

```bash
# Open Firefox profile manager:
$ firefox -P

# Create a new profile, then launch it:
$ firefox -P "profile-name" --no-remote
```

Each profile gets its own `user.js`, `user-overrides.js`, and extensions — completely isolated from your main profile.

---

## 4. Extension: uBlock Origin

### What

The only recommended browser extension. uBlock Origin is the gold standard for content blocking.

### Why only one extension?

Every extension increases attack surface. uBlock Origin alone provides:
- Ad blocking
- Tracker blocking
- Malware domain blocking (with filter lists)
- Phishing URL blocking (with filter lists)
- LAN intrusion blocking (with filter list)
- Cookie notice blocking
- Cosmetic filtering

### Recommended Filter Lists

| Category | List |
|----------|------|
| **Built-in** | uBlock filters (Ads, Badware, Privacy, Quick fixes, Unbreak) |
| **Ads** | EasyList |
| **Privacy** | EasyPrivacy |
| **Privacy** | AdGuard/uBO – URL Tracking Protection |
| **Privacy** | Block Outsider Intrusion into LAN |
| **Malware** | Online Malicious URL Blocklist |
| **Malware** | Phishing URL Blocklist |
| **Multipurpose** | Peter Lowe's Ad and tracking server list |
| **Cookie** | EasyList – Cookie Notices |
| **Cookie** | uBlock filters – Cookie Notices |
| **Annoyances** | uBlock filters – Annoyances |
| **Regional** | Your regional EasyList (if available) |

### Key settings

- Auto-update filter lists: **On**
- Suspend network activity until all filter lists are loaded: **On**
- `no-csp-reports: * true` (block CSP violation reports to third parties)

---

## 5. DNS Path (Firefox)

```
Firefox → DoH (Quad9 9.9.9.9:443) → VPN tunnel (if active) → Internet
```

Firefox uses its **own DNS** — independent of systemd-resolved:
- `network.trr.mode=3`: DoH-only — no fallback to system DNS
- `network.trr.bootstrapAddr=9.9.9.9`: Resolves DoH hostname directly
- Resolver: Quad9 (Switzerland, no-log, DNSSEC-validating, malware blocking)

> System DNS (configured in [Doc 11](11-dns-ntp.md)) handles non-browser queries. Firefox DoH handles all browser queries. Two independent DNS resolvers — defense-in-depth.

---

## 🔧 Applying Changes

### Step 1: Download arkenfox

```bash
$ PROFILE_DIR="$HOME/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release"
$ cd "$PROFILE_DIR"

$ curl -LO https://raw.githubusercontent.com/arkenfox/user.js/master/user.js
$ curl -LO https://raw.githubusercontent.com/arkenfox/user.js/master/updater.sh
$ curl -LO https://raw.githubusercontent.com/arkenfox/user.js/master/prefsCleaner.sh
$ chmod +x updater.sh prefsCleaner.sh
```

### Step 2: Create user-overrides.js

Copy the recommended overrides from Section 2 into `$PROFILE_DIR/user-overrides.js`.

### Step 3: Run arkenfox updater

```bash
# Merges user-overrides.js into user.js automatically:
$ ./updater.sh -s
```

### Step 4: Clean deprecated prefs

**Close Firefox first!**

```bash
$ ./prefsCleaner.sh -s
```

### Step 5: Install uBlock Origin

Open Firefox → `about:addons` → Search "uBlock Origin" → Install → Configure filter lists per Section 4.

---

## ✅ Complete Verification

```bash
# 1. user.js exists:
$ wc -l ~/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release/user.js
# Expected: ~1300+ lines

# 2. user-overrides.js exists:
$ ls ~/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release/user-overrides.js
# Expected: File exists

# 3. Parrot check (arkenfox syntax validation):
$ grep "parrot.*SUCCESS" ~/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release/prefs.js
# Expected: "SUCCESS" parrot string present

# 4. uBlock Origin installed:
$ ls ~/.config/mozilla/firefox/<YOUR-PROFILE-ID>.default-release/extensions/ | grep uBlock
# Expected: uBlock0@raymondhill.net.xpi
```

**In Firefox `about:config` verify:**

| Preference | Expected value |
|-----------|---------------|
| `security.OCSP.require` | false |
| `security.pki.crlite_mode` | 2 |
| `network.trr.mode` | 3 |
| `network.trr.bootstrapAddr` | 9.9.9.9 |
| `network.dns.disableIPv6` | true |
| `media.peerconnection.enabled` | false |
| `dom.security.https_only_mode` | true |
| `network.dns.echconfig.enabled` | true |
| `privacy.fingerprintingProtection` | true |
| `privacy.resistFingerprinting` | false |
| `signon.rememberSignons` | false |

**In `about:support` verify:**
- Content Processes → Seccomp BPF: **true**

---

## ⚠️ Known Tradeoffs

| Configuration | Cost | Benefit |
|--------------|------|---------|
| arkenfox (~1330 settings) | Some website breakage (mitigated by overrides) | Comprehensive privacy/security baseline |
| FPP instead of RFP | Slightly less fingerprint resistance | Dark mode, timezone, window size work normally |
| WebRTC disabled | No in-browser video/audio calls (Jitsi, Meet) | No IP leak via STUN |
| DRM enabled | Widevine CDM loaded for streaming | Can watch Netflix, Spotify, Disney+ |
| OCSP hard-fail disabled | Soft-fail for non-CRLite certs | No page load failures from dead OCSP servers |
| Safe Browsing remote off | No Google remote download check | No telemetry to Google; uBlock compensates |
| Single extension only | No additional functionality | Minimal attack surface |
| DoH Mode 3 (no fallback) | If Quad9 is down, no browser DNS | Maximum DNS leak prevention |

---

## Notes

- **arkenfox updates:** Run `./updater.sh -s` in your profile directory. It automatically merges `user-overrides.js`. Then run `./prefsCleaner.sh -s` (Firefox must be closed).
- **Never edit `user.js` directly.** It gets overwritten on update. Always use `user-overrides.js`.
- **WebRTC disabled** means no video/audio calls in-browser (Jitsi, Google Meet, Zoom web). Enable temporarily in `about:config` if needed: `media.peerconnection.enabled = true`.
- **ECH (Encrypted Client Hello)** only works when the target site supports it AND DoH is active. It encrypts the SNI header — your ISP/VPN cannot see which domain you visit.
- **CRLite mode 2** checks certificate revocation locally using Mozilla's CRLite filters. More reliable than OCSP in the post-OCSP era (Let's Encrypt ended OCSP May 2025). OCSP remains as soft-fail backup.
- **Profile path on Fedora 43+:** `~/.config/mozilla/firefox/` (XDG), not `~/.mozilla/firefox/`.
- **FFmpeg codecs:** Fedora ships `ffmpeg-free` which lacks H.264, H.265, and AAC — most web videos won't play. Install the full FFmpeg from RPM Fusion:
  ```bash
  sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
  sudo dnf swap ffmpeg-free ffmpeg --allowerasing
  ```
  If you already use NVIDIA akmod, RPM Fusion is already enabled — just run the `swap` command. Without full FFmpeg, the VA-API hardware decoding prefs are useless for H.264 content.
- **VA-API hardware decoding:** NVIDIA requires `libva-nvidia-driver`. AMD and Intel work out of the box with mesa. The `force-enabled` pref is needed for NVIDIA; AMD/Intel can omit it.
- **Cookie security — 5 layers:** Total Cookie Protection (isolated jars) + Bounce Tracking Protection + URL Query Stripping + uBlock Origin + VPN tunnel.

---

*Previous: [15 — Intel ME Mitigation](15-intel-me-mitigation.md)*
*Next: [17 — Desktop Stack](17-desktop-stack.md) — GNOME/Wayland hardening, autoclose-xwayland, telemetry disabled*
