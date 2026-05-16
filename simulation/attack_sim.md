# BGP-X Attack Simulation Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

---

## 1. Overview

Attack simulations are distinct from network simulations. Where network simulations test correctness and performance under benign conditions, attack simulations:

- Introduce adversarially controlled nodes into the simulated network
- Measure the adversary's ability to achieve their stated goals
- Validate that BGP-X mitigations reduce attack effectiveness to acceptable levels
- Quantify residual risks

Attack simulations produce statistical results — given N trials, what fraction of the time does the adversary succeed?

The simulation runs actual `bgpx-node` code (in-process or via virtualized network) with actual adversary nodes. No mocking of cryptographic or routing logic.

---

## 2. Adversary Configuration

### 2.1 Adversary Configuration Struct

```rust
pub struct AdversaryConfig {
    /// Adversary type classification
    pub adversary_type: AdversaryType,

    /// Fraction of total nodes controlled by the adversary (for Sybil/node control)
    pub node_control_fraction: f64,

    /// Which node roles the adversary controls
    pub controlled_roles: Vec<NodeRole>,

    /// Adversary capability level
    pub capability: AdversaryCapability,

    /// Adversary goal
    pub goal: AdversaryGoal,

    /// Whether the adversary coordinates across their nodes (vs. independent)
    pub coordinated: bool,

    /// Placement strategy for adversary nodes
    pub placement: NodePlacementStrategy,
}

pub enum AdversaryType {
    /// Internal adversary (controls nodes inside the network)
    Internal,

    /// Global passive adversary (observes all traffic, no node control)
    GlobalPassive,

    /// Network-level adversary (controls infrastructure, e.g., ISP)
    Infrastructure,
}

pub enum AdversaryCapability {
    /// Can only observe traffic (no modification)
    PassiveOnly,

    /// Can observe and drop packets
    ObserveAndDrop,

    /// Can observe, drop, and inject packets
    /// (injection fails cryptographically but can be attempted)
    FullActive,

    /// Can observe all network traffic globally
    GlobalPassive,

    /// Can perform deep packet inspection to identify protocol patterns
    DPIInspection,

    /// Can observe multiple routing domains simultaneously
    PassiveMultiDomain,
}

pub enum AdversaryGoal {
    /// Link a specific client to a specific destination
    SourceDestinationLink {
        target_client: ClientId,
        target_destination: String,
    },

    /// Deanonymize any client accessing a target destination
    Deanonymize {
        target_destination: String,
    },

    /// Degrade network availability
    DenialOfService {
        target_availability_reduction: f64,
    },

    /// Manipulate path selection to route more clients through adversary nodes
    PathBias,

    /// Isolate a mesh island from the broader network
    IslandIsolation {
        target_island: String,
    },

    /// Correlate traffic across routing domains
    CrossDomainCorrelation,
}
```

### 2.2 Node Placement Strategies

```rust
pub enum NodePlacementStrategy {
    /// Random placement (baseline)
    Random,

    /// Adversary tries to place nodes in positions that maximize path coverage
    StrategicEntryExit,

    /// Adversary concentrates in a single ASN (tests ASN diversity enforcement)
    SingleAsn(u32),

    /// Adversary concentrates in a single country (tests country diversity)
    SingleCountry(String),

    /// Adversary creates nodes with falsified ASN/country data
    Sybil {
        real_asn: u32,
        claimed_asns: Vec<u32>,
        real_country: String,
        claimed_countries: Vec<String>,
    },

    /// Adversary targets a specific pool for poisoning
    PoolPoisoning {
        target_pool_id: String,
    },

    /// Adversary creates Sybil nodes targeting domain bridge positions
    DomainBridgeSybil {
        from_domain: DomainId,
        to_domain: DomainId,
        bridge_control_fraction: f64,
    },

    /// Adversary creates a fake mesh island with rogue nodes
    FakeMeshIsland {
        island_id: String,
        node_count: usize,
    },

    /// Adversary attempts to isolate an existing mesh island
    IsolateIsland {
        island_id: String,
        method: IsolationMethod,
    },
}

pub enum IsolationMethod {
    /// Take all bridge nodes for the island offline
    TakeOfflineBridgeNodes,

    /// BGP hijack the bridge node IPs
    BgpHijackBridgeNodeIps,

    /// DDoS the bridge nodes
    DDosBridgeNodes,
}
```

---

## 3. Attack Scenarios

### 3.1 Passive Correlation (10% Node Control, No Cover Traffic)

**Objective**: An adversary controlling 10% of nodes (strategically placed at entry/exit) attempts to link clients to destinations by correlating traffic timing. Cover traffic is disabled.

```toml
[attack_scenario]
name = "passive_correlation_10pct"
description = "10% node control, passive timing correlation, no cover traffic"
trials = 1000

[network]
topology = "geographic"
total_nodes = 500
total_clients = 100

[adversary]
adversary_type = "internal"
node_control_fraction = 0.10
controlled_roles = ["relay", "entry", "exit"]
capability = "passive_only"
coordinated = true
placement = "strategic_entry_exit"
goal = "source_destination_link"

[clients]
cover_traffic_enabled = false

[measurements]
correlation_success_rate = true
false_positive_rate = true
mean_time_to_link_seconds = true

[assertions]
correlation_success_rate_max = 0.15  # At most 15% success with 10% node control
false_positive_rate_min = 0.30       # High false positive rate degrades usefulness
```

### 3.2 Passive Correlation (10% Node Control, With Cover Traffic)

**Objective**: Same as 3.1, but with cover traffic enabled at 100 Kbps.

```toml
[attack_scenario]
name = "passive_correlation_10pct_with_cover"
description = "10% node control, passive timing correlation, cover traffic enabled"
trials = 1000

[network]
topology = "geographic"
total_nodes = 500
total_clients = 100

[adversary]
adversary_type = "internal"
node_control_fraction = 0.10
controlled_roles = ["relay", "entry", "exit"]
capability = "passive_only"
coordinated = true
placement = "strategic_entry_exit"

[clients]
cover_traffic_enabled = true
cover_traffic_target_kbps = 100

[measurements]
correlation_success_rate = true
false_positive_rate = true

[assertions]
correlation_success_rate_max = 0.10  # Cover traffic should reduce success rate
false_positive_rate_min = 0.40       # Cover traffic increases false positives
```

### 3.3 Sybil Path Bias (20% Control)

**Objective**: An adversary creates many nodes with falsified diversity attributes (ASN/country) to increase their presence in constructed paths.

```toml
[attack_scenario]
name = "sybil_path_bias_20pct"
description = "Sybil nodes with falsified ASN/country data"
trials = 500

[adversary]
adversary_type = "internal"
node_control_fraction = 0.20
placement = "sybil"

[adversary.sybil]
real_asn = 99999
claimed_asns = [1001, 2002, 3003, 4004, 5005, 6006, 7007, 8008, 9009, 1010]
real_country = "XX"
claimed_countries = ["US", "DE", "JP", "BR", "AU", "CA", "FR", "SG", "ZA", "NL"]

[measurements]
adversary_nodes_in_paths_fraction = true
path_diversity_actual_vs_claimed = true

[assertions]
# Even with falsified attributes, adversary should not dominate paths
adversary_nodes_in_paths_fraction_max = 0.35
```

### 3.4 Sybil Path Bias (50% Control)

**Objective**: Stress test diversity enforcement with very high adversary control.

```toml
[attack_scenario]
name = "sybil_path_bias_50pct"
description = "Sybil nodes with 50% network control"
trials = 500

[adversary]
adversary_type = "internal"
node_control_fraction = 0.50
placement = "sybil"

[adversary.sybil]
real_asn = 99999
claimed_asns = [1001, 2002, 3003, 4004, 5005]
real_country = "XX"
claimed_countries = ["US", "DE", "JP", "BR", "AU"]

[measurements]
adversary_nodes_in_paths_fraction = true

[assertions]
adversary_nodes_in_paths_fraction_max = 0.70  # Should not reach 100%
```

### 3.5 Entry + Exit Simultaneous Control

**Objective**: Adversary attempts to control both entry and exit of specific paths by operating many high-reputation nodes.

```toml
[attack_scenario]
name = "entry_exit_control"
description = "Adversary controls entry + exit of same path"
trials = 1000

[adversary]
adversary_type = "internal"
node_control_fraction = 0.15
controlled_roles = ["entry", "exit"]
placement = "strategic_entry_exit"
capability = "passive_only"
coordinated = true

[measurements]
fraction_paths_with_adversary_entry_and_exit = true
successful_deanonymization_rate = true

[assertions]
fraction_paths_with_adversary_entry_and_exit_max = 0.05
```

### 3.6 Denial of Service — Path Exhaustion

**Objective**: Adversary attempts to make path construction fail by controlling nodes that go offline when selected.

```toml
[attack_scenario]
name = "dos_path_exhaustion"
description = "Adversary nodes go offline when selected for paths"
trials = 200

[adversary]
adversary_type = "internal"
node_control_fraction = 0.20
capability = "observe_and_drop"

[adversary.behavior]
# Go offline when selected as a path node
go_offline_when_selected = true
offline_duration_seconds = 300

[measurements]
path_construction_success_rate = true
path_construction_time_increase_factor = true
client_impact_duration_seconds = true

[assertions]
path_construction_success_rate_min = 0.85
# Even with 20% malicious nodes, clients should still succeed >85% of the time
```

### 3.7 Global Passive Adversary — No Cover Traffic

**Objective**: Simulate a global passive adversary with full network visibility and measure correlation success rates without cover traffic.

```toml
[attack_scenario]
name = "global_passive_no_cover"
description = "Global passive adversary, no cover traffic"
trials = 500

[adversary]
adversary_type = "global_passive"
capability = "global_passive"
# Global passive has no nodes; it observes all traffic

[clients]
cover_traffic_enabled = false

[measurements]
correlation_success_rate = true
```

### 3.8 Global Passive Adversary — With Cover Traffic

**Objective**: Measure correlation success rates with cover traffic enabled.

```toml
[attack_scenario]
name = "global_passive_with_cover"
description = "Global passive adversary, cover traffic enabled at 100 Kbps"
trials = 500

[adversary]
adversary_type = "global_passive"
capability = "global_passive"

[clients]
cover_traffic_enabled = true
cover_traffic_target_kbps = 100

[measurements]
correlation_success_rate = true

[assertions]
# Cover traffic must meaningfully reduce correlation success rate
correlation_success_rate_max = 0.40
```

### 3.9 Pool Poisoning Attack

**Objective**: Adversary curator signs a fake pool containing their malicious nodes, attempting to get clients to use adversary nodes for specific path segments.

```toml
[attack_scenario]
name = "pool_poisoning"
description = "Adversary curator signs malicious pool"
trials = 500

[adversary]
adversary_type = "internal"
node_control_fraction = 0.10
placement = "pool_poisoning"

[adversary.pool_poisoning]
target_pool_id = "trusted-relays"

[measurements]
adversary_nodes_selected_from_poisoned_pool = true
path_compromise_rate = true

[assertions]
adversary_nodes_selected_from_poisoned_pool_max = 0.20
# Individual node verification should limit this significantly
```

### 3.10 Pool Curator Key Compromise

**Objective**: Simulate a pool curator's key being stolen, followed by emergency key rotation.

```toml
[attack_scenario]
name = "pool_curator_key_compromise"
description = "Pool curator key stolen, emergency rotation"
trials = 200

[adversary]
adversary_type = "internal"
node_control_fraction = 0.10

# Events timeline
events = [
    { at = 60, type = "pool_curator_key_rotation", pool = "trusted-relays", reason = "compromise_confirmed" }
]

[measurements]
paths_using_old_key_members_after_rotation = true

[assertions]
paths_using_old_key_members_after_24h = 0.0
# After 24hr grace period, no old-key members should be selected
```

### 3.11 Routing Policy Bypass

**Objective**: Simulate a LAN device attempting to bypass the BGP-X routing policy in dual-stack mode.

```toml
[attack_scenario]
name = "routing_policy_bypass"
description = "LAN device attempts direct connection bypass"
trials = 100

[network]
mode = "dual_stack"

[device_behavior]
attempt_direct_connections = true

[measurements]
bypass_success_rate = true

[assertions]
# In BGP-X-only mode, this would be 0.0
# In dual-stack, bypass is a feature, but success should be logged
bypass_success_rate_max = 1.0  # Allowed in dual-stack, but measured
```

### 3.12 Pluggable Transport Bypass (DPI)

**Objective**: Adversary uses deep packet inspection to identify BGP-X traffic patterns and block them.

```toml
[attack_scenario]
name = "pt_bypass_dpi_no_pt"
description = "DPI identifies BGP-X traffic without PT"
trials = 500

[adversary]
capability = "dpi_inspection"

[clients]
pt_enabled = false

[measurements]
traffic_identified_as_bgpx = true

[assertions]
traffic_identified_as_bgpx_max = 0.90
```

### 3.13 Pluggable Transport Bypass (DPI with PT Enabled)

**Objective**: Measure DPI effectiveness when pluggable transport is enabled.

```toml
[attack_scenario]
name = "pt_bypass_dpi_with_pt"
description = "DPI attempts identification with PT enabled"
trials = 500

[adversary]
capability = "dpi_inspection"

[clients]
pt_enabled = true
pt_mode = "obfs4"

[measurements]
traffic_identified_as_bgpx = true

[assertions]
traffic_identified_as_bgpx_max = 0.20
```

### 3.14 Mesh Rogue Beacon

**Objective**: Adversary injects unsigned or invalid-signature mesh beacons to attempt to join the mesh island routing tables.

```toml
[attack_scenario]
name = "mesh_rogue_beacon"
description = "Inject unsigned/invalid signature beacons"
trials = 200

[adversary]
adversary_type = "internal"
inject_unsigned_beacons = true
inject_invalid_signature_beacons = true

[measurements]
rogue_nodes_in_routing_table = true

[assertions]
rogue_nodes_in_routing_table = 0
# No unsigned/invalid beacons should be accepted
```

### 3.15 Satellite Timing Correlation

**Objective**: Measure timing correlation success rate when traffic traverses satellite links (predictable latency).

```toml
[attack_scenario]
name = "satellite_timing_correlation"
description = "Timing correlation with satellite link"
trials = 500

[network]
topology = "geographic"
satellite_gateway_enabled = true

[adversary]
capability = "global_passive"
observes_satellite_link = true

[measurements]
correlation_success_rate = true

[assertions]
# Satellite's predictable latency makes timing correlation easier
# Baseline comparison vs. terrestrial-only timing correlation
correlation_success_rate_max = 0.60
```

### 3.16 Cross-Domain Timing Correlation (No Cover)

**Objective**: Adversary observes both clearnet and mesh radio traffic to correlate cross-domain paths without cover traffic.

```toml
[attack_scenario]
name = "cross_domain_timing_no_cover"
description = "Cross-domain correlation, no cover traffic"
trials = 500

[adversary]
capability = "passive_multi_domain"
observes_clearnet = true
observes_mesh_radio = ["lima-district-1"]

[clients]
cover_traffic_enabled = false
cross_domain_path = true

[measurements]
correlation_success_rate = true

[assertions]
correlation_success_rate_max = 0.30
```

### 3.17 Cross-Domain Timing Correlation (With Cover)

**Objective**: Cross-domain correlation with cover traffic enabled.

```toml
[attack_scenario]
name = "cross_domain_timing_with_cover"
description = "Cross-domain correlation, cover traffic enabled"
trials = 500

[adversary]
capability = "passive_multi_domain"
observes_clearnet = true
observes_mesh_radio = ["lima-district-1"]

[clients]
cover_traffic_enabled = true
cover_traffic_target_kbps = 100

[measurements]
correlation_success_rate = true

[assertions]
correlation_success_rate_max = 0.20
```

### 3.18 Domain Bridge Sybil (80% Control)

**Objective**: Adversary controls 80% of domain bridge nodes between clearnet and a mesh island, attempting to dominate cross-domain path selection.

```toml
[attack_scenario]
name = "domain_bridge_sybil_80pct"
description = "Adversary controls 80% of domain bridge nodes"
trials = 500

[adversary]
adversary_type = "internal"
placement = "domain_bridge_sybil"

[adversary.domain_bridge_sybil]
from_domain = "clearnet"
to_domain = "mesh:lima-district-1"
bridge_control_fraction = 0.80

[measurements]
adversary_bridge_selection_rate = true
domain_bridge_path_compromise_rate = true

[assertions]
adversary_bridge_selection_rate_max = 0.50
```

### 3.19 Mesh Island Isolation

**Objective**: Adversary takes all bridge nodes offline to isolate a mesh island from the clearnet.

```toml
[attack_scenario]
name = "mesh_island_isolation"
description = "Adversary isolates mesh island by taking bridges offline"
trials = 100

[adversary]
adversary_type = "internal"
placement = "isolate_island"

[adversary.isolate_island]
island_id = "lima-district-1"
method = "take_offline_bridge_nodes"

[measurements]
island_reachability_from_clearnet = true
intra_island_traffic_success_rate = true

[assertions]
island_reachability_from_clearnet_max = 0.05
intra_island_traffic_success_rate_min = 0.95
```

### 3.20 Fake Mesh Island

**Objective**: Adversary creates a fake mesh island and publishes it to the DHT, attempting to attract traffic.

```toml
[attack_scenario]
name = "fake_mesh_island"
description = "Adversary publishes fake mesh island"
trials = 500

[adversary]
adversary_type = "internal"
placement = "fake_mesh_island"

[adversary.fake_mesh_island]
island_id = "adversary-controlled-island"
node_count = 20
publish_to_dht = true

[measurements]
rogue_island_relay_selection_rate = true
traffic_routed_through_fake_island = true

[assertions]
rogue_island_relay_selection_rate_max = 0.30
```

---

## 4. Attack Simulation Metrics

### 4.1 Primary Metrics

| Metric | Description |
|---|---|---|
| `correlation_success_rate` | Fraction of trials where adversary correctly linked source to destination |
| `false_positive_rate` | Fraction of adversary's claimed links that are incorrect |
| `deanonymization_rate` | Fraction of clients whose identity was revealed |
| `adversary_path_presence` | Fraction of paths containing at least one adversary node |
| `adversary_entry_exit_control` | Fraction of paths where adversary controls both entry and exit |
| `path_construction_failure_rate` | Fraction of path constructions that failed due to adversary |
| `mean_time_to_link` | Average time for adversary to establish a source-destination link |
| `adversary_nodes_selected_from_poisoned_pool` | Fraction of adversary nodes selected from target pool |
| `rogue_nodes_in_routing_table` | Count of invalid beacon nodes accepted into DHT |
| `traffic_identified_as_bgpx` | Fraction of traffic correctly identified by DPI |
| `paths_using_old_key_members` | Fraction of paths using old-key pool members after rotation grace period |
| `adversary_bridge_selection_rate` | Fraction of cross-domain paths using adversary-controlled bridges |
| `domain_bridge_path_compromise_rate` | Fraction of cross-domain paths compromised by bridge control |
| `island_reachability_from_clearnet` | Reachability of island from clearnet during isolation attack |
| `intra_island_traffic_success_rate` | Success rate of traffic within island during isolation attack |
| `rogue_island_relay_selection_rate` | Fraction of paths routed through fake mesh island |
| `n_hop_violations` | Count of paths rejected due to hop count limit (should be 0) |
| `any_to_any_failures` | Count of domain pairs unreachable despite bridge existence |

---

## 5. Baseline Comparison

All attack scenario results are compared to a baseline (same scenario with `node_control_fraction = 0.0` or no adversary). This establishes what the random success rate would be by chance, allowing the adversary's actual advantage to be computed:

```
adversary_advantage = adversary_success_rate - baseline_success_rate
```

A well-designed anonymity system should produce minimal adversary advantage for realistic adversary configurations.

---

## 6. Acceptance Criteria

The following thresholds MUST be met for BGP-X to be considered ready for testnet deployment:

| Attack Scenario | Metric | Required Threshold |
|---|---|---|
| Passive correlation (10%, no cover) | Success rate | < 15% |
| Passive correlation (10%, no cover) | False positive rate | > 30% |
| Passive correlation (10%, with cover) | Success rate | < 10% |
| Sybil path bias (20% control) | Path presence | < 35% |
| Sybil path bias (50% control) | Path presence | < 70% |
| Entry + exit control (15% control) | Entry+exit same path | < 5% |
| DoS path exhaustion (20% control) | Path success rate | > 85% |
| Global passive (no cover) | Correlation success | < 60% |
| Global passive (with cover 100Kbps) | Correlation success | < 40% |
| Pool poisoning (10% control) | Poisoned pool node selection | < 20% |
| Pool curator key rotation | Old-key member selection after 24hr | = 0% |
| Routing policy bypass | Bypass success (BGP-X-only mode) | = 0% |
| PT bypass (no PT) | DPI identification | < 90% |
| PT bypass (with PT) | DPI identification | < 20% |
| Rogue beacon | Routing table contamination | = 0 |
| Satellite timing correlation | Correlation success | < 60% |
| Cross-domain timing (no cover) | Correlation success | < 30% |
| Cross-domain timing (cover 100Kbps) | Correlation success | < 20% |
| Domain bridge Sybil (80% control) | Adversary bridge selection | < 50% |
| Island isolation | Intra-island traffic success | > 95% |
| Island isolation | Reachability from clearnet | < 5% (correct: island unreachable) |
| Fake mesh island | Rogue relay selection rate | < 30% |
| N-hop unlimited | Paths rejected for hop count | = 0% |
| Any-to-any routing | Domain pair reachability (where bridge exists) | = 100% |
| Global passive (multi-domain, cover) | Source-destination link rate | < 15% |

These thresholds are initial targets. They will be refined based on academic analysis and empirical measurement on the testnet.

---

## 7. Reporting

Each scenario produces a JSON report:

```json
{
  "scenario": "domain_bridge_sybil_80pct",
  "trials": 500,
  "adversary_bridge_selection_rate": 0.43,
  "paths_fully_compromised": 0.11,
  "paths_partially_observed": 0.39,
  "paths_unobserved": 0.50,
  "correlation_success_rate": null,
  "false_positive_rate": null,
  "adversary_path_presence": null,
  "adversary_entry_exit_control": null,
  "path_construction_failure_rate": null,
  "adversary_nodes_selected_from_poisoned_pool": null,
  "rogue_nodes_in_routing_table": null,
  "traffic_identified_as_bgpx": null,
  "paths_using_old_key_members": null,
  "island_reachability_during_attack": null,
  "intra_island_traffic_success_rate": null,
  "rogue_island_relay_selection_rate": null,
  "n_hop_violations": 0,
  "any_to_any_failures": 0,
  "pass": true,
  "failures": []
}
```

Fields report `null` when not applicable to the specific scenario.

---

## 8. Test Vectors

Attack simulations rely on correct implementation of the adversary behavior. Test vectors for the correlation algorithm and signature verification are in `/protocol/test_vectors/README.md`.

---

**End of Attack Simulation Specification**
