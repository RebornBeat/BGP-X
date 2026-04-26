# BGP-X Network Simulation Specification

**Version**: 0.1.0-draft

---

## 1. Overview

`bgpx-sim` is the BGP-X network simulation tool. It runs actual bgpx-node daemon code against virtual network infrastructure to validate protocol behavior, measure anonymity properties, and test cross-domain path construction under controlled conditions.

---

## 2. Architecture

```
bgpx-sim runtime
├── Virtual Network Layer (multi-domain)
│   ├── Domain registry (which routing domains exist)
│   ├── Node registry (NodeID → SimNode with domain membership)
│   ├── Link registry (per domain, inter-domain via bridge nodes)
│   └── Cross-domain packet router
│
├── SimNode instances (real bgpx-node code, virtual transport)
│   ├── Domain-aware configuration
│   └── Domain bridge simulation (bridge-capable nodes)
│
├── SimMeshIsland
│   ├── Island topology (nodes, radio links)
│   └── Local mesh DHT
│
├── Scenario runner
│   ├── Domain topology specification
│   └── Cross-domain event simulation
│
├── Traffic generator (client simulation)
│
└── Analysis engine
    ├── Cross-domain path analysis
    ├── Domain reachability metrics
    └── Bridge node availability metrics
```

---

## 3. Virtual Network Layer

### Link Properties

```rust
pub struct LinkProperties {
    pub latency_ms: LatencyDistribution,
    pub packet_loss_rate: f64,
    pub bandwidth_mbps: f64,
    pub jitter: JitterModel,
    pub transport_type: TransportType,
    pub mtu: usize,
    pub dpi_filtered: bool,
    pub domain_pair: Option<(DomainId, DomainId)>,     // For cross-domain links
    pub bridge_transition_latency_ms: Option<f64>,     // Additional latency at bridge
}

pub enum LatencyDistribution {
    Fixed(f64),
    Normal { mean: f64, std_dev: f64 },
    Uniform { min: f64, max: f64 },
    LoRaModel { spreading_factor: u8, bandwidth_khz: u32 },
    SatelliteModel { orbit_type: OrbitType },
}
```

### Network Topologies

```rust
pub enum Topology {
    // Single-domain (existing)
    Internet { node_count: usize, avg_links: usize },
    Geographic { regions: Vec<Region>, nodes_per_region: usize },
    PartialMesh { entry_count: usize, relay_count: usize, exit_count: usize },

    // Mesh-only (existing)
    MeshIsland {
        transport: TransportType,
        node_count: usize,
        topology: MeshTopologyType,
    },

    // Cross-domain (new)
    CrossDomain {
        clearnet_nodes: usize,
        mesh_islands: Vec<MeshIslandConfig>,
        domain_bridge_count_per_pair: usize,
        inter_island_bridges: Vec<(String, String)>,
        island_bridge_nodes_per_island: usize,
    },

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
```

---

## 4. Scenario Definition

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

---

## 5. Standard Scenario Library

All previous scenarios retained. New cross-domain scenarios:

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

---

## 6. Measurements

All previous measurements retained. New:

| Measurement | Description |
|---|---|
| cross_domain_path_construction_latency_ms | Time to build cross-domain path |
| cross_domain_delivery_success_rate | End-to-end success for cross-domain paths |
| domain_bridge_transition_latency_ms | Time spent at domain bridge transitions |
| mesh_island_reachability_pct | Fraction of time islands reachable from clearnet |
| bridge_node_availability_pct | Fraction of time bridge pairs have active bridges |
| n_hop_unlimited_violations | Paths rejected for hop count (MUST be 0) |
| any_to_any_routing_coverage_pct | Domain→domain combinations successfully routed |
| unified_dht_coverage_pct | Nodes discoverable from all domains |
| island_failover_time_ms | Time to route via secondary bridge after primary fails |

---

## 7. CLI Usage

All previous commands retained. New:

```bash
# Run cross-domain scenario
bgpx-sim run --scenario scenarios/cross_domain/clearnet_to_mesh.toml

# Run N-hop unlimited verification
bgpx-sim run-suite --suite n_hop_unlimited
bgpx-sim run-suite --suite any_to_any_routing
bgpx-sim run-suite --suite cross_domain

# Generate cross-domain topology report
bgpx-sim topology --config scenarios/multi_island.toml --output topology_report.html

# Run attack simulation against cross-domain setup
bgpx-sim attack --scenario scenarios/attacks/bridge_sybil.toml
```
