# BGP-X Gateway and Domain Bridge Node Specification

**Version**: 0.1.0-draft

---

## 1. Generalized Role: Domain Bridge Node

The "gateway" concept is generalized to the **Domain Bridge Node** — any node with endpoints in two or more routing domains.

The previous notion of "gateway" (mesh ↔ internet bridge with clearnet exit) is a specific type of domain bridge node. All gateway properties are retained as instances of the general domain bridge model.

**Domain bridge node types**:

| Type | From Domain | To Domain | Exit Policy Required |
|---|---|---|---|
| Clearnet exit gateway | clearnet | clearnet (different network) | Yes |
| Mesh gateway (with exit) | clearnet | mesh:\<island_id\> | Yes |
| Mesh relay bridge (no exit) | clearnet | mesh:\<island_id\> | No |
| Mesh-to-mesh bridge | mesh:\<A\> | mesh:\<B\> | No |
| Satellite gateway | clearnet | satellite:\<orbit_id\> | Yes |
| Satellite-mesh bridge | satellite:\<X\> | mesh:\<Y\> | Yes |

All types use the same daemon code and DOMAIN_BRIDGE hop type. The distinction is configuration.

---

## 2. Architecture

```
Any domain (source)
         │
         ▼
Onion decryption (this node's layer — hop_type = 0x06 DOMAIN_BRIDGE)
         │
         ├── Exit policy check (ONLY when target destination = clearnet internet)
         │   → DENY → STREAM_CLOSE (ERR_EXIT_POLICY_DENIED)
         │   → ALLOW ↓
         │
         ▼
Cross-domain path_id registration
  path_id → source_domain, source_addr
  path_id → target_domain, target_addr (from subsequent hops)
         │
         ▼
Target domain transport selection
         │
         ▼
Target domain address resolution
         │
         ▼
Forward remaining onion payload via target transport (NO re-encryption)

Return traffic (from target domain):
  path_id cross-domain lookup → source_domain, source_addr
  Forward opaque blob via source domain transport (NO decryption)
```

---

## 3. Exit Policy Enforcement

Exit policy applies ONLY when:
- The bridge connects to clearnet AND
- The traffic is destined for the public internet (not for another BGP-X node or mesh service)

For mesh-to-mesh bridges: no exit policy needed.
For clearnet-to-mesh bridges (relay only): no exit policy needed.
For clearnet exit gateways: exit policy required as previously specified.

---

## 4. DoH DNS Resolver (Clearnet Exit Only)

Required only for nodes with clearnet exit capability. Unchanged specification.

```toml
[exit_policy]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

ECS stripping (MANDATORY for exit nodes): strip EDNS Client Subnet from DNS queries to prevent client IP subnet leakage to DNS resolvers and destination servers.

---

## 5. ECH Support (Clearnet Exit Only)

ECH applies only when the bridge includes a clearnet exit. Unchanged specification. Mesh relay bridge nodes do not perform TLS ClientHello and do not need ECH.

---

## 6. Clearnet Client → Mesh Island Service

The most important capability enabled by the domain bridge model. A clearnet client with no mesh hardware can reach a mesh island service:

```
Clearnet client (no mesh radio, no BGP-X router)
         │ UDP/IP
         ▼
[Clearnet relay nodes — standard onion encryption]
         │ UDP/IP
         ▼
[Domain Bridge Node — clearnet UDP endpoint + mesh radio]
  1. Receives onion packet via clearnet UDP
  2. Decrypts its DOMAIN_BRIDGE layer
  3. Stores path_id → clearnet predecessor (for return routing)
  4. Forwards remaining payload via mesh radio
         │ WiFi mesh or LoRa
         ▼
[Mesh relay nodes — onion encryption in mesh domain]
         │ mesh transport
         ▼
[Mesh service]
```

Return path:
```
Mesh service → mesh relays → bridge node (via mesh radio, opaque)
Bridge node: cross-domain path_id lookup → clearnet predecessor
Bridge node → clearnet relays → client (via UDP, opaque)
```

The clearnet client needs no mesh hardware, no BGP-X router, no special configuration. The bridge node handles all domain translation. The client's clearnet IP is hidden from the mesh service.

---

## 7. Mesh-to-Mesh Bridge

A node with two mesh transport interfaces bridges two mesh islands directly without clearnet:

```
Mesh Island A (WiFi mesh)
         │
[Multi-transport Bridge Node]
  Receives from Island A via WiFi mesh
  Decrypts DOMAIN_BRIDGE layer
  Forwards to Island B via LoRa
         │
Mesh Island B (LoRa)
```

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "island-a"
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"

[[routing_domains]]
domain_type = "mesh"
island_id = "island-b"
transports = ["lora"]
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true
```

No clearnet exit policy needed. No DoH DNS needed. Pure radio-to-radio domain bridge.

---

## 8. Unified DHT (Replaces Two-DHT Model)

Gateway nodes NO LONGER synchronize between two separate DHTs. Instead:

- When online: gateway publishes mesh island records directly to the unified internet DHT
- Mesh-only nodes' advertisements are stored in unified DHT by the gateway on their behalf
- When gateway goes offline: records remain valid until TTL; new records cannot be published
- When gateway reconnects: any pending record updates published automatically

No synchronization algorithm needed — there is only one DHT.

---

## 9. Response Handling and Return Path

Return traffic from the clearnet destination travels back through the SAME path in reverse using path_id routing:

1. Response arrives at gateway from clearnet destination
2. Gateway encrypts response using K_exit (session key shared with client)
3. Gateway looks up path_id in cross-domain path table → last clearnet relay
4. Sends encrypted blob via clearnet UDP to last clearnet relay (opaque)
5. Each clearnet relay forwards opaquely via path_id
6. Client decrypts with K_exit

For mesh service responses, the mesh service encrypts with K_service and return path crosses domain boundary at bridge node as described in ARCHITECTURE.md Section 4.2.

---

## 10. Exit Policy Versioning (Unchanged)

When exit policy changes: increment `exit_policy_version`, set `exit_policy_previous_version`, re-publish advertisement. Old policy honored for existing streams; new policy for new streams. Domain bridge nodes without clearnet exit functionality do not use exit policy versioning.

---

## 11. Logging Policy (Unchanged)

`logging_policy = "none"` (technical enforcement): no file handles for access logging; aggregate stats in memory only; only daemon error events logged.

---

## 12. Monitoring

All previous exit node metrics retained. New:

```
bgpx_domain_bridge_transitions_total    # Packets forwarded across domain boundary
bgpx_domain_bridge_latency_ms          # Average bridge transition latency
bgpx_cross_domain_return_forwarded     # Return packets forwarded cross-domain
bgpx_mesh_radio_send_total             # Packets sent via mesh transport
bgpx_mesh_radio_recv_total             # Packets received via mesh transport
bgpx_cross_domain_path_table_size      # Active entries in cross-domain path table
```
```

---

# `sdk/sdk_spec.md`

```markdown
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
