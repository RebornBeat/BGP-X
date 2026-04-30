# BGP-X Domain System (.bgpx)

**Version**: 0.1.0-draft

---

## 1. Overview

The `.bgpx` domain system is BGP-X's equivalent of Tor's `.onion`. It provides:

- **Service addressing**: any BGP-X native service is reachable at a `.bgpx` address
- **Identity binding**: the `.bgpx` address IS the service's public key identity — no CA required
- **Cross-domain access**: `.bgpx` addresses work from clearnet, mesh islands, and any routing domain
- **Human-readable names**: short names via the BGP-X Name Registry (`name.bgpx`)
- **Browser integration**: BGP-X Browser handles `bgpx://` URLs natively
- **Latency transparency**: services declare their latency class; clients adapt

---

## 2. Address Formats

### 2.1 Full ServiceID Address

```
<64-hex-characters>.bgpx

Example:
a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx
```

The 64-hex-character string is the hex encoding of the service's 32-byte Ed25519 public key (the ServiceID). This is the canonical address form. It is self-authenticating: the BGP-X Browser verifies that the service's Ed25519 public key matches the ServiceID in the address. No certificate authority is involved.

**Comparison to .onion v3**: .onion v3 addresses are 56 characters (base32, 35 bytes = 32-byte public key + 2-byte version + 1-byte checksum). `.bgpx` addresses are 64 characters (hex, 32 bytes raw public key). Both are self-authenticating via public key derivation.

### 2.2 Short Name Address (Name Registry)

```
<shortname>.bgpx

Example:
community-news.bgpx
library.bgpx
mesh-radio.bgpx
```

Short names are registered in the BGP-X Name Registry DHT. A short name maps to a ServiceID. Resolution:

```
1. Client queries Name Registry DHT: lookup("community-news")
2. DHT returns NameRecord: { name: "community-news", service_id: "a3f2b9...", expiry: ..., signature: ... }
3. Client verifies signature (NameRecord signed by service Ed25519 key)
4. Client uses service_id to construct overlay path
5. Client presents full-address validation error if resolved ServiceID doesn't match handshake
```

### 2.3 Mesh Island Service Address

```
<64-hex>.mesh.<island_id>.bgpx

Example:
a3f2b9c1....bgpx          (any domain — resolver finds it)
a3f2b9c1....mesh.lima-district-1.bgpx   (explicit mesh island hint)
```

The `mesh.<island_id>` suffix is an optional routing hint. It tells the BGP-X daemon to preferentially construct a cross-domain path through the specified mesh island rather than querying the unified DHT for the service globally. Without the suffix, the daemon queries the DHT and finds the service regardless of which domain it's in — the suffix is an optimization for clients that know which island a service is in.

### 2.4 URL Scheme

```
bgpx://<address>/<path>?<query>

Examples:
bgpx://a3f2b9c1....bgpx/
bgpx://community-news.bgpx/article/1234
bgpx://library.bgpx/search?q=privacy
bgpx://a3f2b9c1....mesh.lima-district-1.bgpx/
```

The BGP-X Browser registers the `bgpx://` URL scheme. Standard HTTP semantics apply: GET, POST, headers, cookies, etc. All carried over BGP-X streams with end-to-end onion encryption.

---

## 3. Service Identity and Authentication

### 3.1 ServiceID Derivation

A BGP-X service's identity is its Ed25519 public key. The ServiceID is the 32-byte raw public key. The `.bgpx` address is the hex encoding of this 32-byte key.

```rust
// Service operator generates a long-term Ed25519 keypair:
let service_keypair = Ed25519KeyPair::generate(&mut csprng);
let service_id = service_keypair.public_key().as_bytes(); // 32 bytes
let bgpx_address = hex::encode(service_id) + ".bgpx";   // 64 hex chars + ".bgpx"
```

The service keypair should be generated once and kept securely. It is the permanent identity of the service.

### 3.2 Browser Authentication (No CA)

When BGP-X Browser connects to `a3f2b9c1....bgpx`:

1. Browser extracts ServiceID from address: hex-decode `a3f2b9c1...` → 32 bytes
2. Browser constructs BGP-X overlay path to the service
3. BGP-X handshake: client performs X25519 key exchange with service's static public key
4. The service's static public key IS the ServiceID
5. Browser verifies: `BLAKE3(service.static_public_key) == address_service_id`
   - If match: display green `.bgpx` verified badge
   - If mismatch: display security error, abort connection

No certificate authority. No expiry date. No revocation lists. The address IS the cryptographic identity.

**Key rotation**: If a service rotates its Ed25519 key, its `.bgpx` address changes. Operators should publish a redirect from the old service (before shutdown) to the new address, and update their Name Registry entry.

### 3.3 Security Indicator in BGP-X Browser

| Indicator | Meaning |
|---|---|
| 🟢 `.bgpx` verified badge | Service Ed25519 key matches address; full onion encryption; no CA needed |
| 🌐 Grey overlay icon | Clearnet site reached via BGP-X overlay; standard TLS only |
| 🔶 Orange warning | Clearnet site NOT going through overlay (if policy allows bypass) |
| 🔴 Red `.bgpx` mismatch | Service key doesn't match address; abort; do not display content |

---

## 4. HTTP/2 Over BGP-X Streams

`.bgpx` services use HTTP/2 framing carried over BGP-X streams. No TLS layer is used for `.bgpx` (BGP-X onion encryption provides transport security). TLS is optional and available for additional authentication scenarios.

### 4.1 Request Multiplexing

HTTP/2's stream multiplexing over a single BGP-X stream connection means:

```
BGP-X Stream (single overlay path connection)
  └── HTTP/2 Session
        ├── HTTP/2 Stream 1: GET /index.html
        ├── HTTP/2 Stream 3: GET /style.css
        ├── HTTP/2 Stream 5: GET /logo.png
        └── HTTP/2 Stream 7: GET /app.js
```

All HTTP/2 streams share one BGP-X overlay path. This is critical for performance: only one path construction cost per page load, regardless of how many resources the page requires.

### 4.2 Resource Prioritization

HTTP/2 stream priority maps to BGP-X stream priority:

| Resource Type | HTTP/2 Priority | BGP-X Stream Priority |
|---|---|---|
| HTML document | Highest (0) | 0 (control class) |
| CSS files | High (1) | 1 |
| Critical JS | High (2) | 2 |
| Images above fold | Medium (3-4) | 3 |
| Images below fold | Low (5-6) | 5 |
| Non-blocking JS | Low (6-7) | 6 |
| Analytics, tracking | Lowest (7) | 7 |

### 4.3 Latency-Tolerant Headers

Services declare their latency class in HTTP response headers:

```http
X-BGP-X-Latency-Class: lora
X-BGP-X-Batch-Resources: true
X-BGP-X-Cache-TTL: 86400
```

**`X-BGP-X-Latency-Class`**: hints to the browser about expected path latency. Values: `standard`, `overlay`, `mesh-wifi`, `mesh-lora`, `satellite-geo`. Browser adjusts rendering strategy.

**`X-BGP-X-Batch-Resources`**: if `true`, browser preloads all `<link rel="preload">` resources in one HTTP/2 PUSH or batch request before rendering. Reduces round-trips on high-latency paths.

**`X-BGP-X-Cache-TTL`**: maximum seconds the browser should cache this resource. Services on LoRa paths should set high values to avoid refetching unchanged content on every visit.

### 4.4 WebSocket over BGP-X

WebSocket connections over `.bgpx`:

```
ws://service.bgpx/channel     (BGP-X browser creates bgpx:// websocket)
wss://service.bgpx/channel    (same, with optional additional TLS layer)
```

BGP-X streams provide reliable ordered delivery; WebSocket frames are carried natively without further adaptation. Real-time bidirectional communication works, with latency matching the path type (LoRa paths will have high round-trip delay for WebSocket messages — applications should use server-push patterns instead of polling for LoRa deployments).

### 4.5 Server-Sent Events (SSE) over BGP-X

SSE is preferred over WebSocket polling for LoRa-tolerant applications:

```javascript
// .bgpx service client (in BGP-X Browser):
const events = new EventSource("bgpx://my-service.bgpx/events");
events.onmessage = (e) => updateUI(e.data);
```

The server pushes updates when they occur. The client does not need to poll. For a community message board on a LoRa island, SSE means users receive new messages when posted without repeatedly querying the server — each query costs LoRa airtime and latency.

---

## 5. Batched Resource Loading for LoRa Paths

When BGP-X Browser detects a LoRa-class path (from daemon path status event), it activates batched resource loading mode:

### Browser Batching Algorithm

```
1. Fetch HTML document (HTTP/2 stream 1)
2. Parse HTML, extract all resource dependencies
3. Build resource dependency graph:
   critical_path = [CSS files, above-fold images, blocking JS]
   deferred_path = [below-fold images, non-blocking JS, analytics]
4. Send all critical_path resources as one HTTP/2 batch:
   HEADERS frame for each (all in one LoRa transmission budget)
5. Render skeleton layout with CSS immediately upon receipt
6. Fill content as resources arrive (progressive rendering)
7. Send deferred_path resources after first render
8. Images load last; render without waiting for all images
```

This reduces the number of round-trips from O(N resources) to O(2) or O(3) for typical pages, significantly reducing the impact of LoRa latency on page load experience.

### User Experience on LoRa Paths

```
Time 0ms:   User enters bgpx://service.bgpx in BGP-X Browser
Time 500ms: BGP-X overlay path constructed (cached or fresh)
Time 1500ms: HTML received; skeleton renders immediately (empty layout, CSS applied)
Time 2500ms: Critical CSS received; full layout renders with correct styles
Time 4000ms: Above-fold text content received; user reads content
Time 6000ms: Below-fold images received; page fully loaded
```

The user sees content within 1.5-4 seconds and can read/interact before images load. This is comparable to early mobile web browsing experiences — usable and functional, not seamless.

**The address bar shows**: `🟢 service.bgpx [LoRa ~3s/resource]` — the latency indicator makes it clear why loading is slower, sets expectations, and prevents users from assuming the service is down.

---

## 6. Name Registry Integration

### 6.1 Short Name Resolution

When BGP-X Browser encounters `shortname.bgpx`:

```
1. Check local name cache (LRU, 10,000 entries, TTL from NameRecord.expiry)
2. If cached and not expired: use cached ServiceID
3. If not cached: query Name Registry DHT
   3a. Send DHT query: GET("name_registry_v1:" + BLAKE3(shortname))
   3b. Receive NameRecord or "not found"
   3c. If "not found": display "Service not found" error page
4. Verify NameRecord signature: BLAKE3("name_registry_v1:" || name || ":" || service_id) signed by service Ed25519 key
5. If signature valid: cache NameRecord; proceed with ServiceID
6. If signature invalid: display "Name record tampered" error page; do not connect
```

### 6.2 DNS-over-BGP-X

All `.bgpx` resolution happens within the BGP-X overlay — no clearnet DNS is consulted. There is no DNS leak for `.bgpx` addresses.

For clearnet addresses resolved by the BGP-X daemon at the exit node, DNS uses DoH (DNS-over-HTTPS) via the exit node. The BGP-X Browser itself does not make clearnet DNS queries for either `.bgpx` or clearnet domains — all DNS is handled by the BGP-X daemon.

### 6.3 Mixed Content Policy

- A `.bgpx` page may embed clearnet sub-resources (images, CSS, scripts from clearnet)
  - Browser warns: "This .bgpx page loads clearnet resources. Clearnet resources may reveal your identity to those servers."
  - User can configure: block all clearnet resources from `.bgpx` pages
- A clearnet page cannot embed `.bgpx` sub-resources (`.bgpx` resources are only accessible within the BGP-X overlay)
- Cross-origin rules: the ServiceID (32-byte hex) is the origin for `.bgpx` services; standard CORS rules apply

---

## 7. .bgpx vs. .onion Comparison

| Feature | .onion v3 (Tor) | .bgpx (BGP-X) |
|---|---|---|
| Address length | 56 chars (base32) | 64 chars (hex) |
| Key type | Ed25519 (same) | Ed25519 (same) |
| Self-authenticating | Yes | Yes |
| CA required | No | No |
| Human-readable names | No (Tor lacks name registry) | Yes (Name Registry DHT) |
| Cross-domain access | No (Tor is internet only) | Yes (clearnet, mesh, satellite) |
| LoRa access | No (Tor is internet only) | Yes (latency-tolerant browser) |
| Mesh island hosting | No | Yes |
| Browser integration | Tor Browser | BGP-X Browser |
| N-hop unlimited | No (fixed 6 hops) | Yes |
| Pool-based trust | No | Yes |
| ECH at exit | No | Yes |

---

## 8. Error Pages

BGP-X Browser displays specific error pages for `.bgpx` failure conditions:

| Error | Cause | Display |
|---|---|---|
| `BGPX_SERVICE_OFFLINE` | Service not found in DHT or path construction failed | "This .bgpx service is offline or unreachable. ServiceID: [hex]" |
| `BGPX_PATH_FAILED` | Overlay path could not be constructed | "Could not build a path to this service. Try again." |
| `BGPX_ISLAND_UNREACHABLE` | Mesh island gateway offline | "The mesh island hosting this service is currently unreachable from the clearnet." |
| `BGPX_KEY_MISMATCH` | Service key doesn't match address | "Security error: this service's identity does not match its address. Do not proceed." |
| `BGPX_NAME_NOT_FOUND` | Short name not in Name Registry | "This .bgpx name is not registered. Check spelling or use the full address." |
| `BGPX_DAEMON_NOT_CONNECTED` | BGP-X daemon not running | "BGP-X Browser is not connected to the BGP-X daemon. Please start the BGP-X daemon." |
| `BGPX_LORA_DUTY_CYCLE` | LoRa path duty cycle exhausted | "LoRa path is paused due to regulatory duty cycle limits. Will resume in ~[N] seconds." |

---

## 9. .bgpx Service Requirements

Services hosted at `.bgpx` addresses SHOULD:

- Implement HTTP/2 (not HTTP/1.1) for multiplexed resource delivery
- Set aggressive cache headers for static resources (`Cache-Control: max-age=86400`)
- Declare latency class via `X-BGP-X-Latency-Class` header
- Set `X-BGP-X-Batch-Resources: true` for resource-heavy pages
- Use Brotli compression (significant for LoRa bandwidth)
- Use server-sent events instead of polling for real-time updates
- Minimize external resources (each clearnet sub-resource = one more leak vector)
- Design for progressive loading (meaningful content visible before images)

Services on LoRa islands MUST also:

- Set `X-BGP-X-Latency-Class: lora`
- Keep page weights minimal (<100 KB total is strongly recommended)
- Avoid JavaScript frameworks with large bundles
- Cache aggressively on the client and server
- Use low-resolution images or text alternatives
