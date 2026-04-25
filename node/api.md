# BGP-X Node Internal API Specification

**Version**: 0.1.0-draft

This document specifies the internal API contracts between the major subsystems of the BGP-X node daemon. It is intended for implementors of the reference implementation and contributors building alternative implementations.

---

## 1. Overview

The BGP-X node daemon is composed of loosely coupled subsystems that communicate through well-defined internal interfaces. This document specifies those interfaces — the function signatures, data types, error conditions, and behavioral contracts for each subsystem boundary.

All interfaces described here are internal. They are not exposed over the network or to external processes. The external-facing interface is the Control API (see `/control-plane/control_api.md`).

---

## 2. Core Data Types

### 2.1 NodeId

```rust
/// A 32-byte BLAKE3 hash of an Ed25519 public key.
/// Used as the primary identifier for nodes in the DHT and routing.
pub struct NodeId([u8; 32]);

impl NodeId {
    pub fn from_public_key(key: &Ed25519PublicKey) -> Self;
    pub fn to_hex(&self) -> String;
    pub fn from_hex(s: &str) -> Result<Self, ParseError>;
    pub fn xor_distance(&self, other: &NodeId) -> NodeId;
}
```

### 2.2 SessionId

```rust
/// A 16-byte random session identifier.
pub struct SessionId([u8; 16]);

impl SessionId {
    pub fn generate() -> Self;  // Uses CSPRNG
    pub fn as_bytes(&self) -> &[u8; 16];
}
```

### 2.3 StreamId

```rust
/// A 32-bit stream identifier within a session.
pub struct StreamId(u32);

impl StreamId {
    pub fn is_client_initiated(&self) -> bool;   // odd IDs
    pub fn is_service_initiated(&self) -> bool;  // even IDs
    pub fn next_client_id(current: StreamId) -> StreamId;
    pub fn next_service_id(current: StreamId) -> StreamId;
}
```

### 2.4 SessionKey

```rust
/// A 32-byte ChaCha20-Poly1305 session key.
/// Stored in locked, zeroizable memory.
pub struct SessionKey(Zeroizing<[u8; 32]>);

impl SessionKey {
    pub fn from_handshake_material(
        dh1: &[u8; 32],
        dh2: &[u8; 32],
        session_id: &SessionId,
        node_id: &NodeId,
    ) -> Self;

    pub fn encrypt(
        &self,
        sequence: u64,
        aad: &[u8],
        plaintext: &[u8],
    ) -> Vec<u8>;

    pub fn decrypt(
        &self,
        sequence: u64,
        aad: &[u8],
        ciphertext: &[u8],
    ) -> Result<Vec<u8>, DecryptionError>;
}

impl Drop for SessionKey {
    fn drop(&mut self) {
        // Zeroize key material on drop
    }
}
```

### 2.5 NodeAdvertisement

```rust
pub struct NodeAdvertisement {
    pub version: u8,
    pub node_id: NodeId,
    pub public_key: Ed25519PublicKey,
    pub roles: Vec<NodeRole>,
    pub endpoints: Vec<Endpoint>,
    pub exit_policy: Option<ExitPolicy>,
    pub bandwidth_mbps: f32,
    pub latency_ms: f32,
    pub uptime_pct: f32,
    pub region: Region,
    pub country: CountryCode,
    pub asn: u32,
    pub operator_id: Option<Ed25519PublicKey>,
    pub protocol_versions_min: u16,
    pub protocol_versions_max: u16,
    pub extensions: ExtensionFlags,
    pub signed_at: DateTime<Utc>,
    pub expires_at: DateTime<Utc>,
    pub signature: Ed25519Signature,
}

impl NodeAdvertisement {
    pub fn verify(&self) -> Result<(), AdvertisementError>;
    pub fn is_expired(&self) -> bool;
    pub fn canonical_json(&self) -> String;
    pub fn to_wire(&self) -> Vec<u8>;
    pub fn from_wire(bytes: &[u8]) -> Result<Self, ParseError>;
}
```

---

## 3. Subsystem Interfaces

### 3.1 ForwardingPipeline

The forwarding pipeline is the hot path. Its interface is minimal to avoid overhead.

```rust
pub trait ForwardingPipeline: Send + Sync {

    /// Process a received UDP datagram.
    /// Called by receiver threads for every incoming packet.
    /// Must be non-blocking and complete in < 1ms (p99).
    fn process_packet(
        &self,
        source_addr: SocketAddr,
        data: &[u8],
    ) -> ProcessResult;

    /// Enqueue a packet for transmission.
    /// Called by dispatch handlers after processing.
    fn enqueue_packet(
        &self,
        dest_addr: SocketAddr,
        data: Bytes,
    ) -> Result<(), QueueFullError>;

    /// Register a new session.
    /// Called by the handshake handler after successful completion.
    fn register_session(
        &self,
        session_id: SessionId,
        session_key: SessionKey,
        remote_addr: SocketAddr,
    ) -> Result<(), SessionError>;

    /// Remove a session.
    fn remove_session(&self, session_id: &SessionId);

    /// Get current forwarding statistics.
    fn stats(&self) -> ForwardingStats;
}

pub enum ProcessResult {
    Forwarded,
    Delivered,         // For local BGP-X native service delivery
    DroppedDecrypt,    // Decryption failure
    DroppedReplay,     // Replay detection
    DroppedUnknown,    // Unknown session
    DroppedInvalid,    // Malformed packet
    HandshakeMessage,  // Route to handshake handler
    DhtMessage,        // Route to DHT handler
    KeepaliveMessage,  // Route to keepalive handler
}
```

### 3.2 SessionManager

```rust
pub trait SessionManager: Send + Sync {

    /// Look up a session by ID.
    /// Must be constant-time whether session exists or not.
    fn get_session(&self, id: &SessionId) -> Option<Arc<Session>>;

    /// Create a new session (called after handshake completes).
    fn create_session(
        &self,
        id: SessionId,
        key: SessionKey,
        remote_node_id: Option<NodeId>,
        remote_addr: SocketAddr,
    ) -> Result<Arc<Session>, SessionError>;

    /// Close a session and clean up all associated streams.
    fn close_session(
        &self,
        id: &SessionId,
        reason: CloseReason,
    );

    /// List all active sessions (for monitoring; no identifying details).
    fn list_sessions(&self) -> Vec<SessionSummary>;

    /// Current session count.
    fn session_count(&self) -> usize;
}

pub struct Session {
    pub id: SessionId,
    pub key: SessionKey,
    pub state: AtomicSessionState,
    pub sequence_window: Mutex<SequenceWindow>,
    pub outbound_sequence: AtomicU64,
    pub last_seen: AtomicU64,   // Unix timestamp
    pub bytes_relayed: AtomicU64,
    pub packets_relayed: AtomicU64,
    pub decryption_failures: AtomicU32,
    pub streams: DashMap<StreamId, Arc<Stream>>,
}
```

### 3.3 HandshakeHandler

```rust
pub trait HandshakeHandler: Send + Sync {

    /// Process a HANDSHAKE_INIT message.
    fn handle_init(
        &self,
        source_addr: SocketAddr,
        payload: &[u8],
    ) -> HandshakeResult;

    /// Process a HANDSHAKE_DONE message.
    fn handle_done(
        &self,
        session_id: &SessionId,
        payload: &[u8],
    ) -> HandshakeResult;
}

pub enum HandshakeResult {
    /// Handshake in progress; response has been sent.
    InProgress,
    /// Handshake complete; session is now established.
    Complete(SessionId),
    /// Handshake failed; reason is logged internally.
    Failed,
}
```

### 3.4 DhtSubsystem

```rust
pub trait DhtSubsystem: Send + Sync {

    /// Find the k nodes closest to a target ID.
    async fn find_nodes(
        &self,
        target: &NodeId,
        max_results: usize,
    ) -> Vec<NodeContact>;

    /// Store a record in the DHT.
    async fn put(
        &self,
        key: &[u8; 32],
        value: Vec<u8>,
    ) -> Result<usize, DhtError>;  // Returns number of storage nodes reached

    /// Retrieve a record from the DHT.
    async fn get(
        &self,
        key: &[u8; 32],
    ) -> Result<Option<Vec<u8>>, DhtError>;

    /// Publish this node's own advertisement.
    async fn publish_advertisement(
        &self,
        advertisement: &NodeAdvertisement,
    ) -> Result<usize, DhtError>;

    /// Get current DHT statistics.
    fn stats(&self) -> DhtStats;

    /// Get the current estimated network size.
    fn estimated_network_size(&self) -> u64;
}

pub struct NodeContact {
    pub node_id: NodeId,
    pub addr: SocketAddr,
}
```

### 3.5 ReputationSystem

```rust
pub trait ReputationSystem: Send + Sync {

    /// Record a reputation event for a node.
    fn record_event(
        &self,
        node_id: &NodeId,
        event: ReputationEvent,
    );

    /// Get the reputation score for a node.
    fn get_score(&self, node_id: &NodeId) -> f32;

    /// Check if a node is blacklisted.
    fn is_blacklisted(&self, node_id: &NodeId) -> bool;

    /// Manually blacklist a node.
    fn blacklist(
        &self,
        node_id: &NodeId,
        reason: BlacklistReason,
        duration: Option<Duration>,
    );

    /// Remove a manual blacklist entry (cannot remove permanent entries).
    fn unblacklist(
        &self,
        node_id: &NodeId,
    ) -> Result<(), ReputationError>;

    /// Flush the reputation database to disk.
    async fn flush(&self) -> Result<(), IoError>;
}

pub enum ReputationEvent {
    RelaySuccess,
    RelayFailure,
    KeepaliveSuccess,
    KeepaliveTimeout,
    HandshakeSuccess,
    HandshakeFailure,
    LatencyWithinAdvertised,
    LatencyExceeded,
    AdvertisementInconsistency,
    ProtocolViolation,
    SuspiciousBehavior,
    BandwidthWithinAdvertised,
    UptimeConsistent,
}
```

### 3.6 ExitPolicyEngine (Exit Nodes Only)

```rust
pub trait ExitPolicyEngine: Send + Sync {

    /// Check whether a destination is permitted by the exit policy.
    fn allows(
        &self,
        protocol: Protocol,
        destination: &Destination,
        port: u16,
    ) -> PolicyDecision;

    /// Establish an outbound connection to a clearnet destination.
    async fn connect(
        &self,
        destination: &Destination,
        port: u16,
        protocol: Protocol,
    ) -> Result<OutboundConnection, ExitError>;

    /// Get the signed exit policy (for inclusion in advertisement).
    fn signed_policy(&self) -> &SignedExitPolicy;
}

pub enum PolicyDecision {
    Allow,
    DenyProtocol,
    DenyPort,
    DenyDestination,
}

pub enum ExitError {
    PolicyDenied(PolicyDecision),
    ConnectionRefused,
    DnsResolutionFailed,
    Timeout,
    MaxConnectionsReached,
}
```

### 3.7 MetricsCollector

```rust
pub trait MetricsCollector: Send + Sync {

    fn increment(&self, counter: Counter);
    fn gauge(&self, gauge: Gauge, value: f64);
    fn histogram(&self, histogram: Histogram, value: f64);

    /// Export all metrics in Prometheus text format.
    fn export_prometheus(&self) -> String;

    /// Get a snapshot of all metrics as a structured value.
    fn snapshot(&self) -> MetricsSnapshot;
}

pub enum Counter {
    PacketsRelayed,
    BytesRelayed,
    SessionsOpened,
    SessionsClosed,
    HandshakesSucceeded,
    HandshakesFailed,
    KeepaliveTimeouts,
    DecryptionFailures,
    ReplayAttempts,
    ExitPolicyDenials,
    DhtQueriesSent,
    DhtQueriesReceived,
}

pub enum Gauge {
    ActiveSessions,
    ActiveStreams,
    DhtRoutingTableSize,
    BlacklistedNodes,
    OutputQueueDepth,
}

pub enum Histogram {
    PacketProcessingLatencyUs,
    HandshakeLatencyMs,
    PathConstructionLatencyMs,
    RttMs,
}
```

---

## 4. Error Types

```rust
/// Top-level error type for the BGP-X node daemon.
pub enum BgpxError {
    Network(NetworkError),
    Session(SessionError),
    Handshake(HandshakeError),
    Dht(DhtError),
    Reputation(ReputationError),
    Exit(ExitError),
    Crypto(CryptoError),
    Config(ConfigError),
    Io(IoError),
}

pub enum NetworkError {
    SocketBindFailed { addr: SocketAddr, cause: IoError },
    SendFailed { dest: SocketAddr, cause: IoError },
    InvalidPacket { reason: &'static str },
}

pub enum SessionError {
    NotFound(SessionId),
    TableFull { max: usize },
    InvalidState { session_id: SessionId, state: SessionState },
    KeyDerivationFailed,
}

pub enum HandshakeError {
    InvalidInit { reason: &'static str },
    InvalidDone { reason: &'static str },
    Timeout(SessionId),
    UnknownSession(SessionId),
}

pub enum CryptoError {
    DecryptionFailed,
    SignatureInvalid,
    KeyDerivationFailed,
    RngFailed,
}
```

---

## 5. Async Task Structure

The daemon runs the following long-lived async tasks within the Tokio runtime:

```rust
#[tokio::main]
async fn main() {
    let config = load_config().await?;
    let state = Arc::new(NodeState::new(config));

    tokio::select! {
        _ = run_udp_listener(state.clone())          => {},
        _ = run_dht_maintenance(state.clone())        => {},
        _ = run_keepalive_monitor(state.clone())      => {},
        _ = run_advertisement_publisher(state.clone()) => {},
        _ = run_control_api_server(state.clone())    => {},
        _ = run_metrics_exporter(state.clone())      => {},
        _ = signal_handler(state.clone())            => {},
    }
}
```

Each task communicates with others through:
- `Arc<NodeState>` — shared immutable configuration and mutable subsystem handles
- Tokio channels (`mpsc`, `broadcast`) — for event notification
- `DashMap` and atomics — for high-concurrency shared state (session table, counters)
