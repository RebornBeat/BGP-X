# BGP-X Routing Algorithm Specification

**Version**: 0.1.0-draft

This document specifies the complete routing algorithm used by BGP-X clients to select paths, score nodes, and make routing decisions across single and multiple routing domains.

---

## 1. Overview

BGP-X routing is **client-driven inter-protocol domain source routing**. The client constructs the complete path—including cross-domain transitions—before transmitting any data. No relay node participates in path selection decisions.

The routing algorithm operates in four phases:

1. **Node database query**: retrieve candidate nodes, respecting pool configuration and domain requirements
2. **Filtering**: apply mandatory constraints (signature validity, blacklist, role, domain membership)
3. **Scoring**: rate nodes using a weighted multi-component function (domain-aware)
4. **Selection**: weighted random sampling with cross-domain diversity enforcement

**N-hop unlimited**: BGP-X imposes **no protocol-level maximum on path hop count** at any level: total path hops, pool segment count, domain traversal count, domain bridge transition count, or mesh relay hops. The protocol enforces no maximum. Practical defaults exist for latency, but these are configuration, not protocol limits.

---

## 2. Node Database

The routing algorithm operates on a local node database maintained by the client daemon.

### 2.1 Database Contents

```rust
struct NodeDatabaseEntry {
    node_id:                NodeId,                   // 32 bytes
    public_key:             Ed25519PublicKey,         // 32 bytes
    roles:                  Vec<NodeRole>,            // {entry, relay, exit, discovery, bridge}

    // Routing Domains & Bridges (Cross-Domain)
    routing_domains:        Vec<RoutingDomainEntry>,  // Endpoints in specific routing domains
    bridge_capable:         bool,                     // Can bridge between domains
    bridges:                Vec<DomainBridge>,        // Specific bridge pairs supported
    serves_domains:         Vec<DomainId>,            // Domain IDs this node serves

    // Network & Geo
    asn:                    u32,
    country:                String,                   // ISO 3166-1 alpha-2
    region:                 Region,                   // Geographic region enum
    operator_id:            Option<Ed25519PublicKey>, // Optional operator identity

    // Exit Policy
    exit_policy:            Option<ExitPolicy>,
    exit_policy_version:    Option<u16>,

    // Performance Metrics
    bandwidth_mbps:         f32,
    latency_ms:             f32,
    uptime_pct:             f32,

    // Reputation & Plausibility
    reputation_score:       f32,                      // 0.0-100.0; from reputation system
    geo_plausibility_score: f32,                      // 0.0-1.0; RTT-based verification

    // Lifecycle
    last_seen:              Timestamp,
    advertisement_expires:  Timestamp,
    withdrawn:              bool,                     // True if node has withdrawn
    blacklisted:            bool,
    blacklist_reason:      Option<String>,

    // Pool & Mesh Membership
    pool_memberships:       Vec<PoolId>,
    island_memberships:     Vec<String>,              // Mesh islands this node belongs to

    // Capabilities
    pt_supported:           Vec<String>,              // Pluggable transports supported
    ech_capable:            bool,                     // Supports ECH
    is_gateway:             bool,                     // Bridges mesh to clearnet
    link_quality_profiles:  HashMap<TransportType, LinkQualityProfile>,
    transport_types:        Vec<TransportType>,

    extensions:             ExtensionFlags,
}

struct RoutingDomainEntry {
    domain_type:            String,                   // "clearnet", "mesh", "lora-regional", etc.
    domain_id:              DomainId,                 // 8-byte domain identifier
    endpoints:              Vec<Endpoint>,            // Transport-specific addresses
    mesh_transport:         Option<MeshTransport>,    // For mesh domains
}

struct DomainBridge {
    from_domain:            DomainId,
    to_domain:              DomainId,
    bridge_latency_ms:      f32,
    bridge_transport:       TransportType,
    available:              bool,
}
```

### 2.2 Database Maintenance

- Nodes are added as advertisements are retrieved from the Unified DHT.
- Nodes are removed when their advertisement expires AND they have not been seen within 30 days.
- **Withdrawn nodes**: immediately removed from the eligible pool; retained 7 days for propagation.
- **Blacklisted nodes**: retained 30 days (for blacklist propagation), then removed.
- Database persisted to disk (encrypted); restored on daemon restart.

### 2.3 Database Size Limits

Maximum: **10,000 nodes**

Eviction when full: least recently seen nodes removed first, ensuring at least 5 nodes per region are retained if available.

---

## 3. Filtering Stage

Before scoring, the routing algorithm filters nodes to those eligible for use in the current path context.

### 3.1 Mandatory Filters (Always Applied)

| Filter | Condition |
|---|---|
| Not blacklisted | `node.blacklisted == false` |
| Not withdrawn | `node.withdrawn == false` |
| Advertisement valid | `current_time < node.advertisement_expires` |
| Signature valid | Verified at database insertion time |
| Role appropriate | Node has required role for this path position (entry, relay, exit, bridge) |
| Minimum uptime | `node.uptime_pct >= 85.0` |
| Valid endpoints | Has reachable endpoint in required domain |
| Not already in path | Not already selected for this path construction |

### 3.2 Version Compatibility Filter

```
node.protocol_versions_min <= session_version <= node.protocol_versions_max
```

Only nodes supporting the client's selected protocol version are eligible.

### 3.3 Pool Membership Filter

When a segment requires a specific pool:

1. `pool_id` must be in `node.pool_memberships` (advisory check).
2. Individual pool membership signature must be verified (mandatory).

### 3.4 ECH Capability Filter

When `require_ech = true` in stream/segment config:

```
# Only exit nodes need ECH capability
if is_exit_segment:
    require node.ech_capable == true
```

### 3.5 Exit Policy Filter (for exit/gateway nodes)

For exit segment selection:

- `"exit"` in `node.roles`
- `node.exit_policy` is not null
- `exit_policy.allow_protocols` contains required protocol
- Destination port in `exit_policy.allow_ports` (or `[0]` = all)
- Destination not in `exit_policy.deny_destinations`
- `exit_policy.logging_policy` satisfies client requirement (if specified)
- `exit_policy.jurisdiction` satisfies client jurisdiction constraints (if specified)
- `exit_policy.ech_capable` = true if stream requires ECH

### 3.6 Mesh Transport Filter

For mesh segments:

- Node has matching transport in `mesh_endpoints`
- Transport type matches segment configuration (WiFi 802.11s, LoRa, BLE, Ethernet P2P)

### 3.7 Domain Membership Filter

For segments requiring nodes in a specific routing domain:

```python
candidates = [n for n in candidates if required_domain_id in n.serves_domains]
```

### 3.8 Bridge Capability Filter (Bridge Positions)

For DOMAIN_BRIDGE hops:

```python
candidates = [n for n in candidates
              if n.bridge_capable
              and has_bridge(n, from_domain, to_domain)
              and get_bridge(n, from_domain, to_domain).available]
```

### 3.9 Satellite Node Filter

Satellite-connected clearnet nodes (`latency_class = satellite-*`):

- Excluded from path positions requiring low latency (interactive traffic).
- Eligible for bulk transfer paths, DHT sync roles, and non-interactive relay positions.

**Clarification**: Commercial satellite internet (Starlink, Iridium, etc.) is **clearnet domain (0x00000001)**, not a separate domain. Satellite nodes are filtered/scored based on latency_class, not domain ID.

---

## 4. Scoring Function

After filtering, each candidate node is assigned a score for weighted random selection.

### 4.1 Score Components

```
score(node, domain) = 0.30 × S_uptime(node)
                    + 0.25 × S_latency(node, domain)
                    + 0.20 × S_bandwidth(node)
                    + 0.15 × S_reputation(node)
                    + 0.10 × S_geo_plausibility(node, domain)
```

Default weights sum to 1.0. Clients MAY override weights via SDK.

### 4.2 Component Functions

```python
def S_uptime(node):
    return node.uptime_pct / 100.0
    # Range: [0.0, 1.0]

def S_latency(node, domain):
    # Domain-aware latency scoring
    if domain.type == "clearnet":
        return max(0, 1.0 - (node.latency_ms / 500.0))
    elif domain.type == "mesh":
        if node.has_lora_transport_for_domain(domain):
            return max(0, 1.0 - (node.latency_ms / 5000.0))  # LoRa: wide range
        else:
            return max(0, 1.0 - (node.latency_ms / 100.0))   # WiFi mesh: tight
    elif domain.type == "satellite":  # Future; currently satellites are clearnet
        return 0.5  # Neutral — satellite latency expected
    return max(0, 1.0 - (node.latency_ms / 500.0))  # Default

def S_bandwidth(node):
    return min(1.0, node.bandwidth_mbps / 1000.0)
    # 1000+ Mbps = max score (diminishing returns above)

def S_reputation(node):
    return node.reputation_score / 100.0
    # reputation_score from reputation system (0.0-100.0)

def S_geo_plausibility(node, domain):
    # Geographic Plausibility is OPTIONAL
    if domain.type == "satellite":
        return 0.5  # Exempt
    if node.rtt_measurement_count < 5:
        return 0.5  # Insufficient measurements
    return node.geo_plausibility_score  # Pre-computed by reputation system
```

### 4.3 Transport-Adjusted Latency Scoring

| Transport | Latency Baseline | Score Reference |
|---|---|---|
| Internet UDP | 500ms max for 0 score | Standard |
| WiFi 802.11s | 100ms max for 0 score | Tighter (mesh should be fast) |
| LoRa | 5000ms max for 0 score | Wider (LoRa is inherently high-latency) |
| Satellite | Not scored by latency | Skip latency score; return 0.5 neutral |

### 4.4 Geographic Plausibility RTT Thresholds

**Note**: Geo plausibility scoring is OPTIONAL. It applies only if the node has declared a jurisdiction in its advertisement. If no jurisdiction is declared, `S_geo_plausibility` returns neutral (0.5).

Expected one-way RTT ranges per region pair:

| Region Pair | Minimum | Maximum Plausible |
|---|---|---|
| Same region | 0ms | 50ms |
| NA ↔ EU | 50ms | 120ms |
| NA ↔ AP | 100ms | 200ms |
| NA ↔ SA | 40ms | 100ms |
| EU ↔ AP | 100ms | 200ms |
| EU ↔ AF | 50ms | 150ms |
| EU ↔ ME | 40ms | 100ms |
| AP ↔ AF | 150ms | 300ms |
| All ↔ Satellite | 20ms | 600ms (GEO: up to 700ms) |

Mesh transport thresholds (by transport type):

| Mesh Transport | Expected RTT per Hop |
|---|---|
| WiFi 802.11s | 1-20ms |
| LoRa | 100ms-5000ms |
| Bluetooth BLE | 10-100ms |
| Ethernet P2P | <5ms |

### 4.5 Bridge Node Scoring (Specialized)

Bridge nodes require specialized scoring that accounts for uptime in both domains:

```python
def score_bridge_node(node, from_domain, to_domain):
    bridge = get_bridge_record(node, from_domain, to_domain)
    if not bridge or not bridge.available:
        return 0.0

    uptime_score = min(
        uptime_in_domain(node, from_domain),
        uptime_in_domain(node, to_domain)
    ) / 100.0

    latency_score = max(0, 1.0 - (bridge.bridge_latency_ms / 2000.0))
    rep_score = node.reputation_score / 100.0
    domain_diversity = min(1.0, len(node.routing_domains) / 5.0)
    geo_coverage = compute_geo_spread(node.routing_domains)

    return (
        0.30 * uptime_score +
        0.25 * latency_score +
        0.20 * rep_score +
        0.15 * domain_diversity +
        0.10 * geo_coverage
    )
```

---

## 5. Diversity Enforcement

### 5.1 Mandatory Constraints (Cross-Segment)

Applied to the **entire path** (all segments and bridge nodes combined):

```python
def check_cross_domain_diversity(all_path_nodes):
    # ASN diversity
    asns = [n.asn for n in all_path_nodes]
    if len(asns) != len(set(asns)):
        return False, "ASN not unique across path"

    # Country diversity
    countries = [n.country for n in all_path_nodes]
    if len(countries) != len(set(countries)):
        return False, "Country not unique across path"

    # Operator diversity (where operator_id is non-null)
    op_ids = [n.operator_id for n in all_path_nodes if n.operator_id]
    if len(op_ids) != len(set(op_ids)):
        return False, "Operator not unique across path"

    # Node uniqueness
    node_ids = [n.node_id for n in all_path_nodes]
    if len(node_ids) != len(set(node_ids)):
        return False, "Same node appears twice"

    return True, "OK"
```

### 5.2 Default Constraints (Applied Unless Overridden)

```python
# Minimum uptime
assert all(n.uptime_pct >= 85.0 for n in path), "Node below minimum uptime"

# Region diversity (at least 2 different regions)
regions = set(n.region for n in all_path_nodes)
assert len(regions) >= 2, "Insufficient region diversity"
```

### 5.3 Constraint Violation Handling

On constraint violation during selection:

1. Remove one of the conflicting nodes from consideration (randomly chosen between conflicting pair).
2. Retry selection for that position.
3. After 10 retries for a position: attempt with reduced path length (if > minimum).
4. If minimum path length cannot be satisfied: fail with `ERR_SEGMENT_CONSTRUCTION_FAILED`.

---

## 6. Path Length Selection

### 6.1 Defaults (Not Enforced Limits)

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain (2 segments + 1 bridge) | 7 |
| Cross-domain (3 segments + 2 bridges) | 10 |
| High-security application-specified | Any |

### 6.2 N-Hop Unlimited Policy

The protocol imposes **no global maximum**. Default configuration maximum: 15 hops (operator-settable). Soft warning is logged for paths exceeding 15 hops (high latency expected). Implementations MUST NOT enforce a maximum hop count at the protocol level.

### 6.3 Automatic Adjustment

| Condition | Adjustment |
|---|---|
| Network has < 50 nodes in database | Reduce to 3 hops (minimum) |
| Application requests high-security mode | Increase to 5-7 hops |
| Path latency exceeds application budget | Reduce by 1 hop (minimum 3) |
| Cover traffic enabled | No length adjustment needed |

### 6.4 Application-Specified Path Length

Applications using the BGP-X SDK can specify min and max:

```rust
PathConfig {
    min_hops: 3,
    max_hops: 7,
    preferred_hops: 4,
    // ...
}
```

---

## 7. Pool-Aware Path Construction

### 7.1 Segment Selection

For each segment, select nodes from the specified pool:

```python
function select_segment_nodes(pool_id, hops, constraints, used_nodes):

    candidates = get_pool_members(pool_id)  # Verified individually
    candidates = apply_mandatory_filters(candidates, used_nodes)

    if len(candidates) < hops:
        if path_config.fallback_to_default:
            candidates = get_default_pool_members()
            candidates = apply_mandatory_filters(candidates, used_nodes)
            if len(candidates) < hops:
                return FAIL(ERR_POOL_INSUFFICIENT_NODES)
        else:
            return FAIL(ERR_POOL_INSUFFICIENT_NODES)

    selected = []
    for i in range(hops):
        eligible = [n for n in candidates
                    if n not in selected
                    and n not in used_nodes
                    and diversity_ok(n, selected + list(used_nodes))]
        if not eligible:
            return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)
        node = weighted_random_select(eligible)
        selected.append(node)

    return selected
```

### 7.2 Multi-Segment Assembly (Single Domain)

```python
function build_multi_segment_path(segment_configs):

    all_selected = []
    for segment in segment_configs:
        segment_nodes = select_segment_nodes(
            pool_id = segment.pool,
            hops = segment.hops,
            constraints = segment.constraints,
            used_nodes = set(all_selected)
        )
        if segment_nodes is FAIL:
            return FAIL(segment_nodes.error)
        all_selected.extend(segment_nodes)

    # Final cross-path diversity check
    if not check_all_diversity_constraints(all_selected):
        return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)

    # Generate path_id
    path_id = generate_path_id()

    return (all_selected, path_id)
```

---

## 8. Full Cross-Domain Construction

For paths spanning multiple routing domains:

```python
def build_path(config):
    if config.domain_segments:
        return build_cross_domain_path(config.domain_segments, config.constraints)
    elif config.segments:
        return build_single_domain_path(config.segments, config.constraints)
    else:
        return build_single_domain_path(
            [{ pool = "bgpx-default", hops = 4, exit = true }],
            config.constraints
        )

def build_cross_domain_path(domain_segments, constraints):
    all_selected = []
    segment_results = []

    for i, seg in enumerate(domain_segments):
        if seg.type == "bridge":
            from_domain = get_previous_domain(domain_segments, i)
            to_domain = get_next_domain(domain_segments, i)

            bridge_candidates = discover_domain_bridges(from_domain, to_domain)
            if bridge_candidates is FAIL:
                if constraints.fallback_any_bridge:
                    bridge_candidates = discover_any_bridge_to(to_domain)
                if bridge_candidates is FAIL:
                    return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

            eligible = [n for n in bridge_candidates
                        if n not in all_selected and diversity_ok(n, all_selected)]
            if not eligible:
                return FAIL(ERR_NO_CROSS_DOMAIN_PATH)

            node = weighted_random_select(eligible, score_fn=score_bridge_node(from_domain, to_domain))
            all_selected.append(node)
            segment_results.append(('bridge', [node]))

        elif seg.type == "segment":
            domain = seg.domain
            pool = seg.pool or "bgpx-default"
            hops = seg.hops

            candidates = get_nodes_in_domain_and_pool(domain, pool)
            candidates = [n for n in candidates if verify_individually(n) and not is_blacklisted(n)]
            candidates = apply_mandatory_filters(candidates, all_selected)

            if len(candidates) < hops:
                if constraints.fallback_to_default and pool != "bgpx-default":
                    candidates = get_nodes_in_domain_and_pool(domain, "bgpx-default")
                    candidates = apply_mandatory_filters(candidates, all_selected)
                    if len(candidates) < hops:
                        return FAIL(ERR_POOL_INSUFFICIENT_NODES)
                else:
                    return FAIL(ERR_POOL_INSUFFICIENT_NODES)

            selected = []
            for _ in range(hops):
                remaining = [n for n in candidates if n not in selected and n not in all_selected]
                eligible = [n for n in remaining if diversity_ok(n, selected + all_selected)]
                if not eligible:
                    return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)
                node = weighted_random_select(eligible, score_fn=score_node_in_domain(domain))
                selected.append(node)
                all_selected.append(node)
            segment_results.append(('segment', selected))

    if not check_cross_domain_diversity(all_selected):
        return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)

    path_id = generate_path_id()
    return (all_selected, segment_results, path_id)
```

---

## 9. Weighted Random Selection

Nodes are NOT selected deterministically by highest score. Weighted random sampling is used:

```python
def weighted_random_select(candidates, score_fn):
    scores = [(n, score_fn(n)) for n in candidates]
    total = sum(s for _, s in scores)
    if total == 0:
        return random.choice(candidates)
    r = random.uniform(0, total)
    cumulative = 0
    for node, s in scores:
        cumulative += s
        if r <= cumulative:
            return node
    return scores[-1][0]
```

This ensures high-scoring nodes are proportionally more likely, but low-scoring eligible nodes have non-zero probability.

---

## 10. Multi-Path Routing

For high-throughput or high-reliability applications, BGP-X supports building multiple paths simultaneously.

### 10.1 Path Isolation

When building multiple paths, the routing algorithm enforces cross-path isolation:

- No node appears in more than one path simultaneously
- No operator appears in more than one path simultaneously
- No ASN appears in more than one path simultaneously

### 10.2 Load Distribution

Streams are distributed across available paths using round-robin assignment. Sensitive streams MAY be pinned to a specific path.

---

## 11. Path Caching and Rebuild

### 11.1 Default Cache TTL

Default: **10 minutes**

### 11.2 Cache Invalidation

Additional cross-domain cache invalidation triggers:

- Domain bridge node goes offline (KEEPALIVE timeout)
- Mesh island becomes unreachable (all bridges offline)
- `DOMAIN_ADVERTISE` record for required bridge pair expires
- `MESH_ISLAND_ADVERTISE` record expires
- Pool curator key rotation affecting any node in any domain segment

Bridge-level discovery cache: 5-minute TTL (shorter than path cache — bridge availability changes more frequently).

### 11.3 Rebuild Triggers (MUST rebuild)

- Any node in path goes offline (KEEPALIVE timeout)
- Any node in path blacklisted
- Path age exceeds TTL
- Pool curator key rotation (re-verify pool members)
- Any path segment fails (all nodes in segment unresponsive)

### 11.4 Rebuild Triggers (SHOULD rebuild)

- Path quality score below rebuild_quality_threshold (default 0.20)
- Measured RTT exceeds 3× baseline
- Packet loss exceeds 5% sustained over 30 seconds

### 11.5 Segment-Level Rebuild

When one segment fails, attempt segment-only rebuild before full path rebuild:

```
1. Identify failed segment
2. Select replacement nodes for that segment only
3. Establish new sessions with replacement segment nodes
4. Migrate in-flight streams to new segment
5. If segment rebuild fails: rebuild entire path
```

### 11.6 Cross-Domain Segment Rebuild

1. Identify failed segment or bridge transition
2. Attempt to rebuild only the failed part
3. If bridge transition fails: attempt alternative bridge node
4. If no alternative: rebuild affected portion
5. If full path fails: `FAIL(ERR_NO_CROSS_DOMAIN_PATH)`

### 11.7 Seamless Rebuild

Client maintains path pool (2-3 pre-built paths). When current path retires, replacement is immediately available:

- Pre-build starts when current path is assigned
- Different pool configurations pre-build separately
- Isolated paths (for sensitive streams) pre-built independently

---

## 12. Return Path Consideration

Return traffic uses `path_id` routing (relay nodes store `path_id → predecessor` mapping). The routing algorithm does not need to construct a separate return path — `path_id` enables bidirectional use of the same constructed path across all domain boundaries.

For long-lived bidirectional sessions, a separate return path MAY be constructed using different nodes. This reduces correlation between inbound and outbound timing patterns.

---

## 13. Metrics

| Metric | Description |
|---|---|
| `path_construction_attempts` | Total attempts |
| `path_construction_successes` | Successes |
| `path_construction_failures` | Failures (with reason breakdown) |
| `path_construction_latency_ms` | Time to construct |
| `cross_domain_constructions` | Paths using `domain_segments` |
| `domain_bridge_selections` | Bridge node selection attempts |
| `domain_bridge_failures` | Bridge node selection failures |
| `node_database_size` | Current entries |
| `node_database_by_region` | Count per region |
| `node_database_by_pool` | Count per pool |
| `bridge_nodes_in_database` | Count of bridge-capable nodes |
| `island_advertisements_cached` | Count of mesh island advertisement cache entries |
| `geo_plausibility_score_avg` | Average geo score across database |
| `diversity_constraint_violations` | Violations by type |
| `pool_fallbacks` | Count of pool insufficient → default fallback |

No node identifiers, path identifiers, or client data in metrics.
