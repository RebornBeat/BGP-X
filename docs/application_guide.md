# BGP-X Application Developer Guide

**Version**: 0.1.0-draft

This document explains how to build applications that use BGP-X — whether as standard applications that benefit from transparent router protection, or as BGP-X native applications that explicitly control overlay capabilities.

---

## 1. Three Application Types

Before choosing an integration approach, understand what each type provides:

### Type 1: Standard Application (No Integration Required)

Your application doesn't use the BGP-X SDK. If the router is running BGP-X and its routing policy routes your application's traffic through the overlay, you get privacy protection with zero code changes.

**When this is enough**:
- You don't need to control which relay nodes are used
- You don't need to specify path length, pools, or ECH requirements
- You don't need to register as a BGP-X native service
- The router-level transparent protection is sufficient for your threat model

**What you get**: transparent onion-routed traffic for all connections matching the router's routing policy. No code changes, no SDK, no configuration.

### Type 2: BGP-X Native Application (SDK Integration)

Your application uses the BGP-X SDK to explicitly control overlay capabilities.

**When you need this**:
- You need specific path constraints (minimum 5 hops, no US jurisdiction, ECH required)
- You want to register as a BGP-X native service (reachable via ServiceID, no clearnet IP required)
- You need to build a double-exit or N-hop unlimited architecture
- You want path quality event subscriptions for adaptive behavior
- You need to manage your own session identity (ephemeral or persistent)

**What you get**: full programmatic control over BGP-X overlay routing. Streams to clearnet or native destinations with explicitly specified path parameters.

### Type 3: Configuration Client (No SDK)

You're building a management tool for BGP-X — a GUI alternative to bgpx-cli.

**When you need this**:
- You're building a router management interface
- You want to display BGP-X status, path information, node reputation, pool health
- You need to manage routing policy rules, pools, blacklists

**What you use**: Control Socket (JSON-RPC 2.0), not the SDK Socket.

---

## 2. SDK Connection Model

This is critical to understand: **the BGP-X SDK does NOT implement its own routing stack.**

The SDK is a client library that communicates with a running BGP-X daemon. All path construction, onion encryption, DHT participation, and pool management are done by the daemon.

```
[Your Application]
        │
        │  SDK API calls (connect_stream, register_service, etc.)
        ▼
[BGP-X SDK Library]
        │
        │  JSON-RPC 2.0 over Unix socket (or TCP for remote daemon)
        ▼
[BGP-X Daemon] (on router, or on same device, or embedded in-process)
        │
        │  UDP/mesh transport
        ▼
[BGP-X Overlay Network]
```

The daemon can be:
- On a **network router** serving your entire LAN — connect to router's SDK socket
- Running **locally** on the same device — connect to local socket
- **Embedded in-process** by the SDK — full stack runs inside your application (for mobile)

### SDK Auto-Discovery

The SDK automatically discovers the daemon in this priority order:

1. `BGPX_SOCKET` environment variable
2. `/var/run/bgpx/sdk.sock` (system daemon install)
3. `~/.bgpx/sdk.sock` (user daemon install)
4. `127.0.0.1:7475` (TCP fallback for local daemon with TCP endpoint enabled)
5. Embedded mode (if compiled with `features = ["embedded"]`)

```rust
// Auto-discover daemon
let client = Client::connect_auto().await?;

// Or specify explicitly
let client = Client::connect(Path::new("/var/run/bgpx/sdk.sock")).await?;

// Remote router via TCP
let client = Client::connect_tcp("192.168.1.1:7475").await?;

// Embedded mode (no daemon process required)
let client = Client::embedded(ClientConfig::default()).await?;
```

### Connecting a LAN Device to the Router Daemon

If your application runs on a LAN device and the BGP-X daemon is on the router:

#### Option A: SSH Socket Forwarding (Recommended)

```bash
# On the LAN device, in a terminal:
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@192.168.1.1

# Keep this running while your app needs the connection
```

In your application:
```rust
let client = Client::connect(Path::new("/tmp/bgpx-router.sock")).await?;
```

#### Option B: Router TCP Endpoint

On the router, configure the daemon:
```toml
[sdk_api]
tcp_listen = "192.168.1.1:7475"  # LAN IP only, never WAN
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/sdk_auth_token"
```

In your application:
```rust
let client = Client::connect_tcp_with_auth(
    "192.168.1.1:7475",
    "your-auth-token-here"
).await?;
```

#### Option C: Standard Application Mode (No SDK Needed)

If your application doesn't need path control or service registration, consider whether you actually need the SDK:
- Just connect to the internet normally
- The router's BGP-X overlay protects your traffic transparently
- No code changes required

---

## 3. Latency-Adaptive Application Design

BGP-X applications may run over paths with widely varying latency:
- Clearnet overlay: 100-400ms per round-trip
- WiFi mesh: 50-200ms per round-trip
- LoRa mesh: 500ms-5s per round-trip
- Satellite GEO clearnet: 1500-3000ms per round-trip

**Tier 3 applications (BGP-X-native) MUST handle LoRa-class latency.** This is not optional — a BGP-X mesh island user may be on a LoRa path, and the application must not time out, show an error, or hang.

### LoRa-Tolerant Design Patterns

**Pattern 1: Subscribe to path status events**

```rust
let mut events = client.subscribe(&[
    EventType::StreamPaused,
    EventType::StreamResumed,
    EventType::PathDegraded,
]).await?;

while let Some(event) = events.next().await {
    match event {
        BgpxEvent::StreamPaused { stream_id, reason, domain } => {
            // Pause outgoing operations; show waiting indicator
            ui.show_waiting(&format!("Stream paused: {} ({})", reason, domain));
        }
        BgpxEvent::StreamResumed { stream_id } => {
            // Resume operations
            ui.hide_waiting();
        }
        _ => {}
    }
}
```

**Pattern 2: Batched requests instead of sequential**

```rust
// WRONG for LoRa paths: sequential requests cause N × round-trip-time latency
let html = fetch("/index.html").await?;
let css = fetch("/style.css").await?;
let data = fetch("/api/data").await?;

// CORRECT for LoRa paths: parallel requests over HTTP/2 multiplexing
let (html, css, data) = tokio::try_join!(
    fetch("/index.html"),
    fetch("/style.css"),
    fetch("/api/data"),
)?;
```

**Pattern 3: Progressive rendering / optimistic UI**

```rust
// Start rendering with cached or skeleton content immediately
ui.render_skeleton();

// Fetch actual content in background
let content = fetch_content().await?;

// Update UI when content arrives (user can see something immediately)
ui.update_content(content);
```

**Pattern 4: Aggressive caching**

```rust
// Cache responses locally with long TTLs
// For static .bgpx resources: days or weeks
// For dynamic content: minutes depending on update frequency
let response = cache.get_or_fetch("resource_key", Duration::from_secs(86400), || {
    fetch_from_service()
}).await?;
```

**Pattern 5: Server-sent events instead of polling**

```rust
// WRONG for LoRa: polling every N seconds costs LoRa airtime
loop {
    let updates = fetch_updates().await?;
    process(updates);
    sleep(Duration::from_secs(30)).await;
}

// CORRECT for LoRa: server pushes when changes occur
let mut event_stream = client.connect_sse("bgpx://service.bgpx/events").await?;
while let Some(event) = event_stream.next().await {
    process(event);
}
```

**Pattern 6: Offline-capable design**

```rust
// Store state locally; sync when connected
let local_db = LocalDatabase::open("app_data.db")?;
local_db.save(state)?;

// Sync when stream is available; handle StreamPaused gracefully
match sync_with_service(&mut stream, &local_db).await {
    Ok(_) => {},
    Err(SdkError::StreamPaused) => {
        // Offline; will sync when stream resumes
        log::info!("Offline mode; sync will resume automatically");
    }
    Err(e) => return Err(e),
}
```

---

## 4. Opening a Stream — Clearnet

### Simple Connection

```rust
use bgpx_sdk::{Client, StreamConfig};

let client = Client::connect_auto().await?;

let mut stream = client.connect_stream(
    "example.com:443",
    StreamConfig::default(),
).await?;

// Use stream like a normal TCP stream (implements AsyncRead + AsyncWrite)
stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;
let mut response = Vec::new();
stream.read_to_end(&mut response).await?;
```

### With Path Constraints

```rust
use bgpx_sdk::{PathConfig, SegmentConfig, SegmentConstraints, LoggingPolicy};

let stream = client.connect_stream_with_path(
    "sensitive-site.com:443",
    PathConfig {
        segments: vec![
            SegmentConfig {
                pool: "bgpx-default".into(),
                hops: 3,
                is_exit: false,
                ..Default::default()
            },
            SegmentConfig {
                pool: "my-private-exit".into(),
                hops: 1,
                is_exit: true,
                constraints: SegmentConstraints {
                    require_ech: true,
                    exit_logging_policy: Some(LoggingPolicy::None),
                    exit_jurisdiction_blacklist: vec!["US".into(), "GB".into()],
                    ..Default::default()
                },
            },
        ],
        ..Default::default()
    },
    StreamConfig {
        require_ech: true,
        isolated_path: true,  // This stream gets its own path, not shared
        ..Default::default()
    },
).await?;
```

### Double-Exit Architecture

For maximum isolation, use two independent exit pools in sequence. This is one of several anonymization strategies supported by BGP-X's N-hop unlimited architecture:

```rust
// Two independent exit pools in sequence
// Exit Pool 1 sees: traffic going to Exit Pool 2 (not destination)
// Exit Pool 2 sees: traffic from Exit Pool 1 (not client)
let stream = client.connect_stream_with_path(
    "destination.example.com:443",
    PathConfig {
        segments: vec![
            SegmentConfig {
                pool: "bgpx-default".into(),
                hops: 2,
                is_exit: false,
                ..Default::default()
            },
            SegmentConfig {
                pool: "exit-tier-1".into(),
                hops: 1,
                is_exit: false,  // Not a clearnet exit — routes to next segment
                ..Default::default()
            },
            SegmentConfig {
                pool: "exit-tier-2".into(),
                hops: 1,
                is_exit: true,  // This is the clearnet exit
                ..Default::default()
            },
        ],
        ..Default::default()
    },
    StreamConfig::default(),
).await?;
```

---

## 5. Connecting to .bgpx Services

### By Full ServiceID

```rust
let stream = client.connect_stream(
    "bgpx://a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx",
    StreamConfig::default(),
).await?;
```

### By Short Name

```rust
// SDK resolves name via Name Registry DHT automatically
let stream = client.connect_stream(
    "bgpx://community-news.bgpx/",
    StreamConfig::default(),
).await?;
```

---

## 6. Cross-Domain Streams

### Clearnet Client → Mesh Island Service

No mesh hardware required. The bridge node handles all radio transmission.

```rust
use bgpx_sdk::{DomainSegmentConfig, DomainSegmentType, RoutingDomain};

let stream = client.connect_stream_cross_domain(
    "bgpx://library.bgpx/",
    vec![
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Clearnet),
            pool: "bgpx-default".into(),
            hops: 3,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::BridgeTo {
                target_domain: RoutingDomain::Mesh {
                    island_id: "lima-district-1".into()
                }
            },
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Mesh {
                island_id: "lima-district-1".into()
            }),
            hops: 2,
            ..Default::default()
        },
    ],
    Default::default(),
).await?;
```

### Clearnet → Mesh → Clearnet Exit (Three Domains)

A path that traverses a mesh island as an intermediate hop for maximum isolation:

```rust
let stream = client.connect_stream_cross_domain(
    "destination.example.com:443",
    vec![
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Clearnet),
            pool: "bgpx-default".into(),
            hops: 2,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::BridgeTo {
                target_domain: RoutingDomain::Mesh { island_id: "trusted-island".into() }
            },
            domain_transition: DomainTransition::Auto,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Mesh { island_id: "trusted-island".into() }),
            hops: 3,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::BridgeTo {
                target_domain: RoutingDomain::Clearnet,
            },
            domain_transition: DomainTransition::Auto,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Clearnet),
            pool: "private-exits".into(),
            hops: 1,
            is_exit: true,
            constraints: SegmentConstraints {
                require_ech: true,
                exit_logging_policy: Some(LoggingPolicy::None),
                ..Default::default()
            },
            ..Default::default()
        },
    ],
    StreamConfig { require_ech: true, ..Default::default() },
).await?;
```

### N-Hop Unlimited (Any Count, Any Combination)

BGP-X imposes no protocol-level maximum on path length, domain count, or domain ordering. Clients may construct paths with any number of hops, traversing any combination of domains in any order.

```rust
let stream = client.connect_stream_with_path(
    "destination.com:443",
    PathConfig {
        domain_segments: vec![
            // 6 clearnet hops in segment 1
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Clearnet),
                hops: 6,
                ..Default::default()
            },
            // Bridge to mesh
            DomainSegmentConfig {
                segment_type: DomainSegmentType::BridgeTo {
                    target_domain: RoutingDomain::Mesh {
                        island_id: "relay-island".into()
                    }
                },
                ..Default::default()
            },
            // 7 mesh hops
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Mesh {
                    island_id: "relay-island".into()
                }),
                hops: 7,
                ..Default::default()
            },
            // Bridge back to clearnet
            DomainSegmentConfig {
                segment_type: DomainSegmentType::BridgeTo {
                    target_domain: RoutingDomain::Clearnet,
                },
                ..Default::default()
            },
            // 5 clearnet exit hops
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Clearnet),
                pool: "private-exits".into(),
                hops: 5,
                is_exit: true,
                ..Default::default()
            },
        ],
        max_total_hops: None,   // No protocol limit enforced
        ..Default::default()
    },
    Default::default(),
).await?;
// Total: 20 hops across 2 bridges and 3 domains — fully valid
```

---

## 7. Registering a .bgpx Service

### Server Side

```rust
let listener = client.register_service(ServiceConfig {
    name: "my-private-api".to_string(),
    advertise: true,    // Publish ServiceID + short name to DHT
    auth_required: false,
    max_connections: 100,
}).await?;

println!("Service address: {}.bgpx", listener.service_id().to_hex());
println!("bgpx://my-private-api.bgpx (if short name registered)");

loop {
    let (mut stream, info) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(stream, info).await;
    });
}
```

### Registering a Short Name

```rust
// After registering the service:
let name_registry = client.name_registry().await?;
name_registry.register(
    "my-api",           // short name
    service_keypair,    // the service's Ed25519 keypair
    NameMetadata {
        description: "My private API service".to_string(),
        service_type: ServiceType::Api,
        icon_hash: None,
    },
    Duration::from_days(90),
).await?;

// Service now reachable at:
// bgpx://my-api.bgpx/
// bgpx://<64-hex>.bgpx/
```

### Mesh Island Service

Services registered within a mesh island are accessible from clearnet clients without any special configuration:

```rust
// On mesh island node:
let listener = client.register_service(ServiceConfig {
    name: "community-forum".to_string(),
    advertise: true,    // Published to unified DHT via bridge node
    ..Default::default()
}).await?;
```

```rust
// From clearnet client (no mesh hardware needed):
let stream = client.connect_stream(
    &format!("bgpx://{}", community_forum_service_id.to_hex()),
    Default::default(),
).await?;
// Daemon automatically builds cross-domain path to mesh island
```

---

## 8. HTTP Server for .bgpx Services

### Latency-Tolerant HTTP Response Headers

```rust
// In your HTTP handler:
fn build_response(body: Vec<u8>) -> HttpResponse {
    HttpResponse::Ok()
        .header("Content-Type", "text/html; charset=utf-8")
        .header("Content-Encoding", "br")          // Brotli: essential for LoRa bandwidth
        .header("Cache-Control", "max-age=86400")   // Aggressive caching for static content
        .header("X-BGP-X-Latency-Class", "lora")   // Hint: this service may be on LoRa path
        .header("X-BGP-X-Batch-Resources", "true") // Browser: batch fetch all resources
        .header("X-BGP-X-Cache-TTL", "86400")      // Cache hint for BGP-X Browser
        .body(body)
}
```

### Minimal Page Weight for LoRa

For services on LoRa-accessible mesh islands, minimize page weight:
- Target: <50 KB total page weight (HTML + CSS + critical JS)
- Avoid large JavaScript frameworks (React, Vue, Angular bundle sizes are too large for LoRa)
- Use SVG for icons (scalable, small)
- Avoid external fonts (load over LoRa = one expensive round-trip for each)
- Images: WebP or AVIF; maximum 30 KB each; lazy-load below fold
- Prefer text + CSS layout over image-heavy design

---

## 9. Domain and Island Discovery

```rust
// List all known routing domains
let domains = client.list_domains().await?;
for domain in domains {
    println!("{:?}: relay_count={}, bridge_count={}",
        domain.domain_type, domain.active_relay_count, domain.bridge_node_count);
}

// Get status of a mesh island
let status = client.get_island_status("lima-district-1").await?;
println!("Island online: {}", status.online);
println!("Active relays: {}", status.active_relay_count);
println!("Bridge nodes: {}", status.bridge_node_count);

// Find bridge nodes between two domains
let bridges = client.discover_bridges(
    RoutingDomain::Clearnet,
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;

// Test connectivity to a domain
let result = client.test_domain_connectivity(
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;
println!("Reachable: {}", result.reachable);
println!("Best bridge latency: {}ms", result.best_bridge_latency_ms.unwrap_or(0));
```

---

## 10. Pool Management

```rust
// Add a private pool (clearnet exit pool)
client.add_pool(Pool {
    pool_id: "my-private-exit".into(),
    pool_type: PoolType::Private,
    domain_scope: PoolDomainScope::Clearnet,
    curator_public_key: my_curator_key,
    members: vec!["node_id_1".parse()?, "node_id_2".parse()?],
    ..Default::default()
}).await?;

// Add a domain-bridge pool (bridge nodes for specific pair)
client.add_pool(Pool {
    pool_id: "trusted-bridges-to-lima".into(),
    pool_type: PoolType::DomainBridge {
        from_domain: DomainId::clearnet(),
        to_domain: DomainId::mesh("lima-district-1"),
    },
    curator_public_key: my_curator_key,
    members: vec!["bridge_node_1".parse()?, "bridge_node_2".parse()?],
    ..Default::default()
}).await?;
```

---

## 11. Path Quality Events

```rust
let mut events = client.subscribe(&[
    EventType::PathBuilt,
    EventType::PathDegraded,
    EventType::PathRebuilt,
    EventType::PoolMemberOffline,
    EventType::DomainBridgeOnline,
    EventType::DomainBridgeOffline,
    EventType::MeshIslandUnreachable,
    EventType::MeshIslandOnline,
    EventType::StreamPaused,
    EventType::StreamResumed,
]).await?;

while let Some(event) = events.next().await {
    match event {
        BgpxEvent::PathDegraded { quality_score, reason } => {
            eprintln!("Path degraded: {:.2} ({})", quality_score, reason);
        }
        BgpxEvent::MeshIslandUnreachable { island_id } => {
            eprintln!("Mesh island unreachable: {}", island_id);
            // Cross-domain streams to that island are paused
        }
        BgpxEvent::MeshIslandOnline { island_id } => {
            eprintln!("Mesh island back online: {}", island_id);
            // Paused streams will resume
        }
        BgpxEvent::DomainBridgeOffline { from_domain, to_domain } => {
            eprintln!("Bridge offline: {} → {}", from_domain, to_domain);
        }
        BgpxEvent::StreamPaused { stream_id, reason, domain } => {
            eprintln!("Stream {:?} paused: {} (in domain {})", stream_id, reason, domain);
        }
        _ => {}
    }
}
```

---

## 12. Identity Management

### Ephemeral Identity (Default — Recommended)

Each session gets a fresh Ed25519 keypair. Activities across sessions cannot be linked at the protocol level.

```rust
// Default: ephemeral identity, no configuration needed
let client = Client::connect_auto().await?;
```

### Persistent Identity (Optional — for authenticated services)

Use a stable identity when the service needs to recognize returning clients.

```rust
let client = Client::connect_with_config(ClientConfig {
    identity_mode: IdentityMode::Persistent {
        key_path: Path::new("/etc/myapp/bgpx_identity_key"),
    },
    ..Default::default()
}).await?;
```

**Warning**: persistent identity creates a linkable fingerprint across sessions. Only use when authentication is necessary and the privacy trade-off is acceptable.

---

## 13. ECH Requirements

ECH applies only to clearnet exit streams. Mesh island native services use BGP-X onion encryption end-to-end — no TLS ClientHello SNI to protect.

```rust
// Require ECH for stream (fail if not available)
StreamConfig {
    require_ech: true,
    prefer_ech: true,    // default: true
    ..Default::default()
}

// Check ECH status after stream established
let info = stream.info();
println!("ECH used: {}", info.ech_negotiated);
println!("ECH available at destination: {}", info.ech_available);
```

If `require_ech = true` and destination doesn't publish ECH configuration, stream open fails with `SdkError::EchRequired`. The application should either fall back to a non-ECH connection (if the security trade-off is acceptable) or inform the user that the destination does not support ECH.

---

## 14. Testing Without a Router

### Local Daemon

```bash
bgpx-node --config dev_config.toml
```

Your application connects to local daemon via auto-discovery.

### Embedded Mode

```rust
// Cargo.toml: bgpx-sdk = { version = "0.1", features = ["embedded"] }

let client = Client::embedded(ClientConfig {
    data_dir: temp_dir(),
    mesh_enabled: false,
    cover_traffic: false,
    ..Default::default()
}).await?;
// Same API as daemon-connected mode
```

### Testing LoRa-Path Behavior

```rust
// Simulate LoRa latency in tests
let client = Client::embedded(ClientConfig {
    simulate_latency: Some(SimulatedLatency {
        class: LatencyClass::LoRa,
        per_hop_ms: 1000,
        hop_count: 5,
    }),
    ..Default::default()
}).await?;

// Test your application's LoRa-tolerant behavior
// Your app should handle 5000ms round-trip without timing out
```

---

## 15. Platform Notes

### Android

BGP-X on Android uses VpnService API for system-wide overlay protection. SDK embedded mode runs as a background service. `.bgpx` URL handling via BGP-X Browser Android. LoRa not supported on Android without USB adapter (USB OTG required).

Target hardware: Android devices (phones, tablets) for client apps. For mesh participation, use a BGP-X Client Node (Tier 2) connected via Bluetooth or USB, or a BGP-X Adapter (Tier 3).

### iOS

iOS Network Extension for system-wide overlay. Embedded SDK mode. BGP-X Browser uses WKWebView with custom URL scheme handler for `bgpx://`. Background execution limited by iOS battery management.

### Desktop (Linux/macOS/Windows)

Recommended: system daemon (`bgpx-node`) + SDK via Unix socket. BGP-X Browser as Electron application. Same application code works with local daemon or remote router daemon (SSH forwarding).

Target hardware:
- **BGP-X Router v1 / Node v1**: For full node participation
- **BGP-X Adapter/Dongle (Tier 3)**: For adding LoRa connectivity to desktop
- **No additional hardware**: For clearnet overlay participation only

---

## 16. Hardware Ecosystem Reference

BGP-X applications run on the full hardware ecosystem:

| Tier | Hardware | SDK Mode | Application Type |
|---|---|---|---|
| Tier 1 | BGP-X Router v1 | Daemon-connected or embedded | Standard, Native, Provider |
| Tier 1 | BGP-X Node v1 | Daemon-connected or embedded | Native (mesh), Bridge |
| Tier 1 | BGP-X Gateway v1 | Daemon-connected | Provider |
| Tier 2 | BGP-X Client Node | Embedded (client firmware) | Client, Sensor, Portable |
| Tier 3 | BGP-X Adapter | Host runs daemon | Client, Bridge |
| Software | OpenWrt Package | Daemon-connected | Standard, Native |

For details, see `hardware/README.md` and `hardware/adapter_spec.md`.
