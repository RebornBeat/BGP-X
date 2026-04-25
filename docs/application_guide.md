# BGP-X Application Developer Guide

This document explains how to build applications that use BGP-X — whether as standard applications that benefit from transparent router protection, or as BGP-X native applications that explicitly control overlay capabilities.

---

## The Three Application Types

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
- You need to build a double-exit architecture for maximum isolation
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

## SDK Connection Model

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
        │  UDP 7474 (or mesh transport)
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

---

## Connecting a LAN Device to the Router Daemon

If your application runs on a LAN device and the BGP-X daemon is on the router:

### Option A: SSH Socket Forwarding (Recommended)

```bash
# On the LAN device, in a terminal:
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@192.168.1.1

# Keep this running while your app needs the connection
```

In your application:
```rust
let client = Client::connect(Path::new("/tmp/bgpx-router.sock")).await?;
```

### Option B: Router TCP Endpoint

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

### Option C: Standard Application Mode (No SDK Needed)

If your application doesn't need path control or service registration, consider whether you actually need the SDK:

- Just connect to the internet normally
- The router's BGP-X overlay protects your traffic transparently
- No code changes required

---

## Opening a Stream to a Clearnet Destination

The most common SDK use case: connect to a clearnet destination with explicit path constraints.

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
use bgpx_sdk::{Client, PathConfig, StreamConfig, SegmentConfig, LoggingPolicy};

let client = Client::connect_auto().await?;

// High-security connection: 6 hops, no Five Eyes, ECH required, private exit
let stream = client.connect_stream_with_path(
    "whistleblower-platform.example.com:443",
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
                ..Default::default()
            },
        ],
        exit_jurisdiction_blacklist: vec![
            "US".into(), "GB".into(), "AU".into(), "CA".into(), "NZ".into(),
        ],
        exit_logging_policy: Some(LoggingPolicy::None),
        ..Default::default()
    },
    StreamConfig {
        require_ech: true,
        prefer_ech: true,
        isolated_path: true,  // This stream gets its own path, not shared
        ..Default::default()
    },
).await?;
```

### Double-Exit Architecture

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

## Registering a BGP-X Native Service

BGP-X native services are reachable through the overlay using a ServiceID (public key hash) — no IP address or domain name required. They can be accessed from anywhere in the BGP-X network without clearnet exposure.

### Server Side

```rust
use bgpx_sdk::{Client, ServiceConfig};

let client = Client::connect_auto().await?;

let listener = client.register_service(ServiceConfig {
    name: "my-private-api".to_string(),
    advertise: true,        // Publish ServiceID to DHT for discoverability
    auth_required: false,   // Whether clients must present authenticated identity
    max_connections: 100,
}).await?;

// Print ServiceID so clients can connect
println!("Service reachable at: bgpx://{}", listener.service_id().to_hex());

// Accept incoming connections
loop {
    let (mut stream, info) = listener.accept().await?;

    tokio::spawn(async move {
        // info.client_id is Some(...) only if client presents authenticated identity
        // info.path_hops tells you how many hops the client used
        handle_connection(stream, info).await;
    });
}
```

### Client Side

```rust
let service_id = ServiceId::from_hex("a3f2b9c1d4e5f6a7...")?;

let mut stream = client.connect_stream(
    &format!("bgpx://{}", service_id.to_hex()),
    StreamConfig::default(),
).await?;

// Use like a normal TCP connection
stream.write_all(b"hello service").await?;
```

---

## Using Pools

### Adding a Private Pool

```rust
use bgpx_sdk::{Client, Pool, PoolType};

let client = Client::connect_auto().await?;

client.add_pool(Pool {
    pool_id: "my-private-exit".to_string(),
    pool_type: PoolType::Private,
    curator_public_key: my_curator_public_key,
    members: vec![
        "node_id_hex_1".parse()?,
        "node_id_hex_2".parse()?,
    ],
    ..Default::default()
}).await?;
```

### Listing Available Pools

```rust
let pools = client.list_pools().await?;
for pool in pools {
    println!("{}: {} members, type {:?}", pool.pool_id, pool.member_count, pool.pool_type);
}
```

---

## Handling Path Quality Events

```rust
let mut events = client.subscribe(&[
    EventType::PathBuilt,
    EventType::PathDegraded,
    EventType::PathRebuilt,
    EventType::PoolMemberOffline,
]).await?;

while let Some(event) = events.next().await {
    match event {
        BgpxEvent::PathDegraded { quality_score, reason } => {
            eprintln!("Path quality degraded: {:.2} ({})", quality_score, reason);
            // Application may want to pause non-critical operations
        }
        BgpxEvent::PathRebuilt { hops, reason } => {
            eprintln!("Path rebuilt: {} hops, reason: {}", hops, reason);
            // Resume operations
        }
        BgpxEvent::PoolMemberOffline { pool_id } => {
            eprintln!("Pool {} lost a member", pool_id);
        }
        _ => {}
    }
}
```

---

## Identity Management

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

## Testing Without a Router

For development and testing:

### Local Daemon

Start a local BGP-X daemon on your development machine:

```bash
bgpx-node --config dev_config.toml
```

Your application connects to the local daemon automatically via auto-discovery.

### Embedded Mode

For unit tests or integration tests where a daemon process is inconvenient:

```rust
// In Cargo.toml: bgpx-sdk = { version = "0.1", features = ["embedded"] }

let client = Client::embedded(ClientConfig {
    data_dir: temp_dir(),
    ..Default::default()
}).await?;

// Same API, full stack runs in-process
```

---

## Platform-Specific Notes

### Android

BGP-X on Android uses the `VpnService` API to create a TUN-equivalent interface. The SDK embedded mode is recommended since background daemon processes are restricted on Android.

```
bgpx-sdk = { version = "0.1", features = ["embedded", "android"] }
```

Battery optimization: embedded mode with mesh transport disabled and cover traffic disabled is recommended for battery-sensitive deployments.

### iOS

BGP-X on iOS uses the `Network Extension` framework. Background execution is restricted by iOS. Embedded mode with the BGP-X daemon configured for minimal resource use.

```
bgpx-sdk = { version = "0.1", features = ["embedded", "ios"] }
```

The iOS app must request the `com.apple.developer.networking.networkextension` entitlement.

### Desktop (Linux, macOS, Windows)

Recommended: install bgpx-node as a system daemon. SDK connects via Unix socket (Linux, macOS) or named pipe (Windows planned).

On macOS, TUN interface creation requires Administrator privileges (utun driver).

On Windows: WinTUN-based TUN support planned. SDK can connect to a locally running daemon.

---

## ECH Requirements in Applications

To require ECH for a specific stream:

```rust
StreamConfig {
    require_ech: true,   // Fail stream if destination doesn't support ECH
    prefer_ech: true,    // Use ECH when available even if not required (default: true)
    ..Default::default()
}
```

To check ECH status after stream is established:

```rust
let stream_info = stream.info();
println!("ECH used: {}", stream_info.ech_negotiated);
println!("ECH reason: {:?}", stream_info.ech_status);
```

If `require_ech = true` and the destination does not publish ECH configuration, the stream open fails with `SdkError::EchRequired`. The application should either fall back to a non-ECH connection (if the security trade-off is acceptable) or inform the user that the destination does not support ECH.
