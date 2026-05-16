# BGP-X SDK Integration Guide

**Version**: 0.1.0-draft

---

## 1. Overview

This guide covers practical integration of the BGP-X SDK for Rust, Python, TypeScript, and Go. It assumes familiarity with the BGP-X architecture (`ARCHITECTURE.md`) and protocol (`protocol/protocol_spec.md`).

The BGP-X SDK connects your application to the BGP-X daemon. The SDK does not implement routing, crypto, or DHT — those live in the daemon. The SDK provides: stream opening, service registration, domain discovery, name resolution, event subscription, and path configuration.

---

## 2. Installation

### Rust

```toml
# Cargo.toml
[dependencies]
bgpx-sdk = "0.1"
tokio = { version = "1", features = ["full"] }
```

### Python

```bash
pip install bgpx-sdk
```

### TypeScript / Node.js

```bash
npm install @bgpx/sdk
```

### Go

```bash
go get github.com/bgpx/sdk-go
```

---

## 3. Connecting to the Daemon

### 3.1 Basic Connection

#### Rust

```rust
use bgpx_sdk::Client;

#[tokio::main]
async fn main() -> Result<(), bgpx_sdk::Error> {
    // Auto-discover daemon (checks env var, then standard socket paths)
    let client = Client::connect_auto().await?;
    println!("Connected. Daemon version: {}", client.version().await?);
    Ok(())
}
```

#### Manual Connection Options

```rust
use std::path::Path;
use std::net::SocketAddr;

// Unix domain socket (local daemon or SSH-forwarded)
let client = Client::connect(Path::new("/var/run/bgpx/sdk.sock")).await?;

// TCP (remote router daemon with TCP enabled)
let client = Client::connect_tcp("192.168.1.1:7475".parse::<SocketAddr>()?).await?;

// Embedded mode (daemon runs in-process; no external daemon needed)
let client = Client::embedded(Default::default()).await?;
```

#### Python

```python
import asyncio
from bgpx_sdk import Client

async def main():
    client = await Client.connect_auto()
    print(f"Connected. Daemon version: {await client.version()}")

asyncio.run(main())
```

#### TypeScript

```typescript
import { Client } from '@bgpx/sdk';

const client = await Client.connectAuto();
console.log(`Connected. Version: ${await client.version()}`);
```

### 3.2 Connecting a LAN Device to the Router Daemon

If your application runs on a LAN device and the BGP-X daemon is on the router, you have three connection options:

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

## 4. HTTP/2 Over BGP-X Streams

BGP-X native services (.bgpx) use HTTP/2 over BGP-X streams.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers — this is standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 5. Opening Streams

### 5.1 Simple Clearnet Stream

```rust
use bgpx_sdk::{Client, StreamConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

let client = Client::connect_auto().await?;

let mut stream = client.connect_stream(
    "example.com:443",
    StreamConfig::default(),
).await?;

// Stream implements AsyncRead + AsyncWrite (tokio)
stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;
let mut buf = vec![0u8; 4096];
let n = stream.read(&mut buf).await?;
println!("Received {} bytes", n);
```

### 5.2 .bgpx Service Stream

```rust
// By full ServiceID
let mut stream = client.connect_stream(
    "bgpx://a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx",
    StreamConfig::default(),
).await?;

// By short name (daemon resolves via Name Registry DHT)
let mut stream = client.connect_stream(
    "bgpx://community-news.bgpx/",
    StreamConfig::default(),
).await?;
```

### 5.3 Stream with Path Constraints

```rust
use bgpx_sdk::{PathConfig, SegmentConfig, SegmentConstraints, LoggingPolicy};

let mut stream = client.connect_stream_with_path(
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
                    exit_jurisdiction_blocklist: vec!["US".into(), "GB".into()],
                    ..Default::default()
                },
            },
        ],
        isolated_path: true,    // Dedicated path; not shared with other streams
        max_total_hops: None,   // No limit (N-hop unlimited)
        ..Default::default()
    },
    StreamConfig { require_ech: true, ..Default::default() },
).await?;
```

### 5.4 Double-Exit Architecture

For maximum isolation, use two independent exit pools in sequence:

```rust
let mut stream = client.connect_stream_with_path(
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
                is_exit: true,   // This is the clearnet exit
                ..Default::default()
            },
        ],
        ..Default::default()
    },
    StreamConfig::default(),
).await?;
```

The first exit node sees: traffic going to Exit Pool 2 (not destination).  
The second exit node sees: traffic from Exit Pool 1 (not original client).  
Neither exit sees both origin and destination.

### 5.5 Cross-Domain Stream (Clearnet to Mesh Island)

```rust
use bgpx_sdk::{DomainSegmentConfig, DomainSegmentType, RoutingDomain};

let mut stream = client.connect_stream_cross_domain(
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
    StreamConfig::default(),
).await?;
```

---

## 6. Identity Management

### 6.1 Ephemeral Identity (Default — Recommended)

Each session gets a fresh Ed25519 keypair. Activities across sessions cannot be linked at the protocol level.

```rust
// Default: ephemeral identity, no configuration needed
let client = Client::connect_auto().await?;
```

### 6.2 Persistent Identity (Optional — for authenticated services)

Use a stable identity when the service needs to recognize returning clients.

```rust
let client = Client::connect_with_config(ClientConfig {
    identity_mode: IdentityMode::Persistent {
        key_path: Path::new("/etc/myapp/bgpx_identity_key"),
    },
    ..Default::default()
}).await?;
```

**Warning**: Persistent identity creates a linkable fingerprint across sessions. Only use when authentication is necessary and the privacy trade-off is acceptable.

---

## 7. Registering a .bgpx Service

```rust
use bgpx_sdk::{ServiceConfig, ServiceType};

// Server: register service and accept connections
let listener = client.register_service(ServiceConfig {
    name: "my-forum".to_string(),
    advertise: true,            // Publish to unified DHT
    auth_required: false,
    max_connections: 500,
    service_type: ServiceType::Web,
    description: "Community forum".to_string(),
}).await?;

println!("Serving at: bgpx://{}.bgpx/", listener.service_id().to_hex());
println!("Short name:  bgpx://my-forum.bgpx/ (if name registered)");

// Register short name
let name_registry = client.name_registry().await?;
name_registry.register(
    "my-forum",
    service_keypair,
    Default::default(),
    std::time::Duration::from_secs(90 * 24 * 3600),
).await?;

// Accept connections
loop {
    let (mut stream, peer_info) = listener.accept().await?;
    println!("Connection from {} via {}", peer_info.path_type, peer_info.domain);
    tokio::spawn(async move {
        handle_client(stream).await;
    });
}
```

### Python Service Registration

```python
from bgpx_sdk import Client, ServiceConfig

client = await Client.connect_auto()

listener = await client.register_service(ServiceConfig(
    name="my-service",
    advertise=True,
    max_connections=100,
))

print(f"Serving at: bgpx://{listener.service_id.hex()}.bgpx/")

async for stream, peer_info in listener:
    asyncio.create_task(handle_client(stream))
```

---

## 8. ECH Status Check

To check ECH status after a stream is established:

```rust
let info = stream.info();
println!("ECH used: {}", info.ech_negotiated);
println!("ECH available at destination: {}", info.ech_available);
```

If `require_ech = true` and the destination doesn't support ECH, stream open fails with `SdkError::EchRequired`.

---

## 9. Latency-Adaptive Pattern

```rust
use bgpx_sdk::{EventType, BgpxEvent, LatencyClass};

// Subscribe to path events for a stream
let mut events = client.subscribe(&[
    EventType::StreamPaused,
    EventType::StreamResumed,
    EventType::PathDegraded,
    EventType::MeshIslandUnreachable,
    EventType::MeshIslandOnline,
]).await?;

let mut app_mode = AppMode::Standard;

tokio::spawn(async move {
    while let Some(event) = events.next().await {
        match event {
            BgpxEvent::PathUpdated { latency_class, .. } => {
                app_mode = match latency_class {
                    LatencyClass::MeshLoRa | LatencyClass::SatelliteGeo =>
                        AppMode::HighLatency,
                    _ =>
                        AppMode::Standard,
                };
            }
            BgpxEvent::StreamPaused { stream_id, reason, domain } => {
                // Queue outgoing; show waiting indicator
                pause_outgoing(stream_id);
            }
            BgpxEvent::StreamResumed { stream_id } => {
                // Flush queue
                flush_outgoing(stream_id);
            }
            BgpxEvent::MeshIslandUnreachable { island_id } => {
                show_offline_notice(&island_id);
            }
            BgpxEvent::MeshIslandOnline { island_id } => {
                hide_offline_notice(&island_id);
            }
            _ => {}
        }
    }
});
```

---

## 10. Domain Discovery

```rust
// List all known routing domains
let domains = client.list_domains().await?;
for domain in &domains {
    println!("{}: relays={}, bridges={}", domain.domain_id, domain.relay_count, domain.bridge_count);
}

// Get status of a specific mesh island
let status = client.get_island_status("lima-district-1").await?;
println!("Island online: {}, relays: {}, bridges: {}",
    status.online, status.active_relay_count, status.bridge_node_count);

// Discover bridge nodes for a domain pair
let bridges = client.discover_bridges(
    RoutingDomain::Clearnet,
    RoutingDomain::Mesh { island_id: "lima-district-1".into() },
).await?;
println!("Available bridges: {}", bridges.len());
for bridge in &bridges {
    println!("  Bridge: latency_ms={}, available={}", bridge.latency_ms, bridge.available);
}

// Test connectivity to a domain
let test = client.test_domain_connectivity(
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;
println!("Reachable: {}, best_bridge_latency: {:?}ms",
    test.reachable, test.best_bridge_latency_ms);
```

---

## 11. Name Registry

```rust
let name_registry = client.name_registry().await?;

// Resolve a short name to ServiceID
let service_id = name_registry.resolve("community-news").await?;
println!("community-news.bgpx → {}", service_id.to_hex());

// Register a name
name_registry.register(
    "my-service",
    &service_keypair,
    NameMetadata { description: "My service".into(), ..Default::default() },
    Duration::from_secs(90 * 24 * 3600),
).await?;

// List own registered names
let names = name_registry.list_own().await?;
for name in &names {
    println!("{}.bgpx → {} (expires: {:?})", name.name, name.service_id.to_hex(), name.expires_at);
}

// Withdraw a name
name_registry.withdraw("my-service", &service_keypair).await?;
```

---

## 12. Error Handling

```rust
use bgpx_sdk::{SdkError, BridgeError};

match stream_result {
    Ok(stream) => { /* use stream */ }

    Err(SdkError::ServiceNotFound(service_id)) =>
        eprintln!("Service {} not found in DHT", service_id.to_hex()),

    Err(SdkError::PathConstructionFailed { reason }) =>
        eprintln!("Path construction failed: {}", reason),

    Err(SdkError::DomainBridgeUnavailable { from, to }) =>
        eprintln!("No bridge available from {:?} to {:?}", from, to),

    Err(SdkError::MeshIslandUnreachable { island_id }) =>
        eprintln!("Island {} is unreachable", island_id),

    Err(SdkError::StreamPaused { reason, domain }) =>
        eprintln!("Stream paused: {} in domain {}", reason, domain),

    Err(SdkError::EchRequired) =>
        eprintln!("ECH required but not available at destination"),

    Err(SdkError::DaemonNotConnected) =>
        eprintln!("BGP-X daemon not reachable; is bgpx-node running?"),

    Err(e) =>
        eprintln!("Unexpected error: {}", e),
}
```

---

## 13. Testing

### 13.1 Unit Testing with Embedded Mode

```rust
#[tokio::test]
async fn test_stream_opens() {
    let client = Client::embedded(ClientConfig {
        data_dir: tempfile::tempdir().unwrap().into_path(),
        mesh_enabled: false,
        ..Default::default()
    }).await.unwrap();

    // Test path construction (uses simulated network in embedded mode)
    let result = client.test_path_construction("example.com:443").await;
    assert!(result.is_ok());
}
```

### 13.2 Simulating LoRa Latency

```rust
let client = Client::embedded(ClientConfig {
    simulate_latency: Some(SimulatedLatency {
        class: LatencyClass::MeshLoRa,
        per_hop_ms: 1000,
        hop_count: 5,
    }),
    ..Default::default()
}).await?;

// Your application's LoRa-tolerant behavior is tested against
// 5-second round-trip times; verify it doesn't time out or error
```

### 13.3 Integration Testing Against Local Daemon

```bash
# Start test daemon (isolated from production)
bgpx-node --config test_config.toml --data-dir /tmp/bgpx-test &

# Run integration tests
BGPX_SOCKET=/tmp/bgpx-test/sdk.sock cargo test --test integration
```

---

## 14. Performance Considerations

### 14.1 Stream Reuse

BGP-X path construction takes 100-500ms (clearnet) or 2-10 seconds (cross-domain via LoRa). Reuse streams for multiple requests to the same service:

```rust
// BAD: New path for each request
for item in items {
    let mut stream = client.connect_stream("bgpx://service.bgpx/", Default::default()).await?;
    send_item(&mut stream, item).await?;
}

// GOOD: HTTP/2 multiplexing over one stream
let mut stream = client.connect_stream("bgpx://service.bgpx/", Default::default()).await?;
let http2 = Http2Client::new(stream);
for item in items {
    http2.post("/api/item", item).await?;  // HTTP/2 concurrent streams
}
```

### 14.2 Connection Pooling

```rust
use bgpx_sdk::ConnectionPool;

let pool = ConnectionPool::new(client.clone(), PoolConfig {
    max_connections_per_service: 3,
    idle_timeout: Duration::from_secs(300),
});

// Pool manages stream lifecycle; reuses connections
let mut stream = pool.acquire("bgpx://service.bgpx/").await?;
```

### 14.3 Bandwidth Awareness

```rust
// Check effective bandwidth before sending large payloads
let path_info = stream.path_info()?;
if path_info.latency_class == LatencyClass::MeshLoRa {
    // Compress aggressively; split large payloads into chunks
    let compressed = brotli_compress(&large_payload)?;
    send_chunked(&mut stream, &compressed, 200).await?;  // 200-byte LoRa chunks
}
```

---

## 15. Geographic Plausibility Note

Geographic plausibility scoring is an **optional reputation signal** in BGP-X. Jurisdiction declaration is **opt-in** — nodes are NOT required to declare their location.

From the SDK perspective:
- You can filter exit nodes by jurisdiction using `exit_jurisdiction_blocklist` or `exit_jurisdiction_allowlist`
- A node's declared jurisdiction (if any) is visible in its advertisement metadata
- You cannot reliably infer jurisdiction from IP address alone (especially for satellite-connected nodes)

---

## 16. Web3 / Blockchain Access

Web3 (Ethereum, Solana, IPFS, etc.) is **application layer** traffic, not a separate routing domain. From BGP-X's perspective, connecting to an Ethereum RPC endpoint or IPFS node is standard clearnet traffic.

- Access Web3 services via clearnet exit nodes
- Use pool constraints to route through specific jurisdictions if desired
- No special SDK configuration needed for Web3 traffic
