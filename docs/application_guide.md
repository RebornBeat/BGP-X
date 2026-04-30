# BGP-X Application Developer Guide

**Version**: 0.1.0-draft

---

## 1. Three Application Types

**Type 1: Standard Application** — no integration required. Router-level transparent protection applies to all traffic matching routing policy. No code changes, no SDK.

**Type 2: BGP-X Native Application** — uses SDK to explicitly control path parameters, register native services, specify cross-domain paths, subscribe to events.

**Type 3: Configuration Client** — uses Control Socket (JSON-RPC 2.0) for daemon management. Not for routing traffic.

---

## 2. SDK Connection Model

The BGP-X SDK connects to the daemon. The SDK does NOT implement routing, handshakes, or crypto.

```
[Your Application]
        │  SDK API calls
        ▼
[BGP-X SDK Library]
        │  JSON-RPC 2.0 over Unix/TCP socket
        ▼
[BGP-X Daemon] (on router, same device, or embedded)
        │  UDP/mesh transport
        ▼
[BGP-X Overlay Network]
```

### Auto-Discovery Priority

1. `BGPX_SOCKET` environment variable
2. `/var/run/bgpx/sdk.sock` (system install)
3. `~/.bgpx/sdk.sock` (user install)
4. `127.0.0.1:7475` (TCP fallback)
5. Embedded mode (if compiled with `features = ["embedded"]`)

### LAN Device Connecting to Router Daemon

**Option A: SSH socket forwarding (recommended)**:
```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@192.168.1.1
```
```rust
let client = Client::connect(Path::new("/tmp/bgpx-router.sock")).await?;
```

**Option B: Router TCP endpoint** (configure in daemon):
```toml
[sdk_api]
tcp_listen = "192.168.1.1:7475"
tcp_auth_required = true
```

**Option C: Standard application mode** — if your app doesn't need path control, don't use the SDK. The router transparently protects traffic.

---

## 3. Opening a Stream — Clearnet

### Simple Connection

```rust
use bgpx_sdk::{Client, StreamConfig};

let client = Client::connect_auto().await?;

let mut stream = client.connect_stream(
    "example.com:443",
    StreamConfig::default(),
).await?;

// Implements AsyncRead + AsyncWrite (tokio)
stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;
let mut response = Vec::new();
stream.read_to_end(&mut response).await?;
```

### With Path Constraints (Single Domain)

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
        isolated_path: true,
        ..Default::default()
    },
).await?;
```

---

## 4. Cross-Domain Streams

### Clearnet Client → Mesh Island Service

No mesh hardware required. The bridge node handles all radio transmission.

```rust
use bgpx_sdk::{DomainSegmentConfig, DomainSegmentType, RoutingDomain, DomainTransition};

let stream = client.connect_stream_cross_domain(
    "bgpx-mesh://lima-district-1/service-id-hex",
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
            domain_transition: DomainTransition::Auto,
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

## 5. Registering a BGP-X Native Service

### Server Side

```rust
let listener = client.register_service(ServiceConfig {
    name: "my-private-api".to_string(),
    advertise: true,        // Publish ServiceID to unified DHT
    auth_required: false,
    max_connections: 100,
}).await?;

println!("Service ID: {}", listener.service_id().to_hex());

loop {
    let (mut stream, info) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(stream, info).await;
    });
}
```

### Client Side

```rust
let service_id = ServiceId::from_hex("a3f2b9c1d4e5f6a7...")?;

// Works from any routing domain — clearnet, mesh, satellite
let mut stream = client.connect_stream(
    &format!("bgpx://{}", service_id.to_hex()),
    Default::default(),
).await?;
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

## 6. Publishing a Mesh Island Service to the Unified DHT

When a service is registered with `advertise = true` on a mesh island node, the service record is stored in the local mesh DHT. The bridge node propagates this record to the unified internet DHT when online.

To manually verify a service is discoverable from clearnet:

```rust
// From clearnet client:
let status = client.get_island_status("your-island-id").await?;
println!("Island online: {}", status.online);
println!("Services: {}", status.registered_services);

// Discover bridge nodes to your island
let bridges = client.discover_bridges(
    RoutingDomain::Clearnet,
    RoutingDomain::Mesh { island_id: "your-island-id".into() },
).await?;
println!("Bridge nodes available: {}", bridges.len());

// Test cross-domain connectivity
let test = client.test_domain_connectivity(
    RoutingDomain::Mesh { island_id: "your-island-id".into() }
).await?;
println!("Reachable: {}", test.reachable);
println!("Best bridge latency: {}ms", test.best_bridge_latency_ms.unwrap_or(0));
```

---

## 7. Domain and Island Discovery

```rust
// List all known routing domains
let domains = client.list_domains().await?;
for domain in domains {
    println!("{:?}: relay_count={}, bridge_count={}",
        domain.domain_type, domain.active_relay_count, domain.bridge_node_count);
}

// Get status of a specific mesh island
let status = client.get_island_status("lima-district-1").await?;
println!("Island online: {}", status.online);
println!("Active relays: {}", status.active_relay_count);
println!("Bridge nodes: {}", status.bridge_node_count);

// Find bridge nodes between two domains
let bridges = client.discover_bridges(
    RoutingDomain::Clearnet,
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;

// Test connectivity
let result = client.test_domain_connectivity(
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;
```

---

## 8. Pool Management

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

## 9. Path Quality Events

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

## 10. ECH Requirements

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

If `require_ech = true` and destination doesn't publish ECH configuration, stream open fails with `SdkError::EchRequired`.

ECH applies only to clearnet exit streams. Mesh island native services use BGP-X onion encryption end-to-end — no TLS ClientHello SNI to protect.

---

## 11. Testing Without a Router

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

---

## 12. Platform Notes

### Android

BGP-X on Android uses VpnService API. SDK embedded mode recommended (background process restrictions). BGP-X mesh support for Android uses the Meshtastic USB adaptation layer if a Meshtastic device is connected.

### iOS

BGP-X on iOS uses Network Extension framework. Embedded mode. Background execution limited by iOS.

### Desktop

Recommended: system daemon (`bgpx-node`) + SDK connects via Unix socket. Same application code works with local daemon or remote router daemon.
