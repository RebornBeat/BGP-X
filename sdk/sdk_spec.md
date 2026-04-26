# BGP-X SDK Specification

**Version**: 0.1.0-draft

---

## 1. SDK Architecture — Router-Centric Model

The BGP-X SDK is a client library that connects to the BGP-X daemon. The SDK does NOT implement routing, handshakes, DHT, or crypto. All of these are done by the daemon.

Connection model, auto-discovery, LAN device access — all unchanged from single-domain specification.

---

## 2. Language Targets

Rust (primary reference), Python (PyO3 bindings), JavaScript/TypeScript (napi-rs bindings), Go (CGo), C (FFI layer) — all planned.

---

## 3. New Types for Cross-Domain

### RoutingDomain

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum RoutingDomain {
    Clearnet,
    BgpxOverlay,
    Mesh { island_id: String },
    LoraRegional { region_id: String },
    Satellite { orbit_id: String },
    Auto,  // Path construction chooses optimal domain
}

impl RoutingDomain {
    pub fn to_domain_id(&self) -> DomainId;
    pub fn from_domain_id(id: DomainId) -> Self;
    pub fn is_mesh(&self) -> bool { matches!(self, Self::Mesh { .. }) }
}
```

### DomainTransition

```rust
pub enum DomainTransition {
    Auto,
    Via { bridge_pool: String },
    ViaNode { node_id: String },
    DirectOrViaInternet,
}
```

### DomainSegmentConfig

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
    Relay,
    Bridge,
    BridgeTo { target_domain: RoutingDomain },
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

### PathConfig (Updated)

```rust
pub struct PathConfig {
    // Cross-domain path — takes priority over segments if specified
    pub domain_segments: Vec<DomainSegmentConfig>,

    // Single-domain pool path — backward compatible
    pub segments: Vec<SegmentConfig>,

    pub fallback_to_default: bool,
    pub enforce_pool_isolation: bool,
    pub max_total_hops: Option<u16>,   // Optional cap; no protocol default maximum
    pub preferred_domains: Vec<RoutingDomain>,
    pub avoid_domains: Vec<RoutingDomain>,
    pub cover_traffic: bool,

    // Single-domain shortcuts (unchanged)
    pub hops: Option<u8>,
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
                    exit_logging_policy: Some(LoggingPolicy::None),
                    ..Default::default()
                },
            }],
            fallback_to_default: true,
            enforce_pool_isolation: true,
            max_total_hops: None,  // No protocol maximum
            preferred_domains: vec![],
            avoid_domains: vec![],
            cover_traffic: false,
            hops: None,
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

### PathInfo (Updated)

```rust
pub struct PathInfo {
    pub hops: u8,
    pub age_seconds: u64,
    pub estimated_rtt_ms: u32,
    pub quality_score: f32,
    pub exit_jurisdiction: String,
    pub exit_logging_policy: LoggingPolicy,
    pub pool_segments: Vec<PoolSegmentInfo>,
    pub domain_segments: Vec<DomainSegmentInfo>,
    pub total_domains: u8,
    pub is_cross_domain: bool,
    pub domain_bridge_count: u8,
}

pub struct DomainSegmentInfo {
    pub domain: RoutingDomain,
    pub hops: u8,
    pub estimated_latency_ms: u32,
    pub is_bridge: bool,
    pub segment_quality: f32,
}
```

### StreamInfo (Updated)

```rust
pub struct StreamInfo {
    pub stream_id: StreamId,
    pub ech_negotiated: bool,
    pub ech_available: bool,
    pub path_hops: u8,
    pub is_cross_domain: bool,
    pub domain_count: u8,
}
```

---

## 4. Core API (Updated)

### Client

```rust
pub struct Client { /* opaque */ }

impl Client {
    pub async fn connect_auto() -> Result<Self, SdkError>;
    pub async fn connect(socket_path: &Path) -> Result<Self, SdkError>;
    pub async fn connect_tcp(addr: &str) -> Result<Self, SdkError>;

    pub async fn status(&self) -> Result<DaemonStatus, SdkError>;

    // Single-domain (unchanged)
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

    // Cross-domain (new)
    pub async fn connect_stream_cross_domain(
        &self,
        destination: &str,
        domain_segments: Vec<DomainSegmentConfig>,
        stream_config: StreamConfig,
    ) -> Result<BgpxStream, SdkError>;

    // Native services (unchanged)
    pub async fn register_service(
        &self,
        config: ServiceConfig,
    ) -> Result<BgpxListener, SdkError>;

    pub async fn path_quality(&self) -> Result<PathQuality, SdkError>;

    pub async fn subscribe(
        &self,
        events: &[EventType],
    ) -> Result<EventStream, SdkError>;

    // Pool management (unchanged)
    pub async fn add_pool(&self, pool: Pool) -> Result<(), SdkError>;
    pub async fn remove_pool(&self, pool_id: &str) -> Result<(), SdkError>;
    pub async fn list_pools(&self) -> Result<Vec<PoolSummary>, SdkError>;

    // Domain management (new)
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
}
```

---

## 5. Usage Examples

### Simple Clearnet Connection (Unchanged)

```rust
let client = Client::connect_auto().await?;
let mut stream = client.connect_stream("example.com:443", Default::default()).await?;
stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await?;
```

### Cross-Domain: Clearnet Client → Mesh Service

```rust
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

### Clearnet → Mesh → Clearnet (Three Domains)

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

### N-Hop Unlimited (20 Hops, 3 Domains)

```rust
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
        max_total_hops: None,  // No protocol limit
        ..Default::default()
    },
    Default::default(),
).await?;
```

### Register BGP-X Native Service (Unchanged)

```rust
let listener = client.register_service(ServiceConfig {
    name: "my-private-api".to_string(),
    advertise: true,
    auth_required: false,
    max_connections: 100,
}).await?;

println!("Service ID: {}", listener.service_id().to_hex());

loop {
    let (mut stream, info) = listener.accept().await?;
    tokio::spawn(async move { handle_connection(stream, info).await; });
}
```

### Monitor Path Quality (Updated)

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
        _ => {}
    }
}
```

### Check Mesh Island Status

```rust
let status = client.get_island_status("lima-district-1").await?;
println!("Island online: {}", status.online);
println!("Active relays: {}", status.active_relay_count);
println!("Bridge nodes: {}", status.bridge_node_count);
```

---

## 6. Error Handling

```rust
pub enum SdkError {
    DaemonNotFound,
    ConnectionFailed(IoError),
    InvalidDaemonState(DaemonState),
    PathConstructionFailed(String),
    StreamReset,
    StreamClosed,
    Timeout,
    ExitPolicyDenied,
    DestinationUnreachable(String),
    EchRequired,
    PoolNotFound(String),
    PoolInsufficientNodes(String),
    MeshTransportUnavailable,
    InvalidConfig(String),
    Internal(String),

    // Cross-domain errors
    DomainNotFound(String),
    DomainBridgeUnavailable { from: String, to: String },
    MeshIslandUnreachable(String),
    NoCrossdomainPath,
    DomainScopeMismatch { required: String, found: String },
    StreamPaused { reason: String, estimated_resume_secs: Option<u64> },
}
```

---

## 7. Events

```rust
pub enum BgpxEvent {
    // All existing events retained
    PathBuilt { hops: u8, exit_jurisdiction: String, is_cross_domain: bool },
    PathDegraded { quality_score: f32, reason: String },
    PathRebuilt { hops: u8, reason: String, is_cross_domain: bool },
    PoolMemberOffline { pool_name: String },
    ServiceConnected,
    ServiceDisconnected,

    // New cross-domain events
    DomainBridgeOnline { from_domain: String, to_domain: String },
    DomainBridgeOffline { from_domain: String, to_domain: String },
    MeshIslandDiscovered { island_id: String },
    MeshIslandUnreachable { island_id: String },
    MeshIslandOnline { island_id: String },
    CrossDomainPathBuilt { domains: Vec<String>, bridge_count: u8 },
    StreamPaused { stream_id: StreamId, reason: String, domain: String },
    StreamResumed { stream_id: StreamId },
}
```

---

## 8. SDK Versioning

SDK version tracks daemon version. SDK 0.1 compatible with daemon 0.1. Reconnection with exponential backoff (100ms base, 30s maximum, ±10% jitter).

---

## 9. Platform-Specific Notes

All platforms support cross-domain routing via the daemon. No platform-specific changes needed for cross-domain. ECH requirements unchanged (exit nodes only). Android and iOS SDK use the daemon via TCP socket on LAN, or embedded mode for standalone deployment.

---

## 10. Embedded Mode (Unchanged)

SDK can embed the full BGP-X daemon within the application process. Same API surface. `Client::embedded(config)` instead of `Client::connect_auto()`. All cross-domain features available in embedded mode.
