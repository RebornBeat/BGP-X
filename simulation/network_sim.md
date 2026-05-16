# BGP-X Network Simulation Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X network simulation framework (`bgpx-sim`) — the tooling used to test protocol behavior, validate path construction, measure anonymity properties, test cross-domain routing, and stress-test the overlay network before and during deployment.

---

## 1. Overview

Network simulation serves multiple purposes in the BGP-X development lifecycle:

1. **Protocol validation**: Verify that the protocol behaves correctly under normal conditions, edge cases, and adversarial inputs without requiring a live network
2. **Performance characterization**: Measure throughput, latency, and resource usage at scale before deploying hardware
3. **Anonymity analysis**: Evaluate the effectiveness of privacy protections under controlled adversary scenarios
4. **Cross-domain validation**: Test routing domain transitions, domain bridge behavior, and mesh-to-clearnet connectivity
5. **Mesh deployment testing**: Validate mesh island operation with realistic radio constraints (duty cycle, MTU fragmentation)

The BGP-X simulation framework is a Rust library (`bgpx-sim`) that instantiates multiple node and client instances in a single process, connected by simulated network links with configurable latency, packet loss, bandwidth, and domain characteristics.

---

## 2. Architecture

### 2.1 Full Architecture Diagram

```
bgpx-sim runtime
│
├── Virtual Network Layer (multi-domain)
│   ├── Domain registry (which routing domains exist)
│   ├── Node registry (NodeID → SimNode with domain membership)
│   ├── Link registry (per domain, inter-domain via bridge nodes)
│   ├── Packet router (delivers packets between SimNodes)
│   └── Cross-domain packet router
│
├── SimNode instances (1..N)
│   ├── Real bgpx-node code (no mocking)
│   ├── Simulated UDP socket (routes through virtual network)
│   ├── Simulated DHT (shared in-memory DHT for all nodes)
│   ├── Domain-aware configuration
│   └── Domain bridge simulation (bridge-capable nodes)
│
├── SimClient instances (1..M)
│   ├── Real bgpx-client code (no mocking)
│   └── Simulated network interface
│
├── SimPool
│   └── Pool advertisement simulation
│
├── SimGateway
│   └── Mesh-to-internet bridge simulation
│
├── SimMeshIsland
│   ├── Island topology (nodes, radio links)
│   └── Local mesh DHT
│
├── Scenario runner
│   ├── Scenario definition (TOML)
│   ├── Event scheduler (inject events at specific times)
│   ├── Domain topology specification
│   ├── Cross-domain event simulation
│   └── Results collector
│
├── Traffic generator (client simulation)
│
└── Analysis engine
    ├── Path analysis (distribution, diversity)
    ├── Anonymity metrics (correlation success rate under adversary)
    ├── Pool analysis
    ├── Cross-domain path analysis
    ├── Domain reachability metrics
    ├── Bridge node availability metrics
    └── Performance metrics (throughput, latency, packet loss)
```

### 2.1 Component Interaction

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SCENARIO RUNNER                                    │
│  - Loads TOML scenario definition                                           │
│  - Instantiates SimNodes, SimClients, SimMeshIslands                        │
│  - Schedules events                                                         │
│  - Collects metrics                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VIRTUAL NETWORK LAYER                                │
│                                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │ Clearnet Domain  │  │ BGP-X Overlay    │  │ Mesh Domains     │          │
│  │ (simulated BGP)  │  │ (virtual)        │  │ (simulated radio)│          │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘          │
│           │                     │                     │                     │
│           └─────────────────────┼─────────────────────┘                     │
│                                 │                                           │
│                        Domain Bridge Nodes                                   │
│                        (link domains)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ANALYSIS ENGINE                                    │
│  - Path correctness                                                         │
│  - Diversity enforcement                                                    │
│  - Cross-domain delivery                                                    │
│  - Anonymity metrics                                                        │
│  - Timing correlation resistance                                            │
│  - Performance metrics                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Virtual Network Layer

### 3.1 Link Properties

Each link between two simulated nodes has configurable properties:

```rust
pub struct LinkProperties {
    /// One-way latency in milliseconds (sampled from distribution)
    pub latency_ms: LatencyDistribution,

    /// Packet loss rate (0.0 = no loss, 1.0 = total loss)
    pub packet_loss_rate: f64,

    /// Bandwidth cap in Mbps (0.0 = unlimited)
    pub bandwidth_mbps: f64,

    /// Jitter model (adds random variation to latency)
    pub jitter: JitterModel,

    /// Transport type for this link
    pub transport_type: TransportType,

    /// Maximum transmission unit for this link
    pub mtu: usize,

    /// Whether DPI filtering is active on this link
    pub dpi_filtered: bool,

    /// Domain pair for cross-domain links (None for intra-domain)
    pub domain_pair: Option<(DomainId, DomainId)>,

    /// Additional latency at bridge transitions (cross-domain only)
    pub bridge_transition_latency_ms: Option<f64>,
}

pub enum LatencyDistribution {
    /// Fixed latency
    Constant(f64),

    /// Uniform random in [min, max]
    Uniform { min: f64, max: f64 },

    /// Normal distribution
    Normal { mean: f64, std_dev: f64 },

    /// Empirical distribution (from real measurement data)
    Empirical(Vec<f64>),

    /// LoRa duty cycle model
    LoRaDutyCycle { 
        base_ms: f64, 
        duty_cycle_pct: f64 
    },

    /// LoRa radio model with spreading factor
    LoRaModel { 
        spreading_factor: u8, 
        bandwidth_khz: u32 
    },

    /// LEO satellite (Starlink, OneWeb, Kuiper)
    SatelliteLeo { 
        base_ms: f64 
    },

    /// GEO satellite (Inmarsat, HughesNet, Viasat)
    SatelliteGeo { 
        base_ms: f64 
    },

    /// Satellite model by orbit type
    SatelliteModel { 
        orbit_type: OrbitType 
    },
}

pub enum JitterModel {
    /// No jitter
    None,

    /// Add uniform random jitter in [-jitter_ms, +jitter_ms]
    Uniform(f64),

    /// Pareto-distributed jitter (models real internet jitter)
    Pareto { scale: f64, shape: f64 },
}

pub enum TransportType {
    UdpIp,
    WifiMesh,
    Lora,
    BluetoothLe,
    EthernetP2P,
    Satellite,
}

pub enum OrbitType {
    Leo,   // 20-60ms
    Meo,   // 150ms
    Geo,   // 600ms+
}
```

### 3.2 Network Topologies

The simulation framework includes built-in topology generators:

```rust
pub enum Topology {
    /// All nodes connected to all others (for small-scale tests)
    FullMesh,

    /// Random graph with specified connectivity (Erdős–Rényi)
    Random {
        node_count: usize,
        edge_probability: f64,
    },

    /// Scale-free network (Barabási–Albert, resembles real internet)
    ScaleFree {
        node_count: usize,
        edges_per_new_node: usize,
    },

    /// Geographic topology: nodes grouped into regions with
    /// intra-region low latency and inter-region higher latency
    Geographic {
        regions: Vec<RegionConfig>,
        intra_region_latency_ms: f64,
        inter_region_latency_ms: HashMap<(Region, Region), f64>,
    },

    /// Load from a topology file (JSON or TOML)
    FromFile(PathBuf),

    /// Mesh island (single transport)
    MeshIsland {
        transport: TransportType,
        node_count: usize,
        topology: MeshTopologyType,
    },

    /// Cross-domain topology (clearnet + mesh islands + bridges)
    CrossDomain {
        clearnet_nodes: usize,
        mesh_islands: Vec<MeshIslandConfig>,
        domain_bridge_count_per_pair: usize,
        inter_island_bridges: Vec<(String, String)>,
        island_bridge_nodes_per_island: usize,
    },

    /// Multiple mesh islands with optional inter-island links
    MultiIslandMesh {
        islands: Vec<MeshIslandConfig>,
        islands_connected_via_clearnet: bool,
        islands_connected_via_direct_lora: bool,
        gateway_nodes: Vec<GatewayNodeConfig>,
    },
}

pub struct MeshIslandConfig {
    pub island_id: String,
    pub node_count: usize,
    pub primary_transport: TransportType,
    pub gateway_count: usize,
    pub internet_connectivity: bool,
    pub latitude_lon: Option<(f64, f64)>,
}

pub struct GatewayNodeConfig {
    pub node_id: NodeId,
    pub island_id: String,
    pub has_wan: bool,
    pub wan_type: Option<TransportType>,
}

pub enum MeshTopologyType {
    FullMesh,
    Chain,
    Star,
    Random { edge_probability: f64 },
}
```

### 3.3 Domain-Aware Virtual Network

```rust
pub struct VirtualNetwork {
    /// Domain registry
    domains: HashMap<DomainId, RoutingDomain>,

    /// Node registry with domain membership
    nodes: HashMap<NodeId, SimNode>,

    /// Link registry: (from_node, to_node) -> LinkProperties
    links: HashMap<(NodeId, NodeId), LinkProperties>,

    /// Domain bridge nodes
    bridges: HashMap<(DomainId, DomainId), Vec<NodeId>>,

    /// Mesh island topologies
    mesh_islands: HashMap<String, MeshIslandNetwork>,

    /// Cross-domain path table (simulated)
    cross_domain_paths: HashMap<PathId, CrossDomainPathEntry>,
}

pub struct CrossDomainPathEntry {
    pub path_id: PathId,
    pub source_domain: DomainId,
    pub predecessor_in_source_domain: SocketAddr,
    pub target_domain: DomainId,
    pub predecessor_in_target_domain: SocketAddr,
}
```

### 3.4 Simulated Time

The simulation can run in two time modes:

```rust
pub enum TimeMode {
    /// Wall clock time — simulation runs in real time
    /// Used for integration tests and latency characterization
    WallClock,

    /// Simulated time — events are processed as fast as possible
    /// Used for large-scale anonymity analysis and stress testing
    /// Simulated time can run 10–1000× faster than real time
    Simulated {
        /// Maximum simulated time to run
        duration: Duration,
        /// Whether to advance time automatically or manually
        advance_mode: TimeAdvanceMode,
    },
}

pub enum TimeAdvanceMode {
    /// Automatically advance simulated clock
    Automatic,
    /// Manually advance clock via API call (for testing)
    Manual,
}
```

---

## 4. Scenario Definition

Scenarios are defined in TOML configuration files that specify the network topology, node configuration, client behavior, adversary configuration, cross-domain setup, and measurement targets.

### 4.1 Basic 4-Hop Single-Domain Scenario

```toml
# BGP-X Simulation Scenario: Basic 4-hop path test
[scenario]
name = "basic_4hop_path"
description = "Verify 4-hop path construction and data delivery"
time_mode = "wall_clock"
duration_seconds = 60

[network]
topology = "geographic"

[[network.regions]]
name = "NA"
node_count = 20
intra_region_latency_ms = 10.0
intra_region_loss_rate = 0.001

[[network.regions]]
name = "EU"
node_count = 25
intra_region_latency_ms = 8.0
intra_region_loss_rate = 0.001

[[network.regions]]
name = "AP"
node_count = 15
intra_region_latency_ms = 12.0
intra_region_loss_rate = 0.002

[network.inter_region_latency_ms]
"NA-EU" = 85.0
"NA-AP" = 155.0
"EU-AP" = 130.0

[nodes]
# All nodes act as relay + entry + discovery
default_roles = ["relay", "entry", "discovery"]

# 5 designated exit nodes (one per region minimum)
[[nodes.exit]]
region = "EU"
count = 2
exit_policy = "minimal_risk"

[[nodes.exit]]
region = "NA"
count = 2
exit_policy = "minimal_risk"

[[nodes.exit]]
region = "AP"
count = 1
exit_policy = "minimal_risk"

[clients]
count = 10
path_config = { hops = 4, min_reputation = 60.0 }

[workload]
# Each client sends 1 MB of data to a simulated clearnet destination
type = "bulk_transfer"
data_size_bytes = 1_048_576
destination = "simulated://clearnet-server:443"

[measurements]
path_construction_latency = true
data_delivery_latency = true
throughput = true
path_diversity = true
keepalive_success_rate = true

[assertions]
# These must pass for the scenario to be considered successful
path_construction_success_rate_min = 0.99
data_delivery_success_rate_min = 0.99
path_asn_diversity = true          # All paths must have unique ASNs
path_country_diversity = true      # All paths must have unique countries
mean_throughput_mbps_min = 10.0
```

### 4.2 Multi-Pool Path Scenario

```toml
[scenario]
name = "basic_4hop_pool_path"
description = "4-hop multi-pool path with ECH"
time_mode = "wall_clock"
duration_seconds = 60

[network]
topology = "geographic"

[[network.regions]]
name = "NA"
node_count = 20
intra_region_latency_ms = 10.0
intra_region_loss_rate = 0.001

[[network.regions]]
name = "EU"
node_count = 25
intra_region_latency_ms = 8.0

[network.inter_region_latency_ms]
"NA-EU" = 85.0
"NA-AP" = 155.0
"EU-AP" = 130.0

[nodes]
default_roles = ["relay", "entry", "discovery"]

[[nodes.exit]]
region = "EU"
count = 3
exit_policy = "minimal_risk"
ech_capable = true

[[nodes.pool]]
pool_id = "trusted-eu-exits"
pool_type = "curated"
member_nodes = "eu_exit"  # References exit nodes above
min_reputation = 70.0

[clients]
count = 10
path_config = {
    segments = [
        { pool = "bgpx-default", hops = 3 },
        { pool = "trusted-eu-exits", hops = 1, exit = true }
    ],
    require_ech = true
}

[destinations]
[[destinations.ech_capable]]
domain = "example.com"
ech_enabled = true

[measurements]
path_construction_latency = true
ech_negotiation_success_rate = true
pool_diversity = true

[assertions]
path_construction_success_rate_min = 0.99
ech_negotiation_success_rate_min = 0.95
```

### 4.3 Cross-Domain Scenario

```toml
[scenario]
name = "clearnet_to_mesh_service"
duration_seconds = 300
random_seed = 42

[network]
topology = "cross_domain"

[[network.mesh_islands]]
island_id = "lima-district-1"
node_count = 15
primary_transport = "wifi_mesh"
gateway_count = 2
internet_connectivity = true

[[network.mesh_islands]]
island_id = "bogota-district-1"
node_count = 10
primary_transport = "lora"
gateway_count = 1
internet_connectivity = true

[network.inter_island_bridges]
direct_lora = [["lima-district-1", "bogota-district-1"]]

[clients]
count = 20
clearnet_client_fraction = 0.5
mesh_client_fraction = 0.5
mesh_island = "lima-district-1"

cross_domain_path_config = {
    domain_segments = [
        { type = "segment", domain = "clearnet", hops = 2 },
        { type = "bridge", from_domain = "clearnet", to_domain = "mesh:lima-district-1" },
        { type = "segment", domain = "mesh:lima-district-1", hops = 2 }
    ]
}

[adversary]
enabled = false
type = "passive"
control_fraction = 0.0

[measurements]
path_construction_success_rate = true
cross_domain_delivery_rate = true
timing_correlation_resistance = false
bridge_failover_time_ms = true
n_hop_unlimited_verified = true

[pass_criteria]
path_construction_success_rate_min = 0.95
cross_domain_delivery_rate_min = 0.95
bridge_failover_time_max_ms = 5000
n_hop_unlimited_violations = 0
```

### 4.4 Mesh-Only Island Scenario

```toml
[scenario]
name = "mesh_island_wifi"
duration_seconds = 120

[network]
topology = "mesh_island"

[[network.mesh_islands]]
island_id = "community-mesh-1"
node_count = 25
primary_transport = "wifi_mesh"
gateway_count = 1
internet_connectivity = true

[network.link_properties]
wifi_mesh_latency_ms = 15.0
wifi_mesh_loss_rate = 0.005

[nodes]
default_roles = ["relay", "entry", "discovery"]

[[nodes.gateway]]
island = "community-mesh-1"
has_wan = true

[clients]
count = 10
mesh_island = "community-mesh-1"

[workload]
type = "mesh_local_transfer"
data_size_bytes = 100_000

[measurements]
intra_mesh_delivery_rate = true
mesh_hop_count = true
duty_cycle_compliance = true
fragment_reassembly_success = true

[assertions]
intra_mesh_delivery_rate_min = 0.98
duty_cycle_violations = 0
```

---

## 5. Simulation Event Types

```rust
pub enum SimEvent {
    /// Take a node offline (simulate crash or network failure)
    NodeOffline(NodeId),

    /// Bring a node back online
    NodeOnline(NodeId),

    /// Introduce a new node to the network
    NodeJoin(NodeConfig),

    /// Partition the network (disconnect two regions)
    NetworkPartition {
        region_a: Region,
        region_b: Region,
    },

    /// Heal a network partition
    NetworkPartitionHeal {
        region_a: Region,
        region_b: Region,
    },

    /// Change link properties between two nodes
    LinkPropertyChange {
        from: NodeId,
        to: NodeId,
        new_properties: LinkProperties,
    },

    /// Pool curator key rotation event
    PoolCuratorKeyRotation {
        pool_id: String,
        reason: RotationReason,
    },

    /// Pool curator node goes offline
    PoolCuratorOffline(String),

    /// Gateway/bridge node goes offline
    GatewayBridgeOffline {
        gateway_node_id: NodeId,
    },

    /// Enable DPI filtering on specified links (for PT simulation)
    DpiFilterEnable {
        affected_links: Vec<(NodeId, NodeId)>,
    },

    /// Disable DPI filtering
    DpiFilterDisable {
        affected_links: Vec<(NodeId, NodeId)>,
    },

    /// Start a workload from a client
    StartWorkload {
        client: ClientId,
        workload: Workload,
    },

    /// Domain bridge node transition (cross-domain simulation)
    DomainBridgeTransition {
        bridge_node_id: NodeId,
        from_domain: DomainId,
        to_domain: DomainId,
    },

    /// Mesh island goes offline (loses all bridges)
    MeshIslandOffline {
        island_id: String,
    },

    /// Mesh island comes back online
    MeshIslandOnline {
        island_id: String,
    },

    /// Inject adversarial node
    AdversaryNodeJoin {
        node_config: NodeConfig,
        adversary_type: AdversaryType,
    },
}

pub enum RotationReason {
    Scheduled,
    CompromiseSuspected,
    CompromiseConfirmed,
    OperatorTransfer,
    HsmReplacement,
}

pub enum AdversaryType {
    PassiveObserver,
    ActiveMitm,
    MaliciousRelay,
    MaliciousBridge,
    SybilAttacker,
}
```

---

## 6. Simulation Runner API

```rust
pub struct Simulator {
    config: SimulatorConfig,
    nodes: HashMap<NodeId, SimNode>,
    clients: Vec<SimClient>,
    network: VirtualNetwork,
    time: SimulatedClock,
    metrics: MetricsCollector,
    pools: HashMap<String, SimPool>,
    mesh_islands: HashMap<String, SimMeshIsland>,
}

impl Simulator {

    /// Create a new simulator from a scenario file.
    pub async fn from_scenario(path: &Path) -> Result<Self, SimError>;

    /// Add a node to the simulation.
    pub async fn add_node(
        &mut self,
        config: NodeConfig,
        position: NetworkPosition,
    ) -> NodeId;

    /// Add a client to the simulation.
    pub async fn add_client(
        &mut self,
        config: ClientConfig,
        position: NetworkPosition,
    ) -> ClientId;

    /// Add a mesh island to the simulation.
    pub async fn add_mesh_island(
        &mut self,
        config: MeshIslandConfig,
    ) -> String;

    /// Add a domain bridge between two domains.
    pub async fn add_domain_bridge(
        &mut self,
        node_id: NodeId,
        from_domain: DomainId,
        to_domain: DomainId,
    );

    /// Run the simulation to completion.
    pub async fn run(&mut self) -> SimulationResult;

    /// Inject an event at a specific simulated time.
    pub fn schedule_event(&mut self, at: Duration, event: SimEvent);

    /// Get current simulation metrics.
    pub fn metrics(&self) -> &MetricsSnapshot;
}

pub enum NetworkPosition {
    Random,
    Region(String),
    Coordinates { lat: f64, lon: f64 },
    NearNode(NodeId),
}

pub struct Workload {
    pub workload_type: WorkloadType,
    pub data_size_bytes: usize,
    pub destination: Destination,
    pub duration: Option<Duration>,
}

pub enum WorkloadType {
    BulkTransfer,
    RequestResponse { requests_per_second: f64 },
    Streaming { bitrate_mbps: f64 },
    Interactive { session_count: usize },
}

pub enum Destination {
    Clearnet { address: String },
    MeshService { island_id: String, service_id: String },
    BgpxService { service_id: String },
}
```

---

## 7. Measurement and Assertions

### 7.1 Built-in Measurements

The simulation framework automatically collects:

| Measurement | Description |
|---|---|
| `path_construction_latency_ms` | Time from request to ready path |
| `path_construction_success_rate` | Fraction of successful path constructions |
| `data_delivery_latency_ms` | End-to-end RTT for a request-response cycle |
| `data_delivery_success_rate` | Fraction of successful deliveries |
| `throughput_mbps` | Sustained throughput per client |
| `keepalive_success_rate` | Fraction of keepalives that received a response |
| `path_hops_distribution` | Histogram of actual path lengths |
| `path_asn_unique_rate` | Fraction of paths with all-unique ASNs |
| `path_country_unique_rate` | Fraction of paths with all-unique countries |
| `path_operator_unique_rate` | Fraction of paths with all-unique operators |
| `session_establishment_time_ms` | Time for complete N-hop handshake |
| `node_discovery_time_ms` | Time to populate node database from DHT |
| `dht_query_latency_ms` | Per-query DHT lookup latency |
| `ech_negotiation_success_rate` | ECH negotiation success fraction |
| `cover_traffic_indistinguishability` | Statistical test result |
| `return_path_routing_success` | Correct path_id routing |
| `fragment_reassembly_success` | Correct LoRa reassembly |
| `pool_curator_rotation_success` | Rotation accepted and applied |
| `geo_plausibility_score_accuracy` | Scores match expected values |
| `cross_domain_path_construction_latency_ms` | Time to build cross-domain path |
| `cross_domain_delivery_success_rate` | End-to-end success for cross-domain paths |
| `domain_bridge_transition_latency_ms` | Time spent at domain bridge transitions |
| `mesh_island_reachability_pct` | Fraction of time islands reachable from clearnet |
| `bridge_node_availability_pct` | Fraction of time bridge pairs have active bridges |
| `n_hop_unlimited_violations` | Paths rejected for hop count (MUST be 0) |
| `any_to_any_routing_coverage_pct` | Domain→domain combinations successfully routed |
| `unified_dht_coverage_pct` | Nodes discoverable from all domains |
| `island_failover_time_ms` | Time to route via secondary bridge after primary fails |
| `pool_diversity` | Correct pools per segment |
| `stream_id_parity_violations` | Client odd / service even enforcement violations |
| `session_rehandshake_success` | Re-handshake at 24hr success rate |
| `node_withdrawal_propagation_time_ms` | Time for NODE_WITHDRAW to propagate |
| `duty_cycle_compliance` | LoRa duty cycle violations |

### 7.2 Assertion Syntax

Assertions are evaluated at the end of a scenario run:

```toml
[assertions]
# Numeric comparisons
path_construction_success_rate_min = 0.99
mean_throughput_mbps_min = 10.0
p99_data_delivery_latency_ms_max = 500.0
cross_domain_delivery_rate_min = 0.95
bridge_failover_time_max_ms = 5000

# Boolean properties (verified for every path constructed)
path_asn_diversity = true
path_country_diversity = true
path_operator_diversity = true
n_hop_unlimited_violations = 0

# Distribution assertions
path_hops_mode = 4              # Most common path length must be 4
path_hops_min = 3               # No path shorter than 3 hops
```

### 7.3 Result Format

```rust
pub struct SimulationResult {
    pub scenario_name: String,
    pub duration_simulated: Duration,
    pub duration_wall_clock: Duration,
    pub assertions_passed: bool,
    pub assertion_failures: Vec<AssertionFailure>,
    pub measurements: HashMap<String, MeasurementSummary>,
    pub events_processed: u64,
    pub packets_routed: u64,
    pub cross_domain_transitions: u64,
}

pub struct MeasurementSummary {
    pub count: u64,
    pub mean: f64,
    pub std_dev: f64,
    pub min: f64,
    pub max: f64,
    pub p50: f64,
    pub p95: f64,
    pub p99: f64,
}

pub struct AssertionFailure {
    pub assertion_name: String,
    pub expected: String,
    pub actual: String,
    pub details: String,
}
```

---

## 8. Standard Scenario Library

The following scenarios are included in the BGP-X simulation framework and MUST pass before any release.

### 8.1 Correctness Scenarios

| Scenario | Description | Pass Criteria |
|---|---|---|
| `basic_3hop` | 3-hop single pool, 10 clients | 100% delivery, correct path structure |
| `basic_4hop` | 4-hop single pool, 10 clients | 100% delivery, correct path structure |
| `multi_pool_2segment` | 2-segment multi-pool path | Correct cross-segment diversity |
| `multi_pool_3segment` | 3-segment double-exit | Exit isolation verified |
| `ech_required` | Streams requiring ECH | ECH negotiated when available |
| `ech_unavailable` | ECH_REQUIRED but destination no ECH | Stream fails with correct error |
| `pool_insufficient` | Pool has too few nodes | Fallback to default (if configured) |
| `pool_curator_rotation` | Key rotation during active paths | Paths continue; members re-verified |
| `return_path_routing` | path_id return routing | Return traffic correctly forwarded |
| `cover_traffic` | COVER indistinguishable from RELAY | Statistical test passes |
| `stream_id_parity` | Client odd / service even enforcement | Violations → RST |
| `session_rehandshake` | Re-handshake at 24hr | New keys; old keys zeroized |
| `node_withdrawal` | NODE_WITHDRAW propagation | Node removed from DHT within 30s |
| `mesh_basic` | WiFi mesh 3-hop | Delivery via mesh transport |
| `mesh_lora` | LoRa transport (simulated duty cycle) | Delivery despite rate limiting |
| `mesh_fragment` | LoRa MTU fragmentation | Reassembly correct |
| `mesh_gateway` | Mesh island + gateway | Clearnet access via gateway |
| `mesh_island_bridging` | Two islands + internet bridge | Cross-island delivery |
| `pt_bypass_dpi` | PT with DPI-filtered links | Traffic passes via obfs4 |
| `geo_plausibility` | RTT measurement and scoring | Scores match expected buckets |
| `diversity_enforcement` | Path diversity constraints | All diversity checks pass |
| `replay_protection` | Inject replayed packets | All replays detected |
| `auth_failure` | Inject packets with corrupt auth tags | All rejected silently |

### 8.2 Resilience Scenarios

| Scenario | Description | Pass Criteria |
|---|---|---|
| `node_churn_10pct` | 10% nodes offline randomly | >95% path construction success |
| `node_churn_30pct` | 30% nodes offline randomly | >85% path construction success |
| `node_failure_in_path` | Node fails mid-transfer | Path rebuilt, transfer completes |
| `exit_failure_regional` | All exits in one region fail | Reroute to other regions |
| `pool_curator_failure` | Pool curator node goes offline | Pool still usable (cached) |
| `gateway_failure` | Mesh gateway offline | Mesh community loses clearnet only |
| `bootstrap_node_failure` | 50% bootstrap nodes offline | New nodes still join |
| `mesh_partition` | WiFi mesh partitioned | LoRa backup maintains connectivity |
| `bridge_failover` | Domain bridge fails mid-session | Traffic reroutes via alternate bridge |

### 8.3 Scale Scenarios

| Scenario | Description | Pass Criteria |
|---|---|---|
| `scale_100_nodes` | 100 nodes, 50 clients | All metrics within bounds |
| `scale_1000_nodes` | 1,000 nodes, 200 clients | Path construction <2s, success >99% |
| `scale_10000_nodes` | 10,000 nodes, 1,000 clients | Path construction <5s |
| `scale_mesh_100` | 100-node mesh island | Delivery within mesh, all roles |
| `scale_multi_pool` | 10 pools, 100 clients | Multi-pool paths at scale |

### 8.4 Cross-Domain Scenarios

| Scenario | Description | Pass Criteria |
|---|---|---|
| `clearnet_to_mesh_basic` | Clearnet client reaches mesh service | Delivery via domain bridge; success ≥ 95% |
| `mesh_to_clearnet_exit` | Mesh client reaches clearnet via gateway | Exit policy applied; success ≥ 95% |
| `mesh_to_mesh_via_overlay` | Two islands via clearnet overlay | Cross-domain delivery ≥ 90% |
| `mesh_to_mesh_direct_bridge` | Two islands via direct LoRa bridge | Direct delivery ≥ 90% |
| `clearnet_to_mesh_to_clearnet` | Three-domain path | All transitions correct; success ≥ 90% |
| `n_hop_unlimited_10` | 10-hop cross-domain path | Constructs and delivers; 0 rejections |
| `n_hop_unlimited_20` | 20-hop cross-domain path | Constructs and delivers; 0 rejections |
| `no_bgpx_router_mesh_access` | Clearnet daemon, no router → mesh | Reaches mesh service via bridge |
| `island_gateway_failover` | Primary bridge fails; secondary active | Reroute within 5s |
| `island_offline_reconnect` | Island loses all bridges; reconnects | Offline during isolation; resumes after |
| `multi_island_mesh` | 3 islands + clearnet overlay | All reachable from each other |
| `store_and_forward_mesh` | Island temporarily offline | Messages delivered after reconnection |
| `unified_dht_discovery` | Mesh island discovers clearnet nodes | Unified DHT serves all domains |
| `any_to_any_routing` | All required domain→domain combinations | 100% reachability where bridge exists |

### 8.5 Anonymity Scenarios

| Scenario | Description | Pass Criteria |
|---|---|---|
| `timing_correlation` | Global passive adversary | Correlation success < 5% with 4-hop paths |
| `sybil_attack` | 10% Sybil nodes | Diversity constraints prevent single operator control |
| `bridge_sybil` | Adversary controls 80% of bridges to specific island | Path diversity limits compromise to < 20% |
| `domain_isolation` | Adversary controls all bridges to target island | Island unreachable (expected) |
| `mesh_radio_interception` | Adversary records all LoRa traffic | Content encrypted; no plaintext |

---

## 9. CLI Usage

### 9.1 Basic Commands

```bash
# Run a single scenario
bgpx-sim run --scenario scenarios/basic_4hop.toml

# Run all standard scenarios
bgpx-sim run-suite --suite standard

# Run with results output
bgpx-sim run \
    --scenario scenarios/scale_10000.toml \
    --output results/scale_$(date +%Y%m%d).json \
    --verbose

# Run in fast simulated time (100× real time)
bgpx-sim run \
    --scenario scenarios/anonymity_analysis.toml \
    --time-mode simulated \
    --time-scale 100
```

### 9.2 Cross-Domain Commands

```bash
# Run cross-domain scenario
bgpx-sim run --scenario scenarios/cross_domain/clearnet_to_mesh.toml

# Run N-hop unlimited verification suite
bgpx-sim run-suite --suite n_hop_unlimited

# Run any-to-any routing suite
bgpx-sim run-suite --suite any_to_any_routing

# Run full cross-domain suite
bgpx-sim run-suite --suite cross_domain
```

### 9.3 Topology and Analysis Commands

```bash
# Generate cross-domain topology report
bgpx-sim topology --config scenarios/multi_island.toml --output topology_report.html

# Run attack simulation against cross-domain setup
bgpx-sim attack --scenario scenarios/attacks/bridge_sybil.toml

# Generate mesh island visualization
bgpx-sim visualize-mesh --island-id lima-district-1 --output mesh_vis.png

# Check unified DHT coverage
bgpx-sim dht-coverage --scenario scenarios/multi_island.toml
```

### 9.4 Suite Commands

```bash
# Run all correctness scenarios
bgpx-sim run-suite --suite correctness

# Run all resilience scenarios
bgpx-sim run-suite --suite resilience

# Run all scale scenarios
bgpx-sim run-suite --suite scale

# Run all anonymity scenarios
bgpx-sim run-suite --suite anonymity

# Run full test suite (all scenarios)
bgpx-sim run-suite --suite all

# Run with specific random seed (for reproducibility)
bgpx-sim run --scenario scenarios/basic_4hop.toml --seed 42
```

---

## 10. Implementation Requirements

### 10.1 MUST

- Use real bgpx-node daemon code (no mocking of routing logic)
- Support both wall-clock and simulated time modes
- Implement all topology types
- Support domain-aware topology construction
- Verify all assertions
- Collect all defined measurements
- Support N-hop unlimited paths (no maximum enforcement)
- Support cross-domain paths with domain bridge nodes
- Simulate LoRa duty cycle constraints accurately
- Simulate satellite latency classes correctly
- Validate cover traffic indistinguishability
- Test return path routing via path_id

### 10.2 MUST NOT

- Mock the BGP-X routing logic (use real code)
- Enforce maximum hop count at any layer
- Ignore cross-domain path construction failures
- Skip diversity enforcement in any scenario
- Accept unsigned or invalid advertisements in simulation DHT

### 10.3 SHOULD

- Support loading custom scenarios from files
- Provide visualization output for topologies
- Support incremental scenario development (add nodes at runtime)
- Support mesh island simulation with realistic radio constraints
- Support pluggable transport simulation with DPI filtering
- Provide detailed failure diagnostics in results

---

## 11. Extension Points

| Extension | Description | Version |
|---|---|
| Cover traffic simulation | COVER packet generation and statistical indistinguishability testing | v1 |
| Pluggable transport simulation | DPI filtering and obfs4-style obfuscation | v1 |
| Satellite link simulation | LEO/GEO latency classes | v1 |
| Cross-domain routing | Domain bridge nodes and multi-domain paths | v1 |
| Mesh island simulation | WiFi mesh and LoRa with duty cycle | v1 |
| Anonymity metrics | Timing correlation analysis | v1 |
| Pool simulation | Pool membership, key rotation, curator failure | v1 |
| Attack scenarios | Sybil, MITM, passive observation | v1 |
| N-hop unlimited verification | 10, 15, 20 hop paths | v1 |
| Any-to-any routing | All domain combinations | v1 |
| Unified DHT coverage | Discovery from all domains | v1 |
| Store-and-forward | Offline island buffering | v1 |

---

## 12. Test Vectors

Simulation-specific test vectors are defined in `/protocol/test_vectors/simulation/`:

- Basic 3-hop path construction
- Basic 4-hop path construction
- Multi-pool path construction
- Cross-domain path construction (clearnet→mesh)
- Cross-domain path construction (mesh→clearnet)
- Domain bridge transition
- Return path routing via path_id
- COVER packet indistinguishability
- Stream multiplexing correctness
- LoRa fragmentation and reassembly
- Mesh island gateway failover
- Pool curator key rotation during active paths
- N-hop unlimited (10, 15, 20 hops)
- Any-to-any routing coverage

---

**End of Network Simulation Specification**
