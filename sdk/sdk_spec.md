# BGP-X SDK Specification

**Version**: 0.1.0-draft
**License**: MIT

---

## 1. SDK Architecture — Router-Centric Model

### 1.1 What the SDK Is

The BGP-X SDK is a client library that connects to the BGP-X daemon. The SDK provides a high-level API for BGP-X native applications.

**The SDK does NOT**:
- Build overlay paths
- Perform handshakes with relay nodes
- Encrypt or decrypt onion layers
- Maintain a DHT connection
- Manage session keys
- Route traffic for other nodes

All of these are done by the daemon. The SDK translates high-level API calls into daemon protocol messages (JSON-RPC 2.0 over Unix socket or TCP).

### 1.2 Connection Model

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
        │  UDP 7474 (clearnet) / WiFi 802.11s / LoRa / BLE
        ▼
[BGP-X Overlay Network]
```

The daemon can be:
- On a **network router** serving your LAN (BGP-X Router v1)
- Running **locally** on the same device
- **Embedded in-process** by the SDK (mobile / constrained environments)

### 1.3 SDK Auto-Discovery

```rust
// Priority order for daemon discovery:
// 1. BGPX_SOCKET environment variable
// 2. /var/run/bgpx/sdk.sock (system install)
// 3. ~/.bgpx/sdk.sock (user install)
// 4. 127.0.0.1:7475 (TCP fallback)
// 5. Embedded mode (if compiled with embedded feature)

let client = Client::connect_auto().await?;
```

### 1.4 LAN Device Connecting to Router Daemon

If your application runs on a LAN device and the BGP-X daemon is on the router:

**Option A: SSH socket forwarding (recommended)**

```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@192.168.1.1
```

```rust
let client = Client::connect(Path::new("/tmp/bgpx-router.sock")).await?;
```

**Option B: Router TCP endpoint**

```toml
# On router daemon config:
[sdk_api]
tcp_listen = "192.168.1.1:7475"
tcp_auth_required = true
```

```rust
let client = Client::connect_tcp_with_auth(
    "192.168.1.1:7475",
    "your-auth-token-here"
).await?;
```

**Option C: Standard application (no SDK needed)**

If your application doesn't need path control, service registration, or event subscriptions — don't use the SDK. The router transparently protects your traffic through the TUN interface. No code changes required.

---

## 2. Language Targets

| Language | Status | Implementation |
|---|---|---|
| Rust | Primary | Native (reference implementation) |
| Python | Planned | PyO3 bindings |
| JavaScript/TypeScript | Planned | Napi-rs bindings |
| Go | Planned | CGo bindings |
| C | Planned | C FFI layer |

All languages expose the same logical API, adapted to language idioms. Rust is used as the reference notation in this specification.

---

## 3. Core API

### 3.1 Client

The `Client` is the top-level SDK object. It represents a connection to the running BGP-X daemon.

```rust
pub struct Client { /* opaque */ }

impl Client {
    // Connection methods
    pub async fn connect_auto() -> Result<Self, SdkError>;
    pub async fn connect(socket_path: &Path) -> Result<Self, SdkError>;
    pub async fn connect_tcp(addr: &str) -> Result<Self, SdkError>;
    pub async fn connect_tcp_with_auth(addr: &str, auth_token: &str) -> Result<Self, SdkError>;

    // Status
    pub async fn status(&self) -> Result<DaemonStatus, SdkError>;

    // Stream operations (single-domain)
    pub async fn connect_stream(
        &self,
        destination: &str,
        config: StreamConfig,
    ) -> Result<BgpxStream, SdkError>;

    pub async fn connect_stream_with_path(
        &self,
        destination: &str,
        path_config: PathConfig,
        stream_config: StreamConfig,
    ) -> Result<BgpxStream, SdkError>;

    // Stream operations (cross-domain)
    pub async fn connect_stream_cross_domain(
        &self,
        destination: &str,
        domain_segments: Vec<DomainSegmentConfig>,
        stream_config: StreamConfig,
    ) -> Result<BgpxStream, SdkError>;

    // Native services
    pub async fn register_service(
        &self,
        config: ServiceConfig,
    ) -> Result<BgpxListener, SdkError>;

    // Path quality
    pub async fn path_quality(&self) -> Result<PathQuality, SdkError>;
    pub async fn build_isolated_path(
        &self,
        config: PathConfig,
    ) -> Result<IsolatedPath, SdkError>;

    // Event subscription
    pub async fn subscribe(
        &self,
        events: &[EventType],
    ) -> Result<EventStream, SdkError>;

    // Pool management
    pub async fn add_pool(&self, pool: Pool) -> Result<(), SdkError>;
    pub async fn remove_pool(&self, pool_id: &str) -> Result<(), SdkError>;
    pub async fn list_pools(&self) -> Result<Vec<PoolSummary>, SdkError>;
    pub async fn get_pool_info(&self, pool_id: &str) -> Result<PoolInfo, SdkError>;

    // Domain management
    pub async fn list_domains(&self) -> Result<Vec<DomainInfo>, SdkError>;
    pub async fn discover_bridges(
        &self,
        from_domain: RoutingDomain,
        to_domain: RoutingDomain,
    ) -> Result<Vec<BridgeNodeInfo>, SdkError>;
    pub async fn get_island_status(
        &self,
        island_id: &str,
    ) -> Result<MeshIslandStatus, SdkError>;
    pub async fn test_domain_connectivity(
        &self,
        target_domain: RoutingDomain,
    ) -> Result<ConnectivityTestResult, SdkError>;

    // Route testing
    pub async fn test_route(&self, destination: &str) -> Result<RouteTestResult, SdkError>;

    // Identity management
    pub async fn get_identity(&self) -> Result<IdentityInfo, SdkError>;
    pub async fn rotate_identity(&self) -> Result<ServiceId, SdkError>;

    // Domain-aware service resolution
    pub async fn resolve_bgpx_name(&self, name: &str) -> Result<ServiceId, SdkError>;
}
```

### 3.2 BgpxStream

`BgpxStream` implements `AsyncRead + AsyncWrite` (Rust Tokio) for ordered streams, providing the same interface as a `TcpStream`.

```rust
pub struct BgpxStream { /* opaque */ }

impl BgpxStream {
    pub fn stream_id(&self) -> StreamId;
    pub fn path_info(&self) -> PathInfo;
    pub fn stats(&self) -> StreamStats;
    pub fn info(&self) -> StreamInfo;
    pub async fn close(self) -> Result<(), SdkError>;
    pub fn reset(self);
}

impl AsyncRead for BgpxStream { /* ... */ }
impl AsyncWrite for BgpxStream { /* ... */ }
```

**Usage example**:

```rust
use bgpx_sdk::Client;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::connect_auto().await?;

    let mut stream = client.connect_stream(
        "example.com:443",
        StreamConfig::default(),
    ).await?;

    // Use like TcpStream
    stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;

    let mut response = Vec::new();
    stream.read_to_end(&mut response).await?;

    println!("{}", String::from_utf8_lossy(&response));
    Ok(())
}
```

### 3.3 BgpxListener (Native Services)

`BgpxListener` allows an application to register as a BGP-X native service — reachable through the overlay by its ServiceID without a clearnet address.

```rust
pub struct BgpxListener { /* opaque */ }

impl BgpxListener {
    pub fn service_id(&self) -> ServiceId;
    pub async fn accept(&self) -> Result<(BgpxStream, ConnectionInfo), SdkError>;
    pub async fn close(self) -> Result<(), SdkError>;
}

pub struct ConnectionInfo {
    pub client_id: Option<PublicKey>,  // Only if client presented authenticated identity
    pub path_hops: u8,                 // Number of hops (no node identities)
}
```

**Usage example — registering a BGP-X native service**:

```rust
let client = Client::connect_auto().await?;

let listener = client.register_service(ServiceConfig {
    name: "my-private-api".into(),
    advertise: true,
    auth_required: false,
    max_connections: 100,
}).await?;

println!("Service reachable at: {}", listener.service_id().to_hex());

loop {
    let (stream, info) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(stream, info).await;
    });
}
```

---

## 4. Configuration Types

### 4.1 RoutingDomain

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum RoutingDomain {
    Clearnet,                                           // BGP-routed internet
    BgpxOverlay,                                        // BGP-X overlay (virtual)
    Mesh { island_id: String },                         // Named mesh island
    LoraRegional { region_id: String },                 // LoRa regional zone
    Auto,                                               // Path construction chooses
}

impl RoutingDomain {
    pub fn to_domain_id(&self) -> DomainId;
    pub fn from_domain_id(id: DomainId) -> Self;
    pub fn is_mesh(&self) -> bool { matches!(self, Self::Mesh { .. }) }
}
```

**Important**: Satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) are **clearnet** domain, NOT a separate domain. A node with satellite WAN uses `RoutingDomain::Clearnet`.

### 4.2 DomainTransition

```rust
pub enum DomainTransition {
    Auto,                                           // Path construction finds bridge automatically
    Via { bridge_pool: String },                    // Use specific pool of bridge nodes
    ViaNode { node_id: String },                    // Use specific bridge node
}
```

### 4.3 DomainSegmentConfig

For cross-domain path construction.

```rust
pub struct DomainSegmentConfig {
    pub segment_type: DomainSegmentType,
    pub domain: Option<RoutingDomain>,
    pub pool: String,
    pub hops: u8,
    pub is_exit: bool,
    pub domain_transition: DomainTransition,
    pub constraints: SegmentConstraints,
}

pub enum DomainSegmentType {
    Relay,                                          // Standard relay hop
    Bridge,                                         // Domain bridge node
    BridgeTo { target_domain: RoutingDomain },      // Bridge to specific domain
}

impl Default for DomainSegmentConfig {
    fn default() -> Self {
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Clearnet),
            pool: "bgpx-default".into(),
            hops: 2,
            is_exit: false,
            domain_transition: DomainTransition::Auto,
            constraints: SegmentConstraints::default(),
        }
    }
}
```

### 4.4 SegmentConfig

For single-domain pool-based paths (backward compatible with earlier versions).

```rust
pub struct SegmentConfig {
    pub pool: String,
    pub hops: u8,
    pub is_exit: bool,
    pub constraints: SegmentConstraints,
}

pub struct SegmentConstraints {
    pub require_ech: bool,
    pub exit_logging_policy: Option<LoggingPolicy>,
    pub exit_jurisdiction_blacklist: Vec<String>,
    pub exit_jurisdiction_whitelist: Vec<String>,
    pub min_reputation_score: Option<f32>,
    pub min_node_uptime: Option<f32>,
}

pub enum LoggingPolicy {
    None,       // No logs
    Metadata,   // Timing/volume only
    Any,        // Full logging (avoid for privacy-sensitive paths)
}
```

### 4.5 PathConfig

```rust
pub struct PathConfig {
    // Cross-domain path — takes priority if specified
    pub domain_segments: Vec<DomainSegmentConfig>,

    // Single-domain pool path — backward compatible
    pub segments: Vec<SegmentConfig>,

    pub fallback_to_default: bool,               // Fall back to default pool if segment insufficient
    pub enforce_pool_isolation: bool,            // Strict cross-segment operator diversity

    // N-hop unlimited
    pub max_total_hops: Option<u16>,             // Optional cap; NO protocol default maximum

    // Domain preferences
    pub preferred_domains: Vec<RoutingDomain>,
    pub avoid_domains: Vec<RoutingDomain>,

    pub cover_traffic: bool,

    // Single-domain shortcuts (for simple cases)
    pub hops: Option<u8>,
    pub min_hops: Option<u8>,
    pub min_reputation_score: Option<f32>,
    pub exit_jurisdiction_blacklist: Vec<String>,
    pub exit_jurisdiction_whitelist: Vec<String>,
    pub exit_logging_policy: Option<LoggingPolicy>,
    pub max_latency_ms: Option<u32>,
    pub min_regions: Option<u8>,
}

impl Default for PathConfig {
    fn default() -> Self {
        PathConfig {
            domain_segments: vec![],
            segments: vec![SegmentConfig {
                pool: "bgpx-default".into(),
                hops: 4,
                is_exit: true,
                constraints: SegmentConstraints {
                    require_ech: false,
                    exit_logging_policy: Some(LoggingPolicy::None),
                    exit_jurisdiction_blacklist: vec![],
                    exit_jurisdiction_whitelist: vec![],
                    min_reputation_score: Some(60.0),
                    min_node_uptime: Some(85.0),
                },
            }],
            fallback_to_default: true,
            enforce_pool_isolation: true,
            max_total_hops: None,                // No protocol limit
            preferred_domains: vec![],
            avoid_domains: vec![],
            cover_traffic: false,
            hops: None,
            min_hops: Some(3),
            min_reputation_score: Some(60.0),
            exit_jurisdiction_blacklist: vec![],
            exit_jurisdiction_whitelist: vec![],
            exit_logging_policy: Some(LoggingPolicy::None),
            max_latency_ms: None,
            min_regions: Some(2),
        }
    }
}
```

### 4.6 StreamConfig

```rust
pub struct StreamConfig {
    pub stream_type: StreamType,                 // Ordered (default) or Datagram
    pub priority: u8,                            // 0-7, default 3 (0 = highest)
    pub idle_timeout_seconds: u64,               // 0 = no timeout, default 300
    pub recv_buffer_size: usize,                 // Default 1 MB
    pub isolated_path: bool,                     // Exclusive path for this stream
    pub path_config: Option<PathConfig>,         // Path config for isolated path
    pub require_ech: bool,                       // Fail if ECH not available at exit
    pub prefer_ech: bool,                        // Use ECH when available (default: true)
}

impl Default for StreamConfig {
    fn default() -> Self {
        StreamConfig {
            stream_type: StreamType::Ordered,
            priority: 3,
            idle_timeout_seconds: 300,
            recv_buffer_size: 1_048_576,
            isolated_path: false,
            path_config: None,
            require_ech: false,
            prefer_ech: true,
        }
    }
}

pub enum StreamType {
    Ordered,     // TCP-like reliable stream
    Datagram,    // UDP-like unreliable
}
```

### 4.7 ServiceConfig

```rust
pub struct ServiceConfig {
    pub name: String,
    pub advertise: bool,                         // Publish ServiceID to DHT
    pub auth_required: bool,                    // Require client authenticated identity
    pub max_connections: usize,
    pub mesh_endpoint: Option<MeshEndpointConfig>,  // Register with mesh endpoint
}

pub struct MeshEndpointConfig {
    pub island_id: String,
    pub transports: Vec<MeshTransportType>,     // wifi_mesh, lora, ble
}
```

---

## 5. Response Types

### 5.1 PathInfo

```rust
pub struct PathInfo {
    pub hops: u8,
    pub age_seconds: u64,
    pub estimated_rtt_ms: u32,
    pub quality_score: f32,                      // 0.0-1.0
    pub exit_jurisdiction: String,
    pub exit_logging_policy: LoggingPolicy,
    pub pool_segments: Vec<PoolSegmentInfo>,
    pub domain_segments: Vec<DomainSegmentInfo>,
    pub total_domains: u8,
    pub is_cross_domain: bool,
    pub domain_bridge_count: u8,
    pub mesh_hops_count: u8,
    pub satellite_hops_count: u8,
}

pub struct PoolSegmentInfo {
    pub pool_name: String,
    pub hops: u8,
}

pub struct DomainSegmentInfo {
    pub domain: RoutingDomain,
    pub hops: u8,
    pub estimated_latency_ms: u32,
    pub is_bridge: bool,
    pub segment_quality: f32,
}
```

### 5.2 StreamInfo

```rust
pub struct StreamInfo {
    pub stream_id: StreamId,
    pub ech_negotiated: bool,                    // ECH was actually used
    pub ech_available: bool,                     // Destination supports ECH
    pub path_hops: u8,
    pub is_cross_domain: bool,
    pub domain_count: u8,
}
```

### 5.3 DaemonStatus

```rust
pub struct DaemonStatus {
    pub state: DaemonState,
    pub version: String,
    pub uptime_seconds: u64,
    pub active_paths: u32,
    pub active_streams: u32,
    pub node_database_size: u32,
    pub mesh_active: bool,
    pub pt_enabled: bool,
    pub ech_capable: bool,                       // For exit nodes
    pub active_domains: Vec<RoutingDomain>,
}

pub enum DaemonState {
    Bootstrap,
    Active,
    Drain,
}
```

---

## 6. Event Subscription

```rust
pub struct EventStream { /* opaque */ }

impl EventStream {
    pub async fn next(&mut self) -> Option<BgpxEvent>;
}

pub enum BgpxEvent {
    // Path events
    PathBuilt { hops: u8, build_time_ms: u32, is_cross_domain: bool },
    PathDegraded { quality_score: f32, reason: DegradationReason },
    PathRebuilt { hops: u8, reason: RebuildReason, is_cross_domain: bool },

    // Stream events
    StreamOpened { stream_id: StreamId },
    StreamClosed { stream_id: StreamId, reason: CloseReason },

    // Node events
    NodeBlacklisted,                             // No node identifier
    GeoPlausibilityAlert,                       // No node identifier

    // Daemon events
    DaemonStateChanged { new_state: DaemonState },

    // Pool events
    PoolMemberOffline { pool_id: String },

    // ECH events
    EchFailed { stream_id: StreamId },
    EchFallback { stream_id: StreamId },

    // Cross-domain events
    DomainBridgeOnline { from_domain: String, to_domain: String },
    DomainBridgeOffline { from_domain: String, to_domain: String },
    MeshIslandDiscovered { island_id: String },
    MeshIslandUnreachable { island_id: String },
    MeshIslandOnline { island_id: String },
    CrossDomainPathBuilt { domains: Vec<String>, bridge_count: u8 },

    // Stream pause/resume (for high-latency paths)
    StreamPaused { stream_id: StreamId, reason: String, estimated_resume_secs: Option<u64> },
    StreamResumed { stream_id: StreamId },
}

pub enum DegradationReason {
    HighLatency,
    PacketLoss,
    LowThroughput,
    NodeUnreachable,
}

pub enum RebuildReason {
    PathExpired,
    QualityDegraded,
    NodeBlacklisted,
    ManualRequest,
}

pub enum CloseReason {
    Normal,
    RemoteError,
    DestinationUnreachable,
    ExitPolicyDenied,
    Timeout,
    ProtocolError,
    EchRequiredNotSupported,
    EchNegotiationFailed,
    DomainBridgeUnavailable,
    MeshIslandUnreachable,
}
```

---

## 7. Error Handling

```rust
pub enum SdkError {
    // Connection errors
    DaemonNotFound,
    ConnectionFailed(IoError),
    InvalidDaemonState(DaemonState),

    // Path errors
    PathConstructionFailed(String),
    PoolNotFound(String),
    PoolInsufficientNodes(String),

    // Stream errors
    StreamReset,
    StreamClosed,
    Timeout,

    // Destination errors
    ExitPolicyDenied,
    DestinationUnreachable(String),

    // ECH errors
    EchRequired,                                // require_ech=true but ECH not available
    EchNegotiationFailed,

    // Mesh/domain errors
    MeshTransportUnavailable,
    DomainNotFound(String),
    DomainBridgeUnavailable { from: String, to: String },
    MeshIslandUnreachable(String),
    NoCrossDomainPath,
    DomainScopeMismatch { required: String, found: String },
    StreamPaused { reason: String, estimated_resume_secs: Option<u64> },

    // Configuration errors
    InvalidConfig(String),

    // Internal errors
    Internal(String),
}
```

---

## 8. Usage Examples

### 8.1 Simple HTTP Request via BGP-X

```rust
let client = Client::connect_auto().await?;

let mut stream = client.connect_stream("example.com:443", Default::default()).await?;

stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;

let mut response = Vec::new();
stream.read_to_end(&mut response).await?;

println!("{}", String::from_utf8_lossy(&response));
```

### 8.2 High-Security Stream with Custom Path

```rust
let client = Client::connect_auto().await?;

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
                    exit_jurisdiction_blacklist: vec![
                        "US".into(),
                        "GB".into(),
                        "AU".into(),
                        "CA".into(),
                        "NZ".into(),  // Five Eyes
                    ],
                    ..Default::default()
                },
            },
        ],
        ..Default::default()
    },
    StreamConfig {
        isolated_path: true,
        priority: 0,
        require_ech: true,
        ..Default::default()
    },
).await?;
```

### 8.3 Double-Exit Architecture

Two independent exit pools in sequence so no single exit sees both origin and destination:

```rust
let client = Client::connect_auto().await?;

let stream = client.connect_stream_with_path(
    "destination.example.com:443",
    PathConfig {
        segments: vec![
            SegmentConfig { pool: "bgpx-default".into(), hops: 2, is_exit: false, ..Default::default() },
            SegmentConfig { pool: "exit-tier-1".into(), hops: 1, is_exit: false, ..Default::default() },
            SegmentConfig { pool: "exit-tier-2".into(), hops: 1, is_exit: true, ..Default::default() },
        ],
        ..Default::default()
    },
    Default::default(),
).await?;

// Exit Pool 1 sees: traffic going to Exit Pool 2 (not destination)
// Exit Pool 2 sees: traffic from Exit Pool 1 (not client)
```

### 8.4 Cross-Domain: Clearnet Client → Mesh Service

Clearnet client with no mesh hardware reaching a mesh island service:

```rust
let client = Client::connect_auto().await?;

let stream = client.connect_stream_cross_domain(
    "bgpx://<service_id_hex>",               // BGP-X native service in mesh island
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
                target_domain: RoutingDomain::Mesh { island_id: "lima-district-1".into() }
            },
            domain_transition: DomainTransition::Auto,
            ..Default::default()
        },
        DomainSegmentConfig {
            segment_type: DomainSegmentType::Relay,
            domain: Some(RoutingDomain::Mesh { island_id: "lima-district-1".into() }),
            hops: 2,
            ..Default::default()
        },
    ],
    Default::default(),
).await?;
```

### 8.5 Cross-Domain: Clearnet → Mesh → Clearnet (Three Domains)

```rust
let client = Client::connect_auto().await?;

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

### 8.6 N-Hop Unlimited (20 Hops, 3 Domains)

```rust
let client = Client::connect_auto().await?;

let stream = client.connect_stream_with_path(
    "destination.example.com:443",
    PathConfig {
        domain_segments: vec![
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Clearnet),
                hops: 6,
                ..Default::default()
            },
            DomainSegmentConfig {
                segment_type: DomainSegmentType::BridgeTo {
                    target_domain: RoutingDomain::Mesh { island_id: "relay-island".into() }
                },
                ..Default::default()
            },
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Mesh { island_id: "relay-island".into() }),
                hops: 7,
                ..Default::default()
            },
            DomainSegmentConfig {
                segment_type: DomainSegmentType::BridgeTo {
                    target_domain: RoutingDomain::Clearnet,
                },
                ..Default::default()
            },
            DomainSegmentConfig {
                segment_type: DomainSegmentType::Relay,
                domain: Some(RoutingDomain::Clearnet),
                pool: "private-exits".into(),
                hops: 5,
                is_exit: true,
                ..Default::default()
            },
        ],
        max_total_hops: None,                   // No protocol limit
        ..Default::default()
    },
    Default::default(),
).await?;
```

### 8.7 Register BGP-X Native Service

```rust
let client = Client::connect_auto().await?;

let listener = client.register_service(ServiceConfig {
    name: "my-private-api".to_string(),
    advertise: true,
    auth_required: false,
    max_connections: 100,
    mesh_endpoint: None,                        // Clearnet-accessible service
}).await?;

println!("Service ID: {}", listener.service_id().to_hex());

loop {
    let (stream, info) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(stream, info).await;
    });
}
```

### 8.8 Register Mesh Island Service

```rust
let client = Client::connect_auto().await?;

let listener = client.register_service(ServiceConfig {
    name: "mesh-local-api".to_string(),
    advertise: true,
    auth_required: false,
    max_connections: 50,
    mesh_endpoint: Some(MeshEndpointConfig {
        island_id: "lima-district-1".into(),
        transports: vec![MeshTransportType::WifiMesh, MeshTransportType::Lora],
    }),
}).await?;

// Service is reachable from:
// 1. Other nodes in the same mesh island (directly via mesh)
// 2. Clearnet BGP-X users (via domain bridge node)
println!("Service ID: {}", listener.service_id().to_hex());
```

### 8.9 Monitor Path Quality

```rust
let client = Client::connect_auto().await?;

let mut events = client.subscribe(&[
    EventType::PathDegraded,
    EventType::PathRebuilt,
    EventType::PoolMemberOffline,
    EventType::DomainBridgeOnline,
    EventType::DomainBridgeOffline,
    EventType::MeshIslandUnreachable,
    EventType::MeshIslandOnline,
    EventType::EchFailed,
]).await?;

while let Some(event) = events.next().await {
    match event {
        BgpxEvent::PathDegraded { quality_score, reason } => {
            eprintln!("Path degraded: {:.2} ({})", quality_score, reason);
        }
        BgpxEvent::MeshIslandUnreachable { island_id } => {
            eprintln!("Mesh island unreachable: {}", island_id);
        }
        BgpxEvent::DomainBridgeOffline { from_domain, to_domain } => {
            eprintln!("Bridge offline: {} → {}", from_domain, to_domain);
        }
        BgpxEvent::EchFailed { stream_id } => {
            eprintln!("ECH failed for stream {}", stream_id);
        }
        _ => {}
    }
}
```

### 8.10 Check Mesh Island Status

```rust
let client = Client::connect_auto().await?;

let status = client.get_island_status("lima-district-1").await?;

println!("Island online: {}", status.online);
println!("Active relays: {}", status.active_relay_count);
println!("Bridge nodes: {}", status.bridge_node_count);
println!("Services: {}", status.service_count);
```

### 8.11 Test Domain Connectivity

```rust
let client = Client::connect_auto().await?;

let result = client.test_domain_connectivity(
    RoutingDomain::Mesh { island_id: "lima-district-1".into() }
).await?;

println!("Reachable: {}", result.reachable);
println!("Latency: {} ms", result.latency_ms);
println!("Available bridges: {}", result.available_bridges);
```

---

## 9. Embedded Mode (Mobile/Constrained)

For mobile applications and environments where a separate daemon process is impractical, the SDK can embed the full BGP-X daemon as a library.

```rust
// Enable embedded feature in Cargo.toml:
// bgpx-sdk = { version = "0.1", features = ["embedded"] }

use bgpx_sdk::embedded::EmbeddedClient;

let client = EmbeddedClient::new(ClientConfig {
    data_dir: app_data_directory(),
    mesh_enabled: false,                       // Disable for battery savings
    cover_traffic: false,                      // Disable for battery savings
    geo_plausibility: false,                   // Skip geo checks
    ..Default::default()
}).await?;

// Same API as daemon-connected mode
let stream = client.connect_stream("example.com:443", Default::default()).await?;
```

### 9.1 Embedded Mode Limitations

- Higher battery drain (crypto in-process)
- Higher memory usage (DHT state in-process)
- Background operation limits on iOS
- Does NOT protect other apps (only the SDK-using app)
- Full bgpx-node daemon stack in-process

### 9.2 Android

Use `VpnService` API for system-level TUN interface. SDK embedded mode provides full stack.

```toml
bgpx-sdk = { version = "0.1", features = ["embedded", "android"] }
```

### 9.3 iOS

Use `Network Extension` framework. Same embedded SDK.

```toml
bgpx-sdk = { version = "0.1", features = ["embedded", "ios"] }
```

---

## 10. Platform-Specific Notes

### 10.1 All Platforms

All platforms support cross-domain routing via the daemon. No platform-specific changes needed for cross-domain support. ECH requirements unchanged (exit nodes only).

### 10.2 Android

- SDK uses daemon via TCP socket on LAN, or embedded mode for standalone deployment
- Use `VpnService` for TUN interface
- Background operation limitations

### 10.3 iOS

- SDK uses daemon via TCP socket on LAN, or embedded mode
- Background operation limitations
- `Network Extension` required for system-level VPN functionality

---

## 11. Geographic Plausibility (Optional Feature)

Geographic plausibility scoring is an **optional** reputation signal in BGP-X. It is NOT required for BGP-X operation.

### When Geo Plausibility Applies

- IF a node declares a jurisdiction in its advertisement: geo plausibility scoring applies
- IF a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

### Jurisdiction Declaration is Optional

Nodes are NOT required to declare a jurisdiction. Declaring jurisdiction is an opt-in privacy/convenience tradeoff:

- **Pro**: Users can select paths that avoid or prefer specific jurisdictions
- **Con**: Declaring jurisdiction reveals information about the node's location

### Exemptions

The following node types are EXEMPT from geo plausibility scoring:

- Satellite-class clearnet nodes (`latency_class = satellite-*`)
- Mesh nodes without declared jurisdiction
- Nodes in domains without internet RTT calibration

### Behavior in Path Construction

A node with poor geo plausibility score:

- Receives lower selection probability (scoring penalty)
- Can still be selected in a path (not hard excluded)
- Persistent implausibility may lead to reputation penalties

Geo plausibility is a reputation signal, not a mandatory filtering criterion.

---

## 12. Satellite Internet

Satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) are **clearnet domain** (0x00000001).

From BGP-X's perspective, a satellite-connected node is a clearnet node with high latency class annotation. The physical medium (fiber vs. satellite radio) is invisible to the BGP-X protocol layer.

When connecting through a path that includes a satellite-connected relay:

```rust
// The SDK treats satellite links as clearnet with high latency
let stream = client.connect_stream(
    "example.com:443",
    StreamConfig {
        prefer_ech: true,                        // ECH works over satellite
        idle_timeout_seconds: 600,               // Higher timeout for GEO satellite
        ..Default::default()
    },
).await?;
```

---

## 13. SDK Versioning

SDK version tracks daemon version. SDK 0.1 is compatible with daemon 0.1.

Breaking SDK changes = new major version.

Connection resilience: SDK reconnects to daemon automatically after disconnect with exponential backoff (100ms base, 30s maximum, ±10% jitter).

---

## 14. HTTP/2 over BGP-X Streams

BGP-X native services (.bgpx addresses) use **HTTP/2 over BGP-X streams**.

HTTP/2 is selected over HTTP/3 because:

- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers. This is standard HTTP/3 over TLS over the exit's clearnet connection — not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.
