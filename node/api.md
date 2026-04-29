# BGP-X Node Internal API Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X node daemon is composed of loosely coupled subsystems communicating through well-defined internal interfaces. This document specifies those interfaces for implementors building the reference implementation or alternative implementations.

All interfaces are internal — not exposed over the network or to external processes. The Control API (`/control-plane/control_api.md`) and SDK API (`/sdk/sdk_spec.md`) are the external-facing interfaces.

---

## 2. Core Data Types

```rust
/// 32-byte BLAKE3 hash of Ed25519 public key
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct NodeId([u8; 32]);
impl NodeId {
    pub fn from_public_key(key: &Ed25519PublicKey) -> Self;
    pub fn to_hex(&self) -> String;
    pub fn from_hex(s: &str) -> Result<Self, ParseError>;
    pub fn xor_distance(&self, other: &NodeId) -> NodeId;
}

/// 16-byte random session identifier
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct SessionId([u8; 16]);
impl SessionId {
    pub fn generate() -> Self;  // CSPRNG
}

/// 8-byte random path identifier — return traffic routing, never all-zeros
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct PathId([u8; 8]);
impl PathId {
    pub fn generate() -> Self;  // CSPRNG, retries until not all-zeros
    pub fn is_valid(&self) -> bool { self.0 != [0u8; 8] }
}

/// 32-bit stream identifier with parity enforcement
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct StreamId(u32);
impl StreamId {
    pub fn is_client_initiated(&self) -> bool { self.0 % 2 == 1 }
    pub fn is_service_initiated(&self) -> bool { self.0 % 2 == 0 }
    pub fn next_client_id(current: StreamId) -> StreamId { StreamId(current.0 + 2) }
}

/// 8-byte routing domain identifier: type (4B BE) + instance hash (4B BE)
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct DomainId([u8; 8]);
impl DomainId {
    pub fn clearnet() -> Self { DomainId([0,0,0,1, 0,0,0,0]) }
    pub fn mesh(island_id: &str) -> Self {
        let hash = blake3::hash(island_id.as_bytes());
        let mut id = [0u8; 8];
        id[0..4].copy_from_slice(&[0,0,0,3]);
        id[4..8].copy_from_slice(&hash.as_bytes()[0..4]);
        DomainId(id)
    }
    pub fn domain_type(&self) -> u32 { u32::from_be_bytes(self.0[0..4].try_into().unwrap()) }
    pub fn is_clearnet(&self) -> bool { self.domain_type() == 1 }
    pub fn is_mesh(&self) -> bool { self.domain_type() == 3 }
    pub fn is_satellite(&self) -> bool { self.domain_type() == 5 }
}

/// Session key (zeroizable) — domain-agnostic, same derivation in all domains
pub struct SessionKey(Zeroizing<[u8; 32]>);
impl SessionKey {
    pub fn from_handshake_material(
        dh1: &[u8; 32],
        dh2: &[u8; 32],
        session_id: &SessionId,
        node_id: &NodeId,
    ) -> Self;
    pub fn encrypt(&self, sequence: u64, aad: &[u8], plaintext: &[u8]) -> Vec<u8>;
    pub fn decrypt(&self, sequence: u64, aad: &[u8], ciphertext: &[u8])
        -> Result<Vec<u8>, DecryptionError>;
}
impl Drop for SessionKey {
    fn drop(&mut self) { /* explicit zeroize using zeroize crate */ }
}

/// Cross-domain path table entry for bridge nodes
pub struct CrossDomainPathEntry {
    pub source_domain: DomainId,
    pub source_addr:   SocketAddr,
    pub target_domain: DomainId,
    pub created:       Instant,
    pub ttl:           Duration,
}
```

---

## 3. Subsystem Interfaces

### 3.1 ForwardingPipeline

```rust
pub trait ForwardingPipeline: Send + Sync {
    fn process_packet(&self, source_addr: SocketAddr, domain: DomainId, data: &[u8]) -> ProcessResult;
    fn enqueue_packet(&self, dest_addr: SocketAddr, domain: DomainId, data: Bytes) -> Result<(), QueueFullError>;
    fn register_session(&self, session_id: SessionId, session_key: SessionKey, remote_addr: SocketAddr) -> Result<(), SessionError>;
    fn remove_session(&self, session_id: &SessionId);

    // Single-domain path table
    fn record_path_id(&self, path_id: PathId, source_addr: SocketAddr, session_id: &SessionId);
    fn lookup_path_predecessor(&self, path_id: &PathId) -> Option<SocketAddr>;

    // Cross-domain path table (for bridge nodes)
    fn record_cross_domain_path_id(
        &self,
        path_id: PathId,
        source_domain: DomainId,
        source_addr: SocketAddr,
        target_domain: DomainId,
    );
    fn resolve_cross_domain_return(
        &self,
        path_id: &PathId,
        from_domain: &DomainId,
    ) -> Option<(DomainId, SocketAddr)>;

    fn forward_return_traffic(&self, path_id: &PathId, packet: Bytes) -> Result<(), ForwardError>;
    fn forward_cross_domain_return(
        &self,
        path_id: &PathId,
        from_domain: &DomainId,
        packet: Bytes,
    ) -> Result<(), ForwardError>;

    fn stats(&self) -> ForwardingStats;
}

pub enum ProcessResult {
    Forwarded,
    CrossDomainForwarded,   // DOMAIN_BRIDGE hop dispatched to target transport
    ReturnForwarded,        // Return traffic forwarded via path_id
    CrossDomainReturnForwarded,  // Return traffic forwarded across domain boundary
    Delivered,              // BGP-X native service delivery
    DroppedDecrypt,
    DroppedReplay,
    DroppedUnknown,
    DroppedInvalid,
    HandshakeMessage,
    DhtMessage,
    KeepaliveMessage,
    MeshBeacon,
    MeshFragment,
    NodeWithdrawal,
    PathQualityReport,
    PoolKeyRotation,
    DomainAdvertise,
    MeshIslandAdvertise,
    CoverTraffic,           // COVER processed identically to RELAY (silent drop after failed parse)
}
```

### 3.2 SessionManager

```rust
pub trait SessionManager: Send + Sync {
    fn get_session(&self, id: &SessionId) -> Option<Arc<Session>>;
    fn create_session(
        &self,
        id: SessionId,
        key: SessionKey,
        keepalive_key: SessionKey,
        remote_node_id: Option<NodeId>,
        remote_addr: SocketAddr,
        domain: DomainId,    // Which routing domain this session is in
    ) -> Result<Arc<Session>, SessionError>;
    fn close_session(&self, id: &SessionId, reason: CloseReason);
    fn schedule_rehandshake(&self, id: &SessionId, at: Duration);
    fn list_sessions(&self) -> Vec<SessionSummary>;
    fn session_count(&self) -> usize;
}

pub struct Session {
    pub id:                    SessionId,
    pub key:                   SessionKey,
    pub keepalive_key:         SessionKey,
    pub domain:                DomainId,         // Routing domain for this session
    pub state:                 AtomicSessionState,
    pub inbound_sequence_window: Mutex<SequenceWindow>,
    pub outbound_sequence:     AtomicU64,
    pub last_seen:             AtomicU64,
    pub bytes_relayed:         AtomicU64,
    pub packets_relayed:       AtomicU64,
    pub decryption_failures:   AtomicU32,
    pub created_at:            Instant,
    pub streams:               DashMap<StreamId, Arc<Stream>>,
    pub path_ids:              DashSet<PathId>,   // path_ids associated with this session
}
```

### 3.3 HandshakeHandler

```rust
pub trait HandshakeHandler: Send + Sync {
    /// Handle HANDSHAKE_INIT (0x02)
    /// cleartext_ephemeral_pub is the 32-byte client ephemeral pubkey from bytes 32-63
    fn handle_init(
        &self,
        source_addr: SocketAddr,
        source_domain: DomainId,
        cleartext_ephemeral_pub: &[u8; 32],
        encrypted_payload: &[u8],
    ) -> HandshakeResult;

    /// Handle HANDSHAKE_DONE (0x04)
    fn handle_done(
        &self,
        session_id: &SessionId,
        payload: &[u8],
    ) -> HandshakeResult;

    /// Handle re-handshake on existing session (24hr key rotation)
    fn handle_rehandshake(
        &self,
        session_id: &SessionId,
        cleartext_ephemeral_pub: &[u8; 32],
        encrypted_payload: &[u8],
    ) -> HandshakeResult;
}

pub enum HandshakeResult {
    InProgress,
    Complete(SessionId),
    RehandshakeComplete(SessionId),
    Failed,
}
```

### 3.4 PathTable

```rust
/// Single-domain path table
pub trait PathTable: Send + Sync {
    fn insert(&self, path_id: PathId, source_addr: SocketAddr, session_id: SessionId);
    fn lookup(&self, path_id: &PathId) -> Option<PathTableEntry>;
    fn remove(&self, path_id: &PathId);
    fn cleanup_expired(&self);
    fn size(&self) -> usize;
}

pub struct PathTableEntry {
    pub source_addr: SocketAddr,
    pub session_id:  SessionId,
    pub created:     Instant,
    pub ttl:         Duration,
}
```

### 3.5 CrossDomainPathTable

```rust
/// Cross-domain path table for bridge nodes
pub trait CrossDomainPathTable: Send + Sync {
    fn insert(
        &self,
        path_id: PathId,
        source_domain: DomainId,
        source_addr: SocketAddr,
        target_domain: DomainId,
    );
    fn lookup(&self, path_id: &PathId) -> Option<CrossDomainPathEntry>;
    fn lookup_return(
        &self,
        path_id: &PathId,
        arriving_from_domain: &DomainId,
    ) -> Option<(DomainId, SocketAddr)>;  // Returns (forward_to_domain, forward_to_addr)
    fn remove(&self, path_id: &PathId);
    fn cleanup_expired(&self);
    fn size(&self) -> usize;
}
```

### 3.6 DhtSubsystem

```rust
pub trait DhtSubsystem: Send + Sync {
    // Standard DHT operations
    async fn find_nodes(&self, target: &NodeId, max_results: usize) -> Vec<NodeContact>;
    async fn find_nodes_in_domain(
        &self,
        target: &NodeId,
        domain: &DomainId,
        max_results: usize,
    ) -> Vec<NodeContact>;

    async fn put(&self, key: &[u8; 32], value: Vec<u8>) -> Result<usize, DhtError>;
    async fn get(&self, key: &[u8; 32]) -> Result<Option<Vec<u8>>, DhtError>;

    // Advertisement publication
    async fn publish_advertisement(&self, advertisement: &NodeAdvertisement) -> Result<usize, DhtError>;
    async fn publish_withdrawal(&self, node_id: &NodeId, private_key: &Ed25519PrivateKey) -> Result<(), DhtError>;

    // Domain bridge records
    async fn publish_domain_bridge(&self, record: &DomainBridgeRecord) -> Result<usize, DhtError>;
    async fn get_domain_bridge(
        &self,
        from_domain: &DomainId,
        to_domain: &DomainId,
    ) -> Result<Option<DomainBridgeRecord>, DhtError>;

    // Mesh island records
    async fn publish_mesh_island(&self, record: &MeshIslandRecord) -> Result<usize, DhtError>;
    async fn get_mesh_island(&self, island_id: &str) -> Result<Option<MeshIslandRecord>, DhtError>;

    // Mesh transport
    async fn broadcast_beacon(&self, beacon: &MeshBeacon) -> Result<usize, DhtError>;

    fn stats(&self) -> DhtStats;
    fn estimated_network_size(&self) -> u64;
}
```

### 3.7 PoolManager

```rust
pub trait PoolManager: Send + Sync {
    async fn get_pool(&self, pool_id: &str) -> Option<Arc<Pool>>;
    async fn add_pool(&self, pool: Pool) -> Result<(), PoolError>;
    async fn remove_pool(&self, pool_id: &str) -> Result<(), PoolError>;
    async fn refresh_pool(&self, pool_id: &str) -> Result<usize, PoolError>;
    async fn verify_pool_membership(&self, pool: &Pool, node: &NodeAdvertisement) -> bool;
    async fn apply_key_rotation(&self, rotation: &KeyRotationRecord) -> Result<(), PoolError>;
    // Domain-aware pool operations
    async fn get_pools_for_domain(&self, domain: &DomainId) -> Vec<Arc<Pool>>;
    async fn get_bridge_pools(
        &self,
        from_domain: &DomainId,
        to_domain: &DomainId,
    ) -> Vec<Arc<Pool>>;
    fn list_pools(&self) -> Vec<PoolSummary>;
}
```

### 3.8 DomainBridgeManager

```rust
/// Manages cross-domain routing for nodes that serve multiple routing domains
pub trait DomainBridgeManager: Send + Sync {
    /// Returns all routing domains this node has active endpoints in
    fn get_served_domains(&self) -> Vec<DomainId>;

    /// Returns all active domain bridge pairs this node supports
    fn get_active_bridges(&self) -> Vec<(DomainId, DomainId)>;

    /// Returns true if this node can bridge the given domain pair
    fn can_bridge(&self, from_domain: &DomainId, to_domain: &DomainId) -> bool;

    /// Execute a domain bridge transition (forward packet to target domain)
    async fn transition(
        &self,
        packet: Bytes,
        from_domain: DomainId,
        to_domain: DomainId,
        path_id: PathId,
        source_addr: SocketAddr,
    ) -> Result<(), BridgeError>;

    /// Record a cross-domain path_id mapping (called from forwarding pipeline)
    fn record_cross_domain_path_id(
        &self,
        path_id: PathId,
        source_domain: DomainId,
        source_addr: SocketAddr,
        target_domain: DomainId,
    );

    /// Resolve return routing for a given path_id arriving from a specific domain
    fn resolve_cross_domain_return(
        &self,
        path_id: &PathId,
        arriving_from_domain: &DomainId,
    ) -> Option<(DomainId, SocketAddr)>;

    /// Mark a bridge pair as temporarily unavailable (e.g., mesh radio offline)
    async fn set_bridge_availability(
        &self,
        from_domain: &DomainId,
        to_domain: &DomainId,
        available: bool,
    ) -> Result<(), BridgeError>;

    /// Statistics for this bridge node
    fn bridge_stats(&self) -> BridgeStats;
}

pub struct BridgeStats {
    pub transitions_total:         u64,
    pub transitions_by_pair:       HashMap<(DomainId, DomainId), u64>,
    pub cross_domain_path_entries: usize,
    pub bridge_latency_ms:         HashMap<(DomainId, DomainId), f64>,
}

pub enum BridgeError {
    DomainNotServed(DomainId),
    NoBridgeForPair(DomainId, DomainId),
    BridgeUnavailable,
    TransportError(TransportError),
    NodeNotResolvable,
}
```

### 3.9 MeshIslandManager

```rust
/// Manages mesh island registration and DHT publishing
pub trait MeshIslandManager: Send + Sync {
    /// Register this node as participating in a mesh island
    async fn register_island(&self, island: MeshIslandConfig) -> Result<(), IslandError>;

    /// Get the current status of a known mesh island
    async fn get_island_status(&self, island_id: &str) -> Option<MeshIslandStatus>;

    /// List all known mesh islands (local cache)
    fn list_known_islands(&self) -> Vec<IslandSummary>;

    /// Force publish mesh island advertisement to unified DHT
    /// (called by bridge node when internet connectivity is available)
    async fn publish_island_advertisement(&self, island_id: &str) -> Result<usize, IslandError>;

    /// Fetch and cache mesh island advertisement from unified DHT
    async fn fetch_island_advertisement(&self, island_id: &str) -> Result<MeshIslandRecord, IslandError>;

    /// Record that a bridge node for an island has come online or gone offline
    fn record_bridge_availability(
        &self,
        island_id: &str,
        bridge_node_id: &NodeId,
        available: bool,
    );
}

pub struct MeshIslandConfig {
    pub island_id:             String,
    pub domain_id:             DomainId,
    pub transports:            Vec<TransportType>,
    pub auto_publish:          bool,
    pub advertisement_interval: Duration,
}

pub struct MeshIslandStatus {
    pub island_id:          String,
    pub domain_id:          DomainId,
    pub online:             bool,
    pub active_relay_count: u32,
    pub bridge_node_count:  u32,
    pub last_advertisement: Option<Instant>,
    pub dht_coverage:       DhtCoverage,
}
```

### 3.10 ReputationSystem

```rust
pub trait ReputationSystem: Send + Sync {
    fn record_event(&self, node_id: &NodeId, event: ReputationEvent);
    fn record_domain_event(&self, node_id: &NodeId, domain: &DomainId, event: ReputationEvent);
    fn get_score(&self, node_id: &NodeId) -> f32;
    fn get_domain_score(&self, node_id: &NodeId, domain: &DomainId) -> f32;
    fn is_blacklisted(&self, node_id: &NodeId) -> bool;
    fn blacklist(&self, node_id: &NodeId, reason: BlacklistReason, duration: Option<Duration>);
    fn unblacklist(&self, node_id: &NodeId) -> Result<(), ReputationError>;
    fn record_rtt(&self, node_id: &NodeId, rtt_ms: f64, domain: &DomainId);
    fn get_geo_plausibility_score(&self, node_id: &NodeId, domain: &DomainId) -> f32;
    async fn flush(&self) -> Result<(), IoError>;
}

pub enum ReputationEvent {
    // Unchanged events
    RelaySuccess, RelayFailure,
    KeepaliveSuccess, KeepaliveTimeout,
    HandshakeSuccess, HandshakeFailure,
    LatencyWithinAdvertised, LatencyExceeded,
    AdvertisementInconsistency,
    ProtocolViolation,
    SuspiciousBehavior,
    GeoPlausible, GeoSuspicious, GeoImplausible, ProtocolViolationGeo,
    EchCapableButFailing,
    CleanWithdrawal, WithdrawalWithoutNotice,

    // New cross-domain events
    DomainBridgeSuccess,           // +0.5: successfully bridged traffic between domains
    DomainBridgeFailure,           // -3.0: domain bridge failed (timeout/no connectivity)
    MeshIslandStable,              // +0.3: mesh island consistently reachable via this bridge
    MeshIslandUnreachable,         // -2.0: mesh island became unreachable via this bridge
    BridgeLatencyWithinAdvertised, // +0.3: bridge transition latency within advertised range
    BridgeLatencyExceeded,         // -1.0: bridge transition latency > 200% of advertised
    BridgeClaimedAvailableOffline, // -4.0: advertised bridge as available but couldn't deliver
    FakeDomainAdvertisement,       // -15.0: claimed domain endpoint it cannot reach
}
```

### 3.11 ExitPolicyEngine

```rust
pub trait ExitPolicyEngine: Send + Sync {
    fn allows(&self, protocol: Protocol, destination: &Destination, port: u16) -> PolicyDecision;
    async fn connect(
        &self,
        destination: &Destination,
        port: u16,
        protocol: Protocol,
    ) -> Result<OutboundConnection, ExitError>;
    async fn resolve_ech_config(&self, domain: &str) -> Option<EchConfig>;
    fn signed_policy(&self) -> &SignedExitPolicy;
    fn current_policy_version(&self) -> u16;
}

pub enum ExitError {
    PolicyDenied(PolicyDecision),
    ConnectionRefused,
    DnsResolutionFailed,
    Timeout,
    MaxConnectionsReached,
    EchRequiredNotAvailable,
    EchNegotiationFailed,
    PrivateIPDenied,
}
```

### 3.12 PluggableTransport

```rust
pub trait PluggableTransport: Send + Sync {
    async fn start(&self) -> Result<(), PtError>;
    async fn stop(&self) -> Result<(), PtError>;
    fn obfuscate(&self, packet: Bytes) -> Bytes;
    fn deobfuscate(&self, packet: Bytes) -> Result<Bytes, PtError>;
    fn is_running(&self) -> bool;
    fn overhead_estimate(&self) -> PtOverhead;
}

pub struct PtOverhead {
    pub cpu_percent_extra:       f32,
    pub bandwidth_percent_extra: f32,
    pub latency_ms_extra:        f32,
}
```

### 3.13 GeoPlausibilityScorer

```rust
pub trait GeoPlausibilityScorer: Send + Sync {
    fn record_rtt(&self, node_id: &NodeId, rtt_ms: f64, domain: &DomainId);
    fn get_score(&self, node_id: &NodeId, domain: &DomainId) -> f32;
    fn check_plausibility(
        &self,
        node_id: &NodeId,
        claimed_region: &Region,
        domain: &DomainId,
    ) -> PlausibilityResult;
    fn is_satellite_exempt(&self, node: &NodeDatabaseEntry) -> bool;
    fn is_lora_mesh_node(&self, node: &NodeDatabaseEntry, domain: &DomainId) -> bool;
}

pub struct PlausibilityResult {
    pub score:             f32,    // 0.0-1.0
    pub ratio:             f64,    // measured_rtt / expected_max_rtt
    pub measurement_count: u32,
    pub recommendation:    PlausibilityRecommendation,
    pub domain:            DomainId,
}
```

### 3.14 RoutingPolicyEngine

```rust
pub trait RoutingPolicyEngine: Send + Sync {
    fn evaluate_packet(&self, packet: &IncomingPacket) -> RoutingDecision;
    fn add_rule(&self, rule: RoutingRule, position: usize) -> Result<RuleId, PolicyError>;
    fn remove_rule(&self, rule_id: &RuleId) -> Result<(), PolicyError>;
    fn reorder_rule(&self, rule_id: &RuleId, new_position: usize) -> Result<(), PolicyError>;
    fn reload(&self) -> Result<(), PolicyError>;
    fn test_destination(
        &self,
        destination: &str,
        device_ip: IpAddr,
    ) -> TestResult;
    fn stats(&self) -> PolicyStats;
}

pub enum RoutingDecision {
    RouteBgpx {
        path_constraints: Option<PathConstraints>,
        domain_segments: Option<Vec<DomainSegmentConfig>>,  // None = single-domain default
    },
    RouteStandard,
    Block,
}
```

### 3.15 MeshTransportManager

```rust
pub trait MeshTransportManager: Send + Sync {
    async fn add_transport(
        &self,
        domain: DomainId,
        transport: Box<dyn MeshTransport>,
    ) -> Result<(), TransportError>;
    async fn remove_transport(
        &self,
        domain: &DomainId,
        transport_type: TransportType,
    ) -> Result<(), TransportError>;
    async fn send(
        &self,
        peer: &NodeId,
        domain: &DomainId,
        data: Bytes,
    ) -> Result<(), TransportError>;
    async fn broadcast(&self, domain: &DomainId, data: Bytes) -> Result<usize, TransportError>;
    fn list_peers(&self, domain: Option<&DomainId>) -> Vec<MeshPeerInfo>;
    fn get_transport_for_domain(&self, domain: &DomainId) -> Option<Arc<dyn MeshTransport>>;
    fn stats(&self) -> MeshStats;
}
```

---

## 4. Error Types

```rust
pub enum BgpxError {
    Network(NetworkError),
    Session(SessionError),
    Handshake(HandshakeError),
    Dht(DhtError),
    Reputation(ReputationError),
    Exit(ExitError),
    Crypto(CryptoError),
    Config(ConfigError),
    Pool(PoolError),
    Mesh(MeshError),
    RoutingPolicy(PolicyError),
    Bridge(BridgeError),
    Island(IslandError),
    Io(IoError),
}

pub enum CryptoError {
    DecryptionFailed,
    SignatureInvalid,
    KeyDerivationFailed,
    RngFailed,
    NonceReuseAttempted,
}

pub enum SessionError {
    NotFound(SessionId),
    TableFull { max: usize },
    InvalidState { session_id: SessionId, state: SessionState },
    KeyDerivationFailed,
    RehandshakeFailed,
}

pub enum BridgeError {
    DomainNotServed(DomainId),
    NoBridgeForPair(DomainId, DomainId),
    BridgeUnavailable,
    TransportError(TransportError),
    NodeNotResolvable,
    CrossDomainPathTableFull,
}

pub enum IslandError {
    IslandNotFound(String),
    DhtPublishFailed(DhtError),
    DhtFetchFailed(DhtError),
    InvalidIslandId(String),
    NoBridgeNodeOnline,
}
```

---

## 5. Async Task Structure

```rust
#[tokio::main]
async fn main() {
    let config = load_config().await?;
    let state = Arc::new(NodeState::new(config));

    tokio::select! {
        _ = run_udp_listener(state.clone())              => {},
        _ = run_mesh_transports(state.clone())           => {},
        _ = run_dht_maintenance(state.clone())           => {},
        _ = run_keepalive_monitor(state.clone())         => {},
        _ = run_advertisement_publisher(state.clone())   => {},
        _ = run_domain_bridge_publisher(state.clone())   => {},
        _ = run_island_advertisement_publisher(state.clone()) => {},
        _ = run_session_rehandshake_scheduler(state.clone()) => {},
        _ = run_path_table_cleanup(state.clone())        => {},
        _ = run_cross_domain_path_table_cleanup(state.clone()) => {},
        _ = run_sdk_socket_server(state.clone())         => {},
        _ = run_control_api_server(state.clone())        => {},
        _ = run_metrics_exporter(state.clone())          => {},
        _ = run_pt_subprocess(state.clone())             => {},
        _ = signal_handler(state.clone())                => {},
    }
}
```

### Background Task Intervals

| Task | Interval | Description |
|---|---|---|
| `run_keepalive_monitor` | 10 seconds | Check session liveness; clean path tables |
| `run_path_table_cleanup` | 30 seconds | Expire single-domain path_id entries |
| `run_cross_domain_path_table_cleanup` | 30 seconds | Expire cross-domain path_id entries |
| `run_advertisement_publisher` | 12 hours | Re-publish node advertisement to DHT |
| `run_domain_bridge_publisher` | 8 hours | Re-publish DOMAIN_ADVERTISE records |
| `run_island_advertisement_publisher` | 8 hours | Re-publish MESH_ISLAND_ADVERTISE if internet available |
| `run_dht_maintenance` | 60 minutes | Refresh stale DHT routing table buckets |
| `run_session_rehandshake_scheduler` | 10 minutes | Check session ages; trigger 24hr re-handshake |

### Session Re-handshake Scheduler

The re-handshake scheduler checks all sessions every 10 minutes:

```rust
async fn run_session_rehandshake_scheduler(state: Arc<NodeState>) {
    let mut interval = tokio::time::interval(Duration::from_secs(600));
    loop {
        interval.tick().await;
        let threshold = Duration::from_secs(
            state.config.sessions.session_rehandshake_age_hours * 3600
        );
        for session in state.session_manager.list_sessions() {
            if session.age() > threshold {
                state.handshake_handler.handle_rehandshake_initiation(&session.id).await;
            }
        }
    }
}
```

### Path Table Cleanup

Path table entries expire at session idle timeout. The single-domain and cross-domain tables are cleaned separately:

```rust
async fn run_path_table_cleanup(state: Arc<NodeState>) {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        interval.tick().await;
        state.path_table.cleanup_expired();
        // Cross-domain table only on bridge-capable nodes
        if state.domain_bridge_manager.is_some() {
            state.cross_domain_path_table.cleanup_expired();
        }
    }
}
```
