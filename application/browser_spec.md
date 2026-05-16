# BGP-X Browser Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification

---

## 1. Overview

BGP-X Browser is the native client for the `.bgpx` domain system and the BGP-X overlay network. It is the equivalent of Tor Browser for `.onion` — but with cross-domain support, latency-adaptive rendering for LoRa and satellite-GEO paths, and integration with all BGP-X routing capabilities.

BGP-X Browser:
- Handles `bgpx://` URLs natively
- Resolves `.bgpx` short names via Name Registry DHT
- Authenticates services via ServiceID (no CA)
- Routes all traffic through the BGP-X daemon (TUN interface for clearnet; SDK for `.bgpx`)
- Adapts rendering for LoRa-class and satellite-GEO-class latency paths
- Displays path status (latency class, domain composition)
- Enforces privacy defaults that exceed those of standard browsers
- Connects to the BGP-X daemon running on a local router (recommended) or locally on the device

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     BGP-X Browser Process                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Renderer Process (Chromium/Gecko)          │ │
│  │  - Standard web page rendering                              │ │
│  │  - JavaScript engine                                        │ │
│  │  - CSS layout                                               │ │
│  │  - bgpx:// custom protocol intercept                        │ │
│  └─────────────────────────────┬───────────────────────────────┘ │
│                                │ IPC                              │
│  ┌─────────────────────────────▼───────────────────────────────┐ │
│  │                   Main Process (Electron/Node.js)            │ │
│  │  - bgpx:// URL scheme handler                               │ │
│  │  - .bgpx name resolution                                    │ │
│  │  - ServiceID authentication                                 │ │
│  │  - Path status subscription                                 │ │
│  │  - Latency class detection and mode switching               │ │
│  │  - Privacy policy enforcement                               │ │
│  │  - Connection to daemon (router or local)                   │ │
│  └─────────────────────────────┬───────────────────────────────┘ │
│                                │ Unix socket / TCP                │
└────────────────────────────────┼─────────────────────────────────┘
                                 │
                    ┌────────────▼────────┐
                    │  BGP-X Daemon       │
                    │  (bgpx-node)        │
                    │  on router or       │
                    │  locally            │
                    └─────────────────────┘
```

### 2.1 Engine Choice

BGP-X Browser is built on one of:

**Option A: Chromium (Electron)**
- Base: Electron framework with Chromium rendering engine
- BGP-X networking layer: native Node.js module (napi-rs wrapping bgpx-sdk Rust)
- bgpx:// protocol: registered as Electron custom protocol
- Advantages: large existing Chromium extension ecosystem; familiar DevTools; GPU acceleration; cross-platform build tooling maturity

**Option B: Firefox/Gecko (modified)**
- Base: Firefox with modified networking backend
- BGP-X networking layer: Rust XPCOM component replacing Gecko networking for bgpx:// scheme
- bgpx:// protocol: registered Gecko URL scheme
- Advantages: Tor Browser codebase familiarity; open-source alignment; better privacy defaults in base browser

Both options produce the same user-visible behavior. The engine is an implementation decision not visible to users or service operators.

---

## 3. bgpx:// Protocol Handler

### 3.1 URL Processing

```
Input URL: bgpx://community-news.bgpx/article/1234?lang=es

Processing:
1. Extract host: "community-news.bgpx"
2. Classify: short name (3-63 chars + ".bgpx")
3. Resolve via SDK: name_registry.resolve("community-news") → ServiceID "a3f2b9..."
4. Canonical URL: bgpx://a3f2b9....bgpx/article/1234?lang=es
5. Address bar shows: community-news.bgpx/article/1234?lang=es (friendly form)
6. Construct BGP-X stream to service_id "a3f2b9..."
7. Open HTTP/2 session over BGP-X stream
8. Send HTTP/2 HEADERS: GET /article/1234?lang=es
```

### 3.2 Connection Establishment

```
bgpx_stream = sdk.connect_stream(
    destination = "bgpx://a3f2b9....bgpx",
    config = StreamConfig {
        require_bgpx_service = true,    // Enforce ServiceID match
        path_constraints = user_pool_config,
        isolated_path = true,           // Separate path per service
    }
)

// On stream established:
verify(bgpx_stream.remote_service_id == extracted_service_id)
// If verification fails: display BGPX_KEY_MISMATCH error; abort; never render content

// Subscribe to path status for this stream:
path_events = bgpx_stream.subscribe_path_events()
latency_class = path_events.current.latency_class
// Activate appropriate rendering strategy (Standard, Overlay, LoRa, Satellite-GEO)
```

### 3.3 Request Routing

```
For bgpx:// URL:
  → BGP-X SDK stream → BGP-X Daemon → overlay path → .bgpx service

For https:// URL (clearnet):
  → BGP-X Daemon TUN interface → overlay path → clearnet exit → destination
  (user's actual IP never seen by destination; exit node's IP is visible)

For http:// URL (insecure clearnet):
  → Same as https:// but browser displays HTTPS upgrade warning
  → User can configure: force all clearnet to HTTPS, warn, or allow

For blocked content (ads, trackers):
  → BGP-X Browser built-in filter list blocks request before sending to daemon
```

---

## 4. User Interface

### 4.1 Address Bar

```
┌──────────────────────────────────────────────────────────────────────┐
│  [🟢 .bgpx] community-news.bgpx/article/1234    [⏸ LoRa ~3s/res]  │
└──────────────────────────────────────────────────────────────────────┘
```

Address bar components:
- **Security badge** (leftmost): 🟢 `.bgpx` verified / 🌐 clearnet via overlay / 🔶 warning / 🔴 error
- **URL display**: for `.bgpx`, shows friendly form (short name or abbreviated ServiceID); click to expand to full `bgpx://a3f2b9....bgpx/...`
- **Latency indicator** (rightmost, when applicable):
  - `[⏸ LoRa ~3s/resource]` for LoRa-class paths
  - `[🛰 Satellite ~2s/resource]` for GEO-satellite-class paths (high latency clearnet or future satellite transport)
  - `[⚡ Overlay ~200ms]` for standard overlay paths
  - Absent for standard clearnet if overlay latency is negligible

Clicking the security badge opens the **Connection Info panel**:
```
┌─────────────────────────────────────────────────────┐
│  .bgpx Verified Connection                          │
│                                                     │
│  Service:     community-news.bgpx                   │
│  ServiceID:   a3f2b9c1...f0a1 (first/last 8 chars) │
│  Path type:   clearnet → LoRa mesh                  │
│  Hop count:   7                                     │
│  Latency:     ~3s per resource (LoRa path)          │
│                                                     │
│  [View full ServiceID] [Path details] [Settings]    │
└─────────────────────────────────────────────────────┘
```

### 4.2 Loading Progress for High-Latency Paths

On LoRa-class paths (1-5s RTT) and Satellite-GEO-class paths (600ms+ RTT), standard browser loading spinners are replaced with a resource progress indicator:

**LoRa Path Indicator:**
```
[████░░░░░░] Loading: 4/10 resources (28 KB / 80 KB)
Critical CSS: loaded ✓
Fonts: loaded ✓
HTML: loaded ✓
Scripts: loading (est. 4.2s)
Images: waiting
```

**Satellite-GEO Path Indicator:**
```
[██░░░░░░░░] Loading: 2/10 resources (12 KB / 80 KB)
Connection: Satellite-GEO latency (~1s RTT)
Critical CSS: loaded ✓
HTML: loaded ✓
Scripts: waiting
```

The indicator shows which resources have loaded and estimates time for remaining resources based on measured throughput. For LoRa paths, the UI also indicates if the stream is in a **Paused** state due to duty cycle limits.

### 4.3 New Tab Page — .bgpx Service Directory

BGP-X Browser new tab page shows:

```
┌─────────────────────────────────────────────────────────────────┐
│  BGP-X Browser                    [Search .bgpx services...]    │
│                                                                 │
│  🌐 Quick access                                               │
│  [Enter bgpx:// address or shortname]                         │
│                                                                 │
│  📡 Available via your connected islands                        │
│  lima-district-1: community-news, library, radio-station       │
│  coastal-mesh: weather-station, community-chat                 │
│                                                                 │
│  🌍 Public directory (clearnet reachable)                      │
│  [community-news.bgpx] Community news, Lima District 1        │
│  [privacy-wiki.bgpx]   Privacy and security wiki              │
│  [mesh-weather.bgpx]   Weather data from mesh sensors         │
│                                                                 │
│  [Browse all] [Add favorite] [Settings]                        │
└─────────────────────────────────────────────────────────────────┘
```

Services in the directory are populated from Name Registry DHT records where `metadata.public_directory = true`. Local island services are discovered via the daemon's known island list.

### 4.4 Privacy Controls Panel

```
┌──────────────────────────────────────────────────────────────────┐
│  Privacy Controls — community-news.bgpx                          │
│                                                                  │
│  Path isolation:         [Per service ✓]                         │
│  Clearnet sub-resources: [Warn before loading ✓]                │
│  JavaScript:             [Enabled ✓]                             │
│  Cookies:                [Service-isolated ✓]                    │
│  Storage:                [Isolated to ServiceID ✓]              │
│  Send path type header:  [Yes ✓] (BGP-X-Path-Type header)      │
│                                                                  │
│  [Reset to defaults] [Block all clearnet] [Save]                │
└──────────────────────────────────────────────────────────────────┘
```

### 4.5 Settings

**Connection settings**:
- BGP-X daemon connection (auto-discover / manual socket path / remote router)
- Default pool configuration
- Cover traffic enable/disable
- Path rebuild aggressiveness

**Privacy settings**:
- Block clearnet sub-resources on `.bgpx` pages (global toggle)
- Block `.bgpx` access from clearnet pages (always on; not configurable)
- JavaScript policy per domain type
- Cookie isolation policy
- Fingerprinting protection level

**Performance settings**:
- Latency mode thresholds (when to activate LoRa/Satellite mode)
- Cache size and retention
- Preloading aggressiveness (more = faster but uses more LoRa/satellite airtime/bandwidth)

**Trusted pools** (advanced):
- Configure which pools are used for path construction
- Private pool management

---

## 5. Privacy Defaults

BGP-X Browser ships with stronger privacy defaults than standard browsers:

| Setting | Standard Browser Default | BGP-X Browser Default |
|---|---|---|
| Third-party cookies | Varies | Blocked |
| Tracking pixels | Allowed | Blocked |
| Fingerprinting | Allowed | Blocked (canvas, audio, font enumeration) |
| Storage partitioning | Varies | Per-ServiceID for .bgpx; per-origin for clearnet |
| HTTPS-only | Opt-in | Enabled by default |
| DNS provider | OS default | BGP-X daemon (DoH at exit node) |
| WebRTC IP leak | Allowed | Blocked (WebRTC disabled for clearnet) |
| Referrer header | Full URL | Origin only (clearnet); not sent (`.bgpx`) |
| History | Retained | Clearnet history: session only by default; .bgpx history: optional |

**Built-in content blocker**: uBlock Origin filter lists embedded. Updated via BGP-X overlay (no clearnet DNS query for filter list updates). Ad and tracker blocking active by default.

---

## 6. Platform Targets

| Platform | Engine | Status |
|---|---|---|
| Linux (x86-64) | Electron + Chromium | Primary; developed first |
| macOS (arm64 + x86-64) | Electron + Chromium | Parallel development |
| Windows 10/11 | Electron + Chromium | Parallel development |
| Android 11+ | Custom + WebView | Separate app (bgpx-browser-android) |
| iOS 16+ | Custom + WKWebView | Separate app (bgpx-browser-ios); WKWebView limitations |

### Linux Desktop

BGP-X Browser distributed as:
- AppImage (portable; no install; works on any modern Linux)
- .deb package (Debian/Ubuntu)
- Flatpak (cross-distro; sandboxed)

Connects to system BGP-X daemon at `/var/run/bgpx/sdk.sock` automatically.

### macOS

BGP-X Browser distributed as:
- .dmg disk image (drag to Applications)
- Homebrew cask: `brew install --cask bgpx-browser`

macOS Gatekeeper: BGP-X Browser is notarized and signed with Apple Developer certificate. No "unidentified developer" warning.

### Windows

BGP-X Browser distributed as:
- NSIS installer (.exe)
- Portable .zip (no install)

Windows Defender SmartScreen: signed with EV code signing certificate. No SmartScreen warning on first run.

### Android

BGP-X Browser for Android uses Android VpnService for system-wide overlay plus a WebView-based browser for `.bgpx` URL handling. See `bgpx-browser-android/` for implementation details.

### iOS

BGP-X Browser for iOS uses Network Extension framework for system overlay. WKWebView limitations on iOS prevent custom URL scheme handling at the network layer — `.bgpx` URLs are handled via a JavaScript bridge and native URL scheme registration. Full `.bgpx` browsing supported within BGP-X Browser on iOS; other apps do not benefit from `.bgpx` resolution.

---

## 7. Embedded Mode (No External Daemon)

For deployments where no BGP-X daemon is running (mobile, developer testing):

```rust
// bgpx-sdk embedded mode:
let browser = BgpxBrowser::embedded(BrowserConfig {
    data_dir: platform_data_dir(),
    mesh_enabled: false,     // Mobile: no radio
    cover_traffic: false,    // Mobile: battery conservation
    ..Default::default()
}).await?;
```

In embedded mode, bgpx-sdk runs the BGP-X routing stack in-process. No separate daemon process needed. Network topology and capabilities are limited (no LoRa without USB adapter; no 802.11s; clearnet overlay only on mobile).

---

## 8. Update Mechanism

BGP-X Browser updates are distributed via the BGP-X overlay:
- Update server: a `.bgpx` native service distributing signed update packages
- Update check: browser queries `bgpx://updates.bgpx/check` at startup and once daily
- No clearnet DNS query for update check — entire update flow through overlay
- Update packages: signed with BGP-X release Ed25519 key (same key as daemon updates)
- Auto-update: enabled by default; can be disabled in settings
- User notification: non-intrusive banner when update available; auto-applies on next restart

---

## 9. HTTP/2 over BGP-X Streams

BGP-X native services (.bgpx) use HTTP/2 over BGP-X streams.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers — this is standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 10. Satellite Latency Adaptation

BGP-X Browser adapts its rendering strategy for satellite-GEO-class latency (600ms+ RTT):

- **Batched resource loading**: Multiple resources requested in one round-trip
- **Progressive rendering**: Skeleton UI shows immediately; content fills progressively
- **Estimated load time**: Latency indicator shows estimated total load time based on measured throughput
- **No spinner timeouts**: Standard browser timeout indicators are disabled or significantly extended
- **Cache aggressively**: High cache priority for satellite-served resources

**Distinction**: Commercial satellite internet (Starlink, Iridium) is clearnet domain (0x00000001). The browser sees "Clearnet via BGP-X" but detects high latency. The UI indicator shows `[🛰 Satellite]` based on measured latency class, not domain type. This informs the user of expected delays without implying a separate "Satellite Domain" (domain 0x00000005 is reserved and not active).
