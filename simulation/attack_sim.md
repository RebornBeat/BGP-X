# BGP-X Attack Simulation Specification

**Version**: 0.1.0-draft

---

## 1. Overview

Attack simulations measure adversary success rates under controlled conditions against known threat classes. The simulation runs actual bgpx-node code with actual adversary nodes.

---

## 2. Adversary Configuration

```rust
pub struct AdversaryConfig {
    pub adversary_type: AdversaryType,
    pub placement: NodePlacementStrategy,
    pub capabilities: Vec<AdversaryCapability>,
    pub coordination: CoordinationModel,
}

pub enum NodePlacementStrategy {
    Random { fraction: f64 },
    TargetEntry { fraction: f64 },
    TargetExit { fraction: f64 },
    TargetPool { pool_name: String, fraction: f64 },
    EntryAndExit { fraction: f64 },
    Sybil { count: usize, strategy: SybilStrategy },
    DomainBridgeSybil {
        from_domain: DomainId,
        to_domain: DomainId,
        bridge_control_fraction: f64,
    },
    FakeMeshIsland {
        island_id: String,
        node_count: usize,
    },
    IsolateIsland {
        island_id: String,
        method: IsolationMethod,
    },
}

pub enum IsolationMethod {
    TakeOfflineBridgeNodes,
    BgpHijackBridgeNodeIps,
    DDosBridgeNodes,
}
```

---

## 3. Attack Scenarios

### Existing Scenarios (All Retained)

`passive_correlation_no_cover`, `passive_correlation_with_cover`, `sybil_relay_20pct`, `sybil_relay_50pct`, `entry_exit_collude`, `dos_entry_nodes`, `global_passive_no_cover`, `global_passive_with_cover`, `pool_poisoning`, `curator_key_compromise`, `pool_rotation_attack`, `routing_policy_bypass`, `pluggable_transport_bypass`, `rogue_mesh_beacon`.

### New Cross-Domain Attack Scenarios

#### Cross-Domain Timing Correlation (No Cover)

```toml
[attack_scenario]
name = "cross_domain_timing_no_cover"
trials = 500
adversary.capability = "passive_multi_domain"
adversary.observes_clearnet = true
adversary.observes_mesh_radio = ["lima-district-1"]
clients.cover_traffic_enabled = false
clients.cross_domain_path = true
measurements = ["correlation_success_rate"]
assertions.correlation_success_rate_max = 0.30
```

#### Cross-Domain Timing Correlation (With Cover)

```toml
[attack_scenario]
name = "cross_domain_timing_with_cover"
adversary.capability = "passive_multi_domain"
adversary.observes_clearnet = true
adversary.observes_mesh_radio = ["lima-district-1"]
clients.cover_traffic_enabled = true
clients.cover_traffic_kbps = 100
measurements = ["correlation_success_rate"]
assertions.correlation_success_rate_max = 0.20
```

#### Domain Bridge Sybil (80% Control)

```toml
[attack_scenario]
name = "domain_bridge_sybil_80pct"
trials = 500
adversary.placement = "domain_bridge_sybil"
adversary.from_domain = "clearnet"
adversary.to_domain = "mesh:lima-district-1"
adversary.bridge_control_fraction = 0.80
measurements = [
    "adversary_bridge_selection_rate",
    "domain_bridge_path_compromise_rate"
]
assertions.adversary_bridge_selection_rate_max = 0.50
```

#### Mesh Island Isolation

```toml
[attack_scenario]
name = "mesh_island_isolation"
adversary.placement = "isolate_island"
adversary.island_id = "lima-district-1"
adversary.method = "take_offline_bridge_nodes"
measurements = [
    "island_reachability_from_clearnet",
    "intra_island_traffic_success_rate"
]
assertions.island_reachability_from_clearnet_max = 0.05
assertions.intra_island_traffic_success_rate_min = 0.95
```

#### Fake Mesh Island

```toml
[attack_scenario]
name = "fake_mesh_island"
adversary.placement = "fake_mesh_island"
adversary.island_id = "adversary-controlled-island"
adversary.node_count = 20
adversary.publish_to_dht = true
measurements = [
    "rogue_island_relay_selection_rate",
    "traffic_routed_through_fake_island"
]
assertions.rogue_island_relay_selection_rate_max = 0.30
```

---

## 4. Acceptance Criteria

All previous thresholds retained. New:

| Scenario | Metric | Pass Threshold |
|---|---|---|
| Cross-domain timing (no cover) | Correlation success rate | < 30% |
| Cross-domain timing (cover 100Kbps) | Correlation success rate | < 20% |
| Domain bridge Sybil (80% control) | Adversary bridge selection rate | < 50% |
| Island isolation | Intra-island traffic success | > 95% |
| Island isolation | Reachability from clearnet | < 5% (correct: island unreachable) |
| Fake mesh island | Rogue relay selection rate | < 30% |
| N-hop unlimited | Paths rejected for hop count | = 0% |
| Any-to-any routing | Domain pair reachability (where bridge exists) | = 100% |
| Global passive (multi-domain, cover) | Source-destination link rate | < 15% |

---

## 5. Reporting

All previous reporting fields retained. New cross-domain fields:

```json
{
  "scenario": "domain_bridge_sybil_80pct",
  "trials": 500,
  "adversary_bridge_selection_rate": 0.43,
  "paths_fully_compromised": 0.11,
  "paths_partially_observed": 0.39,
  "paths_unobserved": 0.50,
  "cross_domain_timing_correlation_rate": null,
  "island_reachability_during_attack": null,
  "n_hop_violations": 0,
  "any_to_any_failures": 0,
  "pass": true,
  "failures": []
}
```
