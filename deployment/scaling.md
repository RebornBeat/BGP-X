# BGP-X Network Scaling Guide

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

---

## 1. Overview

This document covers scaling BGP-X deployments from single nodes to large global networks, including:

- Single node vertical scaling
- Multi-node operator clusters
- Geographic distribution strategy
- DHT scaling in the unified model
- Pool-based coordination and trust tiers
- Domain bridge scaling
- Mesh island scaling
- Gateway redundancy
- High-availability bootstrap nodes
- Monitoring and capacity planning at scale

BGP-X is designed to scale from a single development node to a global production network spanning clearnet, mesh islands, and satellite-connected regions — all operating within a single unified DHT and protocol.

---

## 2. Scale Tiers

### Tier 1: Development / Testnet (10–100 nodes)

**Characteristics**:
- Single region or two regions
- All nodes known to developers
- No real user traffic
- Focus: protocol correctness, basic performance

**DHT behavior**: Small DHT; routing table converges quickly; all advertisements fit comfortably in every node's DHT storage.

**Path construction**: Diversity constraints may need relaxation (fewer than 10 nodes per country makes country-diversity impossible).

**Cross-domain paths**: Limited testing; one or two domain bridge nodes at most.

**Operational approach**: All nodes manually managed by development team.

---

### Tier 2: Early Testnet (100–1,000 nodes)

**Characteristics**:
- 3–4 regions represented
- Mix of team-operated and community-operated nodes
- Limited real user traffic (opt-in beta users)
- Focus: DHT stability, reputation system calibration

**DHT behavior**: Lookup latency stabilizes; advertisement propagation reliable; Sybil resistance begins to matter.

**Path construction**: Most diversity constraints satisfiable for 3-hop paths; 4-hop paths may occasionally fail in underrepresented regions.

**Cross-domain paths**: Multiple domain bridges operational; clearnet-to-mesh and mesh-to-clearnet paths construct reliably.

**Scaling considerations**:
- Bootstrap nodes become critical; ensure high availability
- Reputation system needs calibration data from real node behavior
- Exit nodes need operator documentation and support
- Domain bridge nodes need monitoring for both domains they serve

---

### Tier 3: Public Testnet (1,000–10,000 nodes)

**Characteristics**:
- All 6 regions represented
- Community-operated nodes are majority
- Real user traffic
- Focus: performance under load, anonymity properties, operational stability

**DHT behavior**: Full Kademlia properties apply; lookup latency O(log N); storage load distributes evenly across storage nodes.

**Path construction**: All constraints satisfiable in most regions; 4-hop paths with full diversity construct reliably. Cross-domain paths with multiple domain transitions work correctly.

**Scaling considerations**:
- DHT storage nodes need adequate disk for record storage
- Reputation database size grows; pruning policies become important
- Cover traffic bandwidth costs become significant at scale
- Domain bridge nodes need capacity planning for cross-domain traffic
- Mesh island gateways need redundancy planning

---

### Tier 4: Production Mainnet (10,000+ nodes)

**Characteristics**:
- Global geographic coverage
- Mostly community-operated (decentralized)
- Full user traffic
- Focus: resilience, security, sustainability

**Scaling considerations at this tier**:
- DHT record count > 10,000; storage nodes need 100+ MB for DHT storage
- Reputation system distributed layer becomes important for quality
- BGP-X project shifts to governance role (protocol maintenance, security response) rather than operational role
- Multiple domain bridge nodes per domain pair required for resilience
- Mesh island gateways require geographic and operator diversity

---

## 3. Single Node Scaling

### 3.1 Vertical Scaling Limits

A single bgpx-node instance can handle:

| Metric | Limit | Notes |
|---|---|---|
| Concurrent sessions | ~10,000 (default) | Configurable higher with RAM |
| Relay throughput | ~1 Gbps on 8 cores | Linear with CPU cores |
| DHT queries/hour | ~50,000 | Query-heavy workloads |
| Path constructions/second | ~100 | Depends on DHT lookup latency |
| Cross-domain sessions | ~5,000 | Bridge nodes maintain two path_id tables |

For higher throughput: vertical scaling (more CPU, larger socket buffers) or horizontal scaling (add nodes).

### 3.2 Kernel Tuning for High-Throughput Nodes

```bash
cat >> /etc/sysctl.d/99-bgpx-highperf.conf << 'EOF'
# Increase socket buffer sizes
net.core.rmem_max = 268435456      # 256 MB
net.core.wmem_max = 268435456      # 256 MB
net.core.rmem_default = 16777216   # 16 MB
net.core.wmem_default = 16777216   # 16 MB

# Increase connection backlog
net.core.netdev_max_backlog = 131072
net.ipv4.tcp_max_syn_backlog = 65536

# Increase file descriptors
fs.file-max = 1048576

# Improve throughput
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
EOF

sysctl -p /etc/sysctl.d/99-bgpx-highperf.conf

# Increase file descriptors for bgpx user
echo "bgpx soft nofile 524288" >> /etc/security/limits.conf
echo "bgpx hard nofile 524288" >> /etc/security/limits.conf
```

### 3.3 Configuration for High-Traffic Relay

```toml
[sessions]
max_sessions = 50000

[network]
socket_recv_buffer = 67108864    # 64 MB
socket_send_buffer = 67108864    # 64 MB

[bandwidth]
total_bandwidth_mbps = 0         # Unlimited

[performance]
worker_threads = 8               # Match CPU cores
sender_threads = 4
receiver_threads = 8             # SO_REUSEPORT
```

### 3.4 Configuration for High-Traffic Domain Bridge Node

Domain bridge nodes maintain two path_id tables and process traffic in both domains. Higher resource requirements:

```toml
[sessions]
max_sessions = 30000             # Split between domains

[path_table]
single_domain_capacity = 15000   # Per-domain path_id entries
cross_domain_capacity = 15000    # Cross-domain path_id entries
# Total memory: ~2.5 MB per table = ~5 MB for both

[performance]
worker_threads = 8
# Bridge nodes benefit from more CPU for dual-path processing

[bridge]
health_check_interval = 30       # Seconds between bridge pair health checks
latency_report_interval = 60     # Seconds between bridge latency measurements
```

---

## 4. Operating Multiple Nodes (Operator Cluster)

An operator running multiple BGP-X nodes MUST:

1. Use the same `operator_id` in all nodes (operator's Ed25519 public key)
2. Use the same operator private key for signing exit policies
3. Expect that path construction will enforce operator diversity — no two nodes with the same operator_id in one path

### 4.1 Why Operator Diversity Matters

If you run 10 nodes and your `operator_id` is consistent, path construction ensures no two of your nodes appear in the same path. This preserves anonymity guarantees — a single operator cannot see both ends of a path.

If you run multiple nodes WITHOUT an `operator_id`, they appear as independent operators. This breaks path diversity enforcement and should be avoided.

### 4.2 Multi-Node Reputation Tracking

Reputation events for individual nodes are tracked independently. A misbehaving node from your cluster doesn't automatically affect others.

However, pool curators and the reputation system can track operator-level behavior: if many nodes from the same operator misbehave, the operator-level reputation can degrade, reducing selection probability for all your nodes.

---

## 5. Geographic Distribution Strategy

### 5.1 Minimum Viable Presence (Self-Sufficient Community)

For a BGP-X community to construct valid 4-hop paths internally (all-unique country and ASN constraints):

```
Minimum:
  4 nodes minimum, each in a different country and ASN

Recommended:
  8-12 nodes across 3+ regions, diverse ASNs
  Including 2-3 exit nodes in privacy-friendly jurisdictions
  At least 2 domain bridge nodes if operating a mesh island
```

### 5.2 Regional Exit Coverage

For good exit node coverage: at least one exit per region (NA, EU, AP). This ensures exit jurisdiction diversity across all client configurations.

| Region | Recommended Exit Count | Notes |
|---|---|---|
| EU | 3+ | Strong privacy laws; high demand |
| NA | 2+ | Avoid Five Eyes jurisdictions where possible |
| AP | 2+ | Singapore, Japan good jurisdictions |
| SA | 1+ | Emerging demand |
| AF | 1+ | Underserved; high-value for continent |
| ME | 1+ | Growing demand |

### 5.3 Region Bootstrap Sequence

Expand in this order for maximum path diversity benefit:

```
Phase 1: NA (North America) + EU (Europe)
         → Basic transatlantic diversity; 3-hop paths work reliably

Phase 2: Add AP (Asia-Pacific)
         → Tri-regional diversity; global paths possible

Phase 3: Add SA (South America) + AF (Africa)
         → Full global coverage; strong geographic diversity for 4-hop paths

Phase 4: Add ME (Middle East)
         → Complete 6-region coverage; maximum path diversity
```

### 5.4 Minimum Nodes per Region for Production Quality

| Region | Minimum Relay | Minimum Exit | Minimum Entry | Minimum Domain Bridge |
|---|---|---|---|---|
| NA | 50 | 5 | 20 | 3 |
| EU | 50 | 5 | 20 | 3 |
| AP | 30 | 3 | 15 | 2 |
| SA | 20 | 2 | 10 | 2 |
| AF | 15 | 2 | 8 | 2 |
| ME | 15 | 2 | 8 | 2 |

Domain bridge nodes enable cross-domain connectivity. For communities operating mesh islands, at least 2 domain bridge nodes per island are recommended for redundancy.

---

## 6. DHT Scaling in the Unified Model

### 6.1 Single Unified DHT Architecture

BGP-X operates **one unified Kademlia DHT** spanning all routing domains:

- All nodes regardless of routing domain participate in the same key space
- Domain bridge nodes with clearnet endpoints serve as the physical DHT storage infrastructure accessible from the internet
- Mesh-only nodes access the unified DHT via their bridge nodes' caches
- No separate "mesh DHT" — one DHT, accessible from all domains

This unified model simplifies scaling compared to systems with multiple DHTs that require synchronization.

### 6.2 Storage Node Requirements at Scale

| Network Size | Approx. DHT Records | Storage per Node | Lookup Latency |
|---|---|---|---|
| 1,000 nodes | 1,000 records | ~4 MB | ~100ms |
| 10,000 nodes | 10,000 records | ~40 MB | ~200ms |
| 100,000 nodes | 100,000 records | ~400 MB | ~350ms |
| 1,000,000 nodes | 1,000,000 records | ~4 GB | ~500ms |

Storage requirements scale linearly with network size. Disk is cheap; this is not a significant constraint.

### 6.3 DHT Query Volume at Scale

Each node issues approximately:
- 1 DHT lookup per 30 minutes (routing table refresh)
- 1 DHT_PUT per 12 hours (advertisement re-publication)
- 1 DHT_PUT per 8 hours (domain bridge advertisement, if bridge node)
- 1 DHT_PUT per 8 hours (mesh island advertisement, if gateway)
- N DHT lookups per path construction (where N = target path length × 2 for candidate nodes)

At 10,000 nodes, aggregate DHT query rate:
- Refresh queries: ~5.5 queries/second across the network
- Advertisement re-publications: ~0.2 PUTs/second
- Per-path-construction queries: proportional to client count × path construction rate

This is well within the capacity of a Kademlia DHT at this scale.

### 6.4 DHT Storage Node Selection

Storage nodes in the DHT are selected by key proximity. For the unified DHT:

- Any node may be a storage node for keys near its NodeID
- Bridge nodes serve as critical storage nodes for mesh-domain records (they cache for mesh-only nodes)
- No special designation required — Kademlia naturally distributes storage

For high-availability deployments: ensure nodes with stable, high-bandwidth connections are distributed across the key space.

---

## 7. Pool-Based Coordination

Multiple operators can coordinate trust through shared pools without centralizing operations.

### 7.1 Community Relay Pool Setup

1. One operator generates curator keypair and creates pool
2. All participating operators submit NodeIDs for inclusion
3. Curator signs each membership record
4. Pool published to unified DHT
5. All members add pool_id to their advertisements

```bash
# Curator: create pool
bgpx-cli pools create \
    --name "EU Trusted Relays" \
    --type curated \
    --curator-key /etc/bgpx/curator_private_key

# Curator: add members
bgpx-cli pools add-member \
    --pool-id hex... \
    --node-id hex... \
    --curator-key /etc/bgpx/curator_private_key

# Curator: publish
bgpx-cli pools publish --pool-id hex...

# Members: add pool to advertisement
# (in node.toml)
[advertisement]
pool_memberships = ["pool-id-hex"]
```

### 7.2 Pool Key Custodianship

For community pools with no single point of trust failure:

- Use hardware key with multiple authorized signers (e.g., 2-of-3 multisig with YubiKeys)
- Or: rotate curator key to a new operator each year (using key rotation mechanism)
- Or: accept single curator and plan for rotation if curator becomes unavailable

### 7.3 Domain-Scoped Pools

Pools may be scoped to specific routing domains:

```toml
# Pool configuration for mesh-only pool
[pool]
pool_id = "mesh-lima-relays"
pool_type = "curated"
domain_scope = "mesh:lima-district-1"

# Members must have endpoints in the specified domain
```

Domain-scoped pools enable:
- Mesh island operators to manage their own trust community
- Geographic-specific trust tiers
- Reduced DHT query scope for domain-specific path construction

### 7.4 Cross-Pool Paths

Paths may traverse multiple pools:

```
[Entry Pool: bgpx-default] → [Middle Pool: eu-trusted] → [Exit Pool: private-exit]
```

Each pool provides isolation. Cross-pool paths increase anonymity by ensuring no single pool curator can observe traffic crossing the entire path.

---

## 8. Domain Bridge Scaling

### 8.1 Bridge Node Capacity Planning

Domain bridge nodes handle traffic in both domains simultaneously and maintain two path_id tables.

| Metric | Clearnet-Only Node | Domain Bridge Node |
|---|---|---|
| Concurrent sessions | ~10,000 | ~5,000 per domain = ~10,000 total |
| Memory overhead | 50 MB base + 256 B × sessions | 50 MB base + 256 B × sessions × 2 (path tables) |
| CPU overhead | 1 core per 250 Mbps | 1 core per 200 Mbps (dual-domain processing) |

### 8.2 Minimum Bridge Node Deployment

For a mesh island of 50 nodes:

```
Minimum:
  2 domain bridge nodes (different operators, different locations if possible)
  Each with stable clearnet connection + mesh radio coverage

Recommended:
  3 domain bridge nodes
  At least 2 with satellite WAN backup (Starlink/Iridium)
  Geographic spread within the island for radio coverage redundancy
```

### 8.3 Bridge Pair Redundancy

```
                    ┌─── Bridge A (primary)
                    │    Location 1, Operator 1
Mesh Island ────────┤    Clearnet ISP A
                    │
                    └─── Bridge B (backup)
                         Location 2, Operator 2
                         Clearnet ISP B
```

Configuration: community members include BOTH bridges in their pool. Path selection distributes traffic across both bridges. If one fails, paths rebuild through the other.

```toml
[pools]
discover_pools = ["bgpx-default", "community-bridges"]

# community-bridges pool contains both Bridge A and Bridge B
```

### 8.4 Bridge Node Monitoring

```yaml
# Prometheus alerting rules for bridge nodes
- alert: BgpxBridgeDomainOffline
  expr: bgpx_bridge_domain_available == 0
  for: 5m
  severity: critical
  annotations:
    summary: "Bridge node lost connectivity to domain {{ $labels.domain_id }}"

- alert: BgpxBridgeLatencyHigh
  expr: bgpx_bridge_latency_ms > 500
  for: 15m
  severity: warning
  annotations:
    summary: "Bridge latency high: {{ $value }}ms"

- alert: BgpxBridgeCrossDomainTableNearCapacity
  expr: bgpx_bridge_cross_domain_table_usage_pct > 80
  for: 30m
  severity: warning
  annotations:
    summary: "Bridge cross-domain path table at {{ $value }}% capacity"
```

---

## 9. Mesh Island Scaling

### 9.1 Mesh Island Size Categories

| Size | Node Count | Coverage | Gateways |
|---|---|---|---|
| Small | 5-20 | Neighborhood, single building | 1-2 |
| Medium | 20-100 | Town, campus, industrial zone | 2-4 |
| Large | 100-500 | City district, region | 4-8 |
| Very Large | 500-2000 | Large city, multiple districts | 8-20 |
| Regional | 2000+ | Entire region, multiple cities | 20+ |

### 9.2 Mesh Transport Capacity

| Transport | Bandwidth | Range | Nodes per Gateway | Latency |
|---|---|---|---|---|
| WiFi 802.11s | 10-100 Mbps | 100m-1km (line of sight) | 20-50 | 5-50ms |
| LoRa SF7 | 5-50 Kbps | 2-10km | 50-200 | 500ms-2s |
| LoRa SF12 | 0.3-1 Kbps | 5-20km | 200-1000 | 2-10s |
| BLE | 100-500 Kbps | 10-100m | 5-20 | 10-100ms |

### 9.3 Gateway Scaling for Mesh Islands

For a medium-sized mesh island (50-100 nodes):

```
Minimum:
  2 gateways, different operators
  One primary ISP, one backup (satellite or secondary ISP)

Recommended:
  3-4 gateways
  At least 2 with satellite backup
  At least 1 in each major coverage area of the island
```

Gateway capacity formula:
```
gateway_count = ceil(peak_concurrent_users / 500)
             + ceil(total_mesh_nodes / 50)
             + redundancy_factor (minimum 1)

For 100 nodes, 200 concurrent users:
  ceil(200/500) + ceil(100/50) + 1 = 0 + 2 + 1 = 3 gateways minimum
```

### 9.4 Mesh Island DHT Participation

Mesh islands participate in the unified DHT via their gateway nodes:

- Gateway nodes cache DHT records for mesh-only nodes
- When island goes offline: records cached by gateway persist in unified DHT
- When island reconnects: gateway syncs cached records back to mesh nodes
- Multiple gateways provide DHT storage redundancy

---

## 10. Gateway Redundancy

### 10.1 Clearnet Gateway Redundancy

For mesh communities that depend on a gateway for clearnet access:

```
                    ┌─── Gateway A (primary)
                    │    Operator 1, ISP A
                    │    Domain bridge: clearnet ↔ mesh
                    │
Mesh Community ─────┤
                    │
                    └─── Gateway B (backup)
                         Operator 2, ISP B (or satellite)
                         Domain bridge: clearnet ↔ mesh
```

Configuration: community members include BOTH gateways in their pool. Path selection will distribute traffic across both gateways. If one fails, paths rebuild through the other.

```toml
[pools]
discover_pools = ["bgpx-default", "community-exits"]

# community-exits pool contains both Gateway A and Gateway B
```

### 10.2 Gateway Failover Time

When a gateway fails:

1. Active paths through gateway A timeout after 90 seconds (session dead timeout)
2. Path rebuild triggered automatically
3. New paths constructed through Gateway B
4. Typical failover time: 10-30 seconds for timeout detection + 2-5 seconds for path rebuild

For faster failover: implement active gateway health monitoring and pre-build backup paths.

### 10.3 Gateway Failover with Pre-Built Paths

```toml
[paths]
prebuild_count = 2                # Pre-build 2 paths
pool_preference = ["primary-gateway", "backup-gateway"]  # Prefer primary, keep backup ready

[gateway]
health_check_interval = 10        # Check gateway health every 10 seconds
failover_trigger_threshold = 2     # Failover after 2 failed health checks
```

With pre-built backup paths, failover time reduces to ~5-10 seconds.

---

## 11. High-Availability Bootstrap Nodes

### 11.1 Bootstrap Node Role

Bootstrap nodes provide initial network entry for new nodes. They serve DHT records only — they do not relay traffic or participate in paths.

For a self-sufficient community, operate internal bootstrap nodes:

```toml
[node]
roles = ["discovery"]              # Bootstrap-only: no relay/entry/exit

[dht]
bootstrap_nodes = []               # Bootstrap nodes don't use other bootstraps
```

Distribute these internally (e.g., embedded in community-specific software release with updated hardcoded bootstrap list). This enables the community to operate even if official BGP-X bootstrap nodes are unreachable.

### 11.2 Minimum Bootstrap Node Count

| Network Size | Minimum Bootstrap Nodes | Geographic Spread |
|---|---|---|
| Development (<100) | 2 | Single region acceptable |
| Testnet (100-10,000) | 5 | 3+ regions |
| Production (10,000+) | 10+ | All 6 regions |
| Self-sufficient community | 2+ | Within community or trusted external |

### 11.3 Bootstrap Node Sizing

Bootstrap nodes handle only DHT queries — no relay traffic:

```toml
[performance]
worker_threads = 2                # Minimal CPU needed
max_sessions = 0                   # No sessions (discovery only)

[dht]
storage_capacity = 100000         # Store up to 100k records
```

### 11.4 DNS-Based Bootstrap Fallback

For communities without internal bootstrap nodes:

```
_bgpx-bootstrap._udp.bgpx.network. TXT "node_id_hex,port"
_bgpx-bootstrap._udp.bgpx.network. TXT "node_id_hex2,port2"
```

Nodes can bootstrap via DNS over HTTPS (DoH) if direct bootstrap connectivity fails.

---

## 12. Multi-Pool Architecture

A large operator or community may run parallel pools for different trust tiers:

### 12.1 Trust Tier Example

```
Pool: "community-default"   → General relay nodes (Tier 1)
Pool: "community-trusted"   → Vetted, high-uptime relays (Tier 2)
Pool: "community-exits"     → Verified exit operators (Tier 3)
Pool: "community-private"   → Core operators only (Tier 4)
Pool: "community-bridges"   → Domain bridge nodes (Tier 5)
```

### 12.2 Path Configurations by Trust Level

```
Standard:          [community-default: 3 hops] → [community-exits: 1 hop]
High-security:     [community-default: 2 hops] → [community-trusted: 1 hop] → [community-private: 1 hop]
Maximum security:  [community-trusted: 2 hops] → [community-private: 2 hops]
Cross-domain:      [community-default: 2 hops] → [community-bridges: 1 hop] → [community-default: 1 hop]
```

### 12.3 Pool Member Scaling

| Pool Size | Members | Curator Effort | Recommended |
|---|---|---|---|
| Small | 5-20 | Manual curation | Yes |
| Medium | 20-100 | Semi-automated verification | Yes |
| Large | 100-1000 | Automated verification required | Curated pool may not scale well |
| Very Large | 1000+ | Consider public pool | Curated impractical |

For large trusted pools: use semi-private pools where membership is by application with automated reputation and uptime verification.

---

## 13. Monitoring at Scale

### 13.1 Network Health Metrics (Aggregate)

The BGP-X project maintains public dashboards showing (no individual node or user data):

```
Global network metrics:
- Total node count (by region, by role, by domain type)
- Network uptime percentage (weighted by node count)
- Median path construction latency (sampled)
- Anonymity set size estimate (from DHT network size estimate)
- Bootstrap node availability
- Domain bridge pair health (up/down/latency)
- Mesh island connectivity status
```

### 13.2 Per-Operator Monitoring

Node operators are responsible for their own monitoring. The Control API and Prometheus exporter provide all necessary data.

### 13.3 Prometheus Federation

For multi-node deployments, use Prometheus federation:

```yaml
# prometheus-federation.yml
scrape_configs:
  - job_name: 'bgpx-nodes'
    static_configs:
      - targets:
          - 'node1.example.com:9474'
          - 'node2.example.com:9474'
          - 'node3.example.com:9474'

  - job_name: 'bgpx-bridges'
    static_configs:
      - targets:
          - 'bridge1.example.com:9474'
          - 'bridge2.example.com:9474'

  - job_name: 'bgpx-gateways'
    static_configs:
      - targets:
          - 'gateway1.example.com:9474'
          - 'gateway2.example.com:9474'
```

### 13.4 Alerting Rules for Multi-Node Operations

```yaml
groups:
  - name: bgpx-general
    rules:
      - alert: BgpxOperatorNetworkDegraded
        expr: avg(bgpx_sessions_active) by (operator_id) < 10
        for: 30m
        severity: warning
        annotations:
          summary: "Operator's BGP-X nodes have low activity"

      - alert: BgpxPoolMemberCountLow
        expr: bgpx_pool_members_active < 5
        for: 15m
        severity: warning
        annotations:
          summary: "Pool {{ $labels.pool_id }} has too few active members"

      - alert: BgpxGatewayDown
        expr: bgpx_gateway_bridge_active == 0
        for: 5m
        severity: critical
        annotations:
          summary: "Gateway bridge to clearnet offline"

  - name: bgpx-bridge
    rules:
      - alert: BgpxBridgeDomainOffline
        expr: bgpx_bridge_domain_available == 0
        for: 5m
        severity: critical
        annotations:
          summary: "Bridge node lost connectivity to domain {{ $labels.domain_id }}"

      - alert: BgpxBridgeLatencyHigh
        expr: bgpx_bridge_latency_ms > 500
        for: 15m
        severity: warning
        annotations:
          summary: "Bridge latency high: {{ $value }}ms"

      - alert: BgpxBridgeCrossDomainTableNearCapacity
        expr: bgpx_bridge_cross_domain_table_usage_pct > 80
        for: 30m
        severity: warning
        annotations:
          summary: "Bridge cross-domain path table at {{ $value }}% capacity"

  - name: bgpx-mesh
    rules:
      - alert: BgpxMeshIslandOffline
        expr: bgpx_mesh_island_nodes_reachable == 0
        for: 10m
        severity: critical
        annotations:
          summary: "Mesh island {{ $labels.island_id }} has no reachable nodes"

      - alert: BgpxMeshGatewayDown
        expr: count(bgpx_mesh_gateway_active == 1) by (island_id) == 0
        for: 10m
        severity: critical
        annotations:
          summary: "Mesh island {{ $labels.island_id }} has no active gateways"

  - name: bgpx-dht
    rules:
      - alert: BgpxDHTLookupLatencyHigh
        expr: histogram_quantile(0.95, rate(bgpx_dht_lookup_duration_seconds_bucket[5m])) > 1.0
        for: 30m
        severity: warning
        annotations:
          summary: "DHT lookup latency high: {{ $value }}s (p95)"

      - alert: BgpxDHTStorageNearCapacity
        expr: bgpx_dht_storage_usage_pct > 80
        for: 30m
        severity: warning
        annotations:
          summary: "DHT storage at {{ $value }}% capacity"
```

---

## 14. Protocol Upgrade Coordination at Scale

When a protocol upgrade (new MAJOR version) is required:

```
Month -6: Announce upcoming upgrade in CHANGELOG and documentation
Month -3: Release software supporting both old and new version
Month  0: New version is default; old version still supported
Month +3: Old version deprecated (clients warned)
Month +6: Old version support ends; nodes running old version
          are flagged in reputation system
```

This 12-month total window ensures the network upgrades smoothly without forcing simultaneous updates.

### 14.1 Cross-Domain Upgrade Considerations

For networks with mesh islands and domain bridges:

- Domain bridge nodes MUST upgrade before cross-domain paths work correctly with new protocol features
- Mesh islands may upgrade independently of the main network
- Bridge nodes advertise supported protocol versions; path construction selects compatible nodes

---

## 15. Capacity Planning

### 15.1 Per-Metric Formulas

| Metric | Formula |
|---|---|---|
| Sessions per node | max_sessions (default: 10,000) |
| Memory per relay node | 50 MB base + 256 bytes × session_count |
| Memory per bridge node | 50 MB base + 512 bytes × session_count (dual path tables) |
| CPU for forwarding | ~1 core per 250 Mbps throughput |
| CPU for bridge node | ~1 core per 200 Mbps throughput |
| Network: relay | (total_bandwidth / relay_node_count) per node |
| Network: exit | (total_clearnet_bandwidth / exit_count) per node |
| Network: bridge | (cross_domain_bandwidth / bridge_count) per node |
| DHT storage | ~2 KB per stored advertisement × stored_count |
| Cross-domain path table | ~128 bytes per entry × entry_count |

### 15.2 Scaling Decision Points

| Condition | Action |
|---|---|---|
| CPU >80% sustained | Add relay nodes or upgrade hardware |
| Memory >85% | Increase RAM or reduce max_sessions |
| Session table >90% | Add nodes or increase max_sessions |
| Single pool <5 active members | Add members or enable fallback_to_default |
| Gateway utilization >80% | Add second gateway |
| Bootstrap nodes <5 healthy | Deploy additional bootstrap nodes |
| Cross-domain path table >80% | Increase RAM or add bridge nodes |
| Mesh island gateway overloaded | Add additional gateways |
| DHT lookup latency >500ms | Add more storage nodes across key space |

### 15.3 Example Capacity Calculation

For a medium BGP-X deployment:

```
Target: 10,000 nodes, 3 regions, 1 mesh island (100 nodes)

Infrastructure:
  - 500 relay nodes (3 regions × ~167 each)
  - 30 exit nodes (3 regions × 10 each)
  - 20 entry nodes (3 regions × ~7 each)
  - 10 domain bridge nodes (3 bridges for mesh island + 7 other cross-domain)
  - 4 mesh island gateways (for 100-node island)
  - 10 bootstrap nodes (global)

Per-node capacity (relay):
  - Sessions: 10,000 default
  - Memory: 50 MB + 256 B × 10,000 = 52.5 MB
  - CPU: 4 cores for ~1 Gbps
  - Storage: 40 MB for DHT (at 10,000 nodes)

Per-node capacity (bridge):
  - Sessions: 5,000 per domain × 2 = 10,000 total
  - Memory: 50 MB + 512 B × 10,000 = 55 MB
  - CPU: 8 cores recommended
  - Storage: 40 MB for DHT + bridge records

Mesh island infrastructure:
  - 100 nodes (80 relay + 15 entry + 5 bridge)
  - 4 gateways (different locations, 2 with satellite backup)
  - 2 internal bootstrap nodes
```

---

## 16. Satellite-Connected Node Scaling

### 16.1 Satellite Node Characteristics

Satellite-connected nodes (Starlink, Iridium, Inmarsat, etc.) are clearnet domain nodes with high latency:

| Satellite Type | Latency | Bandwidth | Use Case |
|---|---|---|---|
| LEO (Starlink, Kuiper) | 20-60ms RTT | 50-500 Mbps | Standard relay/bridge role |
| MEO (O3b) | 150ms RTT | 10-100 Mbps | Relay, not time-sensitive |
| GEO (Inmarsat, HughesNet) | 600ms+ RTT | 1-50 Mbps | DHT-only, low-throughput relay |

### 16.2 Satellite Node Capacity Adjustments

```toml
# Configuration for LEO satellite node (Starlink)
[sessions]
max_sessions = 5000              # Reduced due to higher latency

[performance]
keepalive_interval = 60          # Extended from 25s default
session_dead_timeout = 180       # Extended from 90s default

[gateway]
latency_class = "satellite-leo"  # 20-60ms

# Configuration for GEO satellite node (Inmarsat)
[sessions]
max_sessions = 1000              # Severely reduced

[performance]
keepalive_interval = 120         # Extended significantly
session_dead_timeout = 360        # Extended significantly

[gateway]
latency_class = "satellite-geo"  # 600ms+
```

### 16.3 Satellite Gateway Placement

For regions without fiber connectivity:

```
Remote Region
     │
     │ LoRa mesh
     ▼
[Mesh Island: 20-50 nodes]
     │
     │ WiFi/LoRa to gateway
     ▼
[BGP-X Gateway Node with Starlink WAN]
     │
     │ Satellite uplink
     ▼
Global BGP-X Network
```

Satellite gateways bridge mesh islands to the global BGP-X network where no terrestrial ISP is available.

---

## 17. Summary

BGP-X scaling encompasses:

- **Vertical scaling** of individual nodes (CPU, RAM, sessions)
- **Horizontal scaling** through operator clusters and geographic distribution
- **DHT scaling** in the unified model (single DHT, all domains)
- **Pool-based coordination** for trust management without centralization
- **Domain bridge scaling** for cross-domain connectivity
- **Mesh island scaling** with appropriate gateway redundancy
- **Gateway redundancy** for clearnet access resilience
- **Bootstrap node redundancy** for network entry resilience
- **Monitoring** with Prometheus federation and domain-specific alerts
- **Capacity planning** with per-metric formulas

The unified DHT and domain-agnostic protocol simplify scaling compared to multi-DHT architectures. The primary scaling challenges are operational (ensuring geographic diversity, bridge redundancy, gateway failover) rather than architectural.

---

**End of Scaling Guide**
