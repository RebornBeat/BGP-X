# BGP-X Path Construction and Routing Algorithm Specification

**Version**: 0.1.0-draft

---

## 1. Overview

Path construction is the process by which a BGP-X client selects an ordered sequence of relay nodes to form a path from the client to its destination. BGP-X uses **client-side inter-protocol domain source routing** — the sending party constructs the complete path, including cross-domain transitions, before transmitting any data.

No relay node participates in path selection decisions. The client:

1. Determines the domain sequence required for the path
2. Queries the unified DHT for candidate nodes per domain segment
3. Discovers domain bridge nodes for each required domain transition
4. Filters nodes using hard constraints
5. Scores remaining nodes using a weighted function (domain-aware)
6. Selects nodes using weighted random sampling with cross-domain diversity enforcement
7. Verifies the assembled path meets diversity requirements
8. Generates path_id (8 bytes, CSPRNG)
9. Constructs onion-encrypted packets for the selected cross-domain path

This is a fundamental design choice. Delegating path selection to the network (as BGP does) would allow network-level adversaries to influence routing. By keeping path selection at the client, BGP-X ensures that no single network entity can force all clients to route through a surveillance point.

The routing algorithm operates in four phases:

1. **Node database query**: retrieve candidate nodes, respecting pool configuration and domain requirements
2. **Filtering**: apply mandatory constraints (signature validity, blacklist, role, domain membership)
3. **Scoring**: rate nodes using a weighted multi-component function (domain-aware)
4. **Selection**: weighted random sampling with cross-domain diversity enforcement

---

## 2. N-Hop Unlimited Policy

BGP-X path construction imposes **no maximum on total path hops, domain count, domain traversal count, or segment count**.

The routing algorithm will construct a path of any length the client requests, provided:
- Sufficient eligible nodes exist in each domain segment
- Mandatory diversity constraints can be satisfied
- The 3-hop absolute minimum is met

The common header contains no hop counter field. No protocol mechanism decrements or enforces a hop maximum. Only the physical minimum (3 hops) and practical performance constraints apply.

Implementations MUST NOT enforce a maximum hop count at the protocol level.

Practical defaults (per-path configuration, not enforced limits):

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain, 2 segments + 1 bridge | 7 (3+1+3) |
| Cross-domain, 3 segments + 2 bridges | 10 (3+1+3+1+2) |
| High-security application-specified | Any |

The daemon logs a soft informational message when total hops exceeds 15 (high latency expected) but does NOT reject or limit the path.

---

## 3. Node Database

The routing algorithm operates on a local node database maintained by the client daemon.

### 3.1 Database Contents

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

### 3.2 Database Maintenance

- Nodes are added as advertisements are retrieved from the Unified DHT.
- Nodes are removed when their advertisement expires AND they have not been seen within 30 days.
- **Withdrawn nodes**: immediately removed from the eligible pool; retained 7 days for propagation.
- **Blacklisted nodes**: retained 30 days (for blacklist propagation), then removed.
- Database persisted to disk (encrypted); restored on daemon restart.

### 3.3 Database Size Limits

Maximum: **10,000 nodes**

Eviction when full: least recently seen nodes removed first, ensuring at least 5 nodes per region are retained if available.

---

## 4. DHT Pools

DHT Pools are a first-class path construction concept. Pools are named, signed collections of BGP-X nodes grouped by trust level or operational purpose.

### 4.1 Pool Types

| Type | Description | Discovery | Trust Level |
|---|---|---|---|
| Public | Open DHT; any node can claim membership | Public DHT lookup | Baseline |
| Curated | Curator signs membership; elevated signal | Public DHT lookup | Elevated (curator vouches) |
| Private | Membership not published; shared out-of-band | Out-of-band | Maximum (you know members) |
| Semi-private | Discoverable ID, restricted join | Public DHT lookup | Medium |
| Ephemeral | Session-scoped, temporary | Direct communication | Contextual |
| Domain-scoped | Restricted to nodes in a specific routing domain | Public DHT lookup | Domain-specific trust |

### 4.2 Default Pool

When no pool is specified, the path uses the "default" public pool — all nodes in the DHT that are not restricted to a specific pool.

### 4.3 Pool Advertisement Structure

```json
{
  "version": 1,
  "pool_id": "<BLAKE3 hash, hex, 32 bytes>",
  "pool_name": "My Trusted Relay Pool",
  "pool_type": "curated",
  "domain_scope": "any",
  "domains_covered": ["clearnet", "mesh:lima-district-1"],
  "curator_public_key": "<Ed25519 public key, base64url>",
  "member_list_format": "embedded",
  "members": ["node_id_1_hex", "node_id_2_hex", "..."],
  "member_count": 42,
  "description": "High-uptime relays operated by vetted community members",
  "min_reputation_for_selection": 70.0,
  "regions_covered": ["EU", "NA"],
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-05-01T12:00:00Z",
  "signature": "<Ed25519 signature of canonical JSON>"
}
```

**Fields for Cross-Domain Support:**

| Field | Values | Description |
|---|---|---|
| `domain_scope` | `"any"` (default), `"clearnet"`, `"mesh:<island_id>"`, `"bridge:clearnet→mesh:<island_id>"` | Restricts pool membership to a specific domain |
| `domains_covered` | Array of domain identifiers | Informational metadata about which domains this pool serves |

For large pools (>100 members), `member_list_format` = "external" and a `member_endpoint` URL is included. The external endpoint returns the member list on request.

### 4.4 Pool Discovery

Pool advertisements are stored in the **unified DHT** at:
```
pool_key = BLAKE3("bgpx-pool-v1" || pool_id_bytes)
```

Pool TTL: 7 days. Re-publication: every 3 days.

Pool discovery: standard DHT_GET with pool_key.

### 4.5 Pool Trust Model

**Trust is NOT transitive across pools.** Being in Pool A does not grant any trust from Pool B. Each node is independently verified regardless of pool membership.

Even if a pool curator is compromised, individual node verification (signature check on each node's own advertisement) provides defense in depth.

Pool membership is a reputation signal, not a bypass for cryptographic verification.

### 4.6 Pool Member Verification at Construction Time

Before including a node from a pool in a path, the client MUST:

1. Verify the node's pool membership signature (signed by pool curator)
2. Fetch the node's own advertisement from DHT
3. Verify the node's own advertisement signature (signed by node private key)
4. Verify the node's advertisement has not expired
5. Verify the node is not blacklisted locally

Pool membership alone is insufficient — the node's own advertisement must be separately valid.

### 4.7 Pool Node Churn

Pool membership does not guarantee node availability. Before construction:

- Individual node advertisements are checked for expiry
- Nodes with expired or invalid advertisements are excluded
- If a pool has insufficient active nodes (< pool minimum after filtering):
    - If `fallback_to_default = true`: fall back to default pool for this segment
    - If `fallback_to_default = false`: fail construction with ERR_POOL_INSUFFICIENT_NODES

---

## 5. Path Anatomy

A path consists of one or more **segments**, each within a single routing domain. Domain transitions between segments are handled by **domain bridge nodes**.

```
Single-domain path:
  [Segment 1: clearnet, 4 hops]

Cross-domain path (2 segments):
  [Segment 1: clearnet, 3 hops] → [DOMAIN_BRIDGE] → [Segment 2: mesh:island-1, 3 hops]

Cross-domain path (3 segments):
  [Segment 1: clearnet, 2 hops] → [BRIDGE 1] → [Segment 2: mesh:island-A, 2 hops] → [BRIDGE 2] → [Segment 3: clearnet, 2 hops]

Cross-domain path (any N):
  Any combination, any order, any number of times, no limit
```

A path consists of three categories of nodes within a segment:

```
[Entry Node] → [Relay Nodes × 0..N] → [Exit Node]
```

| Role | Count | Minimum | Description |
|---|---|---|---|
| Entry | 1 | 1 | First hop. Knows client IP. |
| Relay | 0..N | 0 | Middle hops. No knowledge of origin or destination. |
| Exit | 1 | 1 | Last hop for clearnet. Knows destination. |

**Minimum total path length**: **3 hops** across the full path regardless of domain count.

Default total path length: **4 hops** (entry + 2 relays + exit)

For BGP-X native service connections (no clearnet exit needed), the last node in the path is the service's entry node. The exit node role is replaced by a delivery node.

---

## 6. Domain Segment Configuration

Path segments are specified with domain, pool, and hop count:

```toml
# Single-domain (existing syntax — backward compatible)
segments = [
    { pool = "bgpx-default", hops = 3 },
    { pool = "my-private-exit", hops = 1, exit = true }
]

# Cross-domain (new syntax)
domain_segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 3 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:lima-district-1" },
    { type = "segment", domain = "mesh:lima-district-1", pool = "island-default", hops = 2 }
]

# Clearnet → mesh → clearnet
domain_segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 3 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "private-exits", hops = 1, exit = true }
]

# Mesh-to-mesh via clearnet overlay
domain_segments = [
    { type = "segment", domain = "mesh:island-A", hops = 2 },
    { type = "bridge", from_domain = "mesh:island-A", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 3 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:island-B" },
    { type = "segment", domain = "mesh:island-B", hops = 2 }
]
```

The `domain_bridge` field names the required transition. The routing algorithm discovers eligible bridge nodes for this transition from the unified DHT.

---

## 7. Anonymization Strategies Using Pools

The following strategies use DHT pools and path constraints to achieve specific privacy goals.

### 7.1 Strategy 1: Multi-Jurisdiction Routing

Route traffic through multiple legal jurisdictions to complicate legal coercion:

```toml
domain_segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "segment", domain = "clearnet", pool = "eu-no-log-exits", hops = 1, exit = true }
]
exit_jurisdiction_blacklist = ["US", "GB", "AU", "CA", "NZ"]  # Five Eyes
```

### 7.2 Strategy 2: Trust Tier Separation

Separate path segments by trust level:

```toml
domain_segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "segment", domain = "clearnet", pool = "curated-trusted", hops = 1 },
    { type = "segment", domain = "clearnet", pool = "my-private-exit", hops = 1, exit = true }
]
```

Untrusted entry nodes protect against entry-level observation. Trusted middle relays provide additional isolation. Self-operated exit provides maximum exit-trust.

### 7.3 Strategy 3: Pool Randomization (Traffic Diversity)

Client randomly selects from multiple curated pools per connection:

```toml
# Client randomly selects one of several curated pools per connection
# Prevents analysis of which specific pool nodes always appear in paths.
segment_pool_rotation = ["pool-a", "pool-b", "pool-c"]
```

Each new path selects from a different pool in the rotation. This prevents any single curator from observing all traffic patterns.

### 7.4 Strategy 4: Geographic Scatter

Force path to cross multiple distinct geographic regions:

```toml
domain_segments = [
    { type = "segment", domain = "clearnet", pool = "na-relay-pool", hops = 1 },
    { type = "segment", domain = "clearnet", pool = "eu-relay-pool", hops = 1 },
    { type = "segment", domain = "clearnet", pool = "ap-exit-pool", hops = 1, exit = true }
]
```

This forces traffic through North America → Europe → Asia-Pacific, making traffic correlation across jurisdictions more difficult.

### 7.5 Strategy 5: Double-Exit Architecture

Two independent exit segments in sequence:

```toml
segments = [
    { pool = "bgpx-default", hops = 2 },
    { pool = "exit-tier-1", hops = 1 },           # NOT the clearnet exit
    { pool = "exit-tier-2", hops = 1, exit = true }  # ACTUAL clearnet exit
]
```

- Exit Pool 1 nodes see: traffic going to Exit Pool 2 nodes (not destination)
- Exit Pool 2 nodes see: traffic from Exit Pool 1 nodes (not original client)
- No single exit operator sees both path origin AND destination

### 7.6 Strategy 6: Cross-Domain Double-Exit (Maximum Isolation)

```toml
domain_segments = [
    { type = "segment", domain = "clearnet", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 2 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", hops = 1, exit = true }
]
```

The intermediate mesh segment acts as an opacity layer between the two clearnet segments.

---

## 8. Explicit Exit Specification

Clients can specify exit selection by:

- **Pool**: all exit nodes from a named pool
- **NodeID**: specific node by hex ID
- **Selection mode**: random (default), round_robin, lowest_latency, specified

```rust
PathConfig {
    segments: vec![
        SegmentConfig {
            pool: "bgpx-default".into(),
            hops: 3,
            is_exit: false,
            exit_selection: ExitSelection::Random,
            ..Default::default()
        },
        SegmentConfig {
            pool: "my-private-exit".into(),
            hops: 1,
            is_exit: true,
            exit_selection: ExitSelection::Specified("node_id_hex".into()),
            constraints: SegmentConstraints {
                require_ech: true,
                exit_logging_policy: Some(LoggingPolicy::None),
                exit_jurisdiction_whitelist: Some(vec!["CH".into(), "IS".into()]),
                ..Default::default()
            },
        },
    ],
    ..Default::default()
}
```

### 8.1 Exit Constraints

| Constraint | Description |
|---|---|
| `require_ech` | Exit must support Encrypted Client Hello |
| `exit_logging_policy` | "none", "minimal", "detailed" — exit declares its policy |
| `exit_jurisdiction_whitelist` | Only exits in listed countries |
| `exit_jurisdiction_blacklist` | No exits in listed countries |
| `exit_provider_blacklist` | No exits operated by listed provider IDs |

---

## 9. Any-to-Any Connectivity

The path construction algorithm makes no assumption about which domain the client entered from. A clearnet client and a mesh client run the same algorithm. A path starting in any domain may include segments from any other domain.

All of the following domain sequences are valid:

```
clearnet → clearnet (standard single-domain)
clearnet → mesh
mesh → clearnet
mesh → mesh (via overlay or direct bridge)
clearnet → mesh → clearnet
mesh → clearnet → mesh
clearnet → mesh → mesh → clearnet
clearnet → satellite → clearnet
clearnet → mesh → satellite → clearnet
[any combination, unlimited]
```

The protocol enforces no ordering. The algorithm constructs whatever sequence the client specifies.

---

## 10. Filtering Stage

Before scoring, the routing algorithm filters nodes to those eligible for use in the current path context.

### 10.1 Mandatory Filters (Always Applied)

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

### 10.2 Version Compatibility Filter

```
node.protocol_versions_min <= session_version <= node.protocol_versions_max
```

Only nodes supporting the client's selected protocol version are eligible.

### 10.3 Pool Membership Filter

When a segment requires a specific pool:

1. `pool_id` must be in `node.pool_memberships` (advisory check).
2. Individual pool membership signature must be verified (mandatory).

### 10.4 ECH Capability Filter

When `require_ech = true` in stream/segment config:

```
# Only exit nodes need ECH capability
if is_exit_segment:
    require node.ech_capable == true
```

### 10.5 Exit Policy Filter (for exit/gateway nodes)

For exit segment selection:

- `"exit"` in `node.roles`
- `node.exit_policy` is not null
- `exit_policy.allow_protocols` contains required protocol
- Destination port in `exit_policy.allow_ports` (or `[0]` = all)
- Destination not in `exit_policy.deny_destinations`
- `exit_policy.logging_policy` satisfies client requirement (if specified)
- `exit_policy.jurisdiction` satisfies client jurisdiction constraints (if specified)
- `exit_policy.ech_capable` = true if stream requires ECH

### 10.6 Mesh Transport Filter

For mesh segments:

- Node has matching transport in `mesh_endpoints`
- Transport type matches segment configuration (WiFi 802.11s, LoRa, BLE, Ethernet P2P)

### 10.7 Domain Membership Filter

For segments requiring nodes in a specific routing domain:

```python
candidates = [n for n in candidates if required_domain_id in n.serves_domains]
```

### 10.8 Bridge Capability Filter (Bridge Positions)

For DOMAIN_BRIDGE hops:

```python
candidates = [n for n in candidates
              if n.bridge_capable
              and has_bridge(n, from_domain, to_domain)
              and get_bridge(n, from_domain, to_domain).available]
```

### 10.9 Satellite Node Filter

Satellite-connected clearnet nodes (`latency_class = satellite-*`):

- Excluded from path positions requiring low latency (interactive traffic).
- Eligible for bulk transfer paths, DHT sync roles, and non-interactive relay positions.

---

## 11. Path Constraints

Path constraints are enforced during node selection. Violating a constraint is a hard failure — the algorithm retries rather than relaxes the constraint.

### 11.1 Mandatory Constraints

These constraints MUST be enforced for all paths:

#### 11.1.1 ASN Diversity

No two nodes in the same path may have the same ASN (Autonomous System Number).

```
for all pairs (n_i, n_j) in path where i ≠ j:
    assert n_i.asn ≠ n_j.asn
```

Rationale: An adversary who controls a major ASN (e.g., a large transit provider) could observe traffic at multiple points in the path if two hops share the same AS.

#### 11.1.2 Country Diversity

No two nodes in the same path may be in the same country.

```
for all pairs (n_i, n_j) in path where i ≠ j:
    assert n_i.country ≠ n_j.country
```

Rationale: Legal coercion operates at national boundaries. Two hops in the same country may both be subject to the same legal order.

#### 11.1.3 Operator Diversity

No two nodes in the same path may share an operator_id.

```
for all pairs (n_i, n_j) in path where i ≠ j:
    if n_i.operator_id is not null and n_j.operator_id is not null:
        assert n_i.operator_id ≠ n_j.operator_id
```

Rationale: An adversary who operates multiple nodes could place them in the same path if operator identity were not checked. This constraint prevents a single operator from controlling both entry and exit.

#### 11.1.4 Node Uniqueness

A node may not appear more than once in a path.

```
assert len(path) == len(set(node.node_id for node in path))
```

### 11.2 Default Constraints (Applied Unless Overridden)

#### 11.2.1 Minimum Uptime

```
assert node.uptime_pct >= 85.0
```

Nodes with low uptime reduce path reliability and anonymity (frequent circuit rebuilds are observable).

#### 11.2.2 Region Diversity

At least two different regions (NA, EU, AP, SA, AF, ME) must be represented in the path.

```
regions = set(n.region for n in path)
assert len(regions) >= 2
```

### 11.3 Cross-Domain Diversity Enforcement

Diversity is enforced across the **ENTIRE path** — all segments and all domain bridge nodes combined.

Domain boundaries do NOT reset diversity counting. A node in ASN 12345 in the clearnet segment means no other node in ASN 12345 may appear in any mesh segment or bridge position of the same path.

### 11.4 Optional Constraints (Configurable per Application)

| Constraint | Description | Default |
|---|---|---|
| max_latency_ms | Maximum acceptable total path latency estimate | None |
| min_bandwidth_mbps | Minimum bandwidth each hop must report | 10 |
| min_reputation_score | Minimum node reputation score (0–100) | 60 |
| exit_jurisdiction_whitelist | Only allow exits in listed countries | None |
| exit_jurisdiction_blacklist | Exclude exits in listed countries | None |
| exit_logging_policy | Only use exits with specific logging policy | None |
| path_length | Override default path length | 4 |
| geographic_spread_km | Minimum geographic distance between hops | None |

---

## 12. Domain Bridge Discovery Algorithm

```python
function discover_domain_bridges(from_domain, to_domain):

    # Compute symmetric bridge key
    sorted_pair = sorted([from_domain.id_bytes, to_domain.id_bytes])
    bridge_key = BLAKE3("bgpx-domain-bridge-v1" || sorted_pair[0] || sorted_pair[1])

    # Query unified DHT
    bridge_record = dht_get(bridge_key)
    if bridge_record is None:
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    if not verify_bridge_record_signature(bridge_record):
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    # Individually verify each listed bridge node
    verified_bridges = []
    for node_id in bridge_record.bridge_nodes:
        node_advert = fetch_node_advertisement(node_id)
        if node_advert and verify_node_advertisement(node_advert):
            if node_advert.bridge_capable:
                if serves_both_domains(node_advert, from_domain, to_domain):
                    if not is_blacklisted(node_id):
                        verified_bridges.append(node_advert)

    if not verified_bridges:
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    return verified_bridges
```

Individual node advertisement verification is MANDATORY — pool membership or bridge record inclusion alone is insufficient.

---

## 13. Scoring Function

After filtering, each candidate node is assigned a score for weighted random selection.

### 13.1 Score Components

```
score(node, domain) = 0.30 × S_uptime(node)
                    + 0.25 × S_latency(node, domain)
                    + 0.20 × S_bandwidth(node)
                    + 0.15 × S_reputation(node)
                    + 0.10 × S_geo_plausibility(node, domain)
```

Default weights sum to 1.0. Clients MAY override weights via SDK.

### 13.2 Component Functions

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

### 13.3 Transport-Adjusted Latency Scoring

| Transport | Latency Baseline | Score Reference |
|---|---|---|
| Internet UDP | 500ms max for 0 score | Standard |
| WiFi 802.11s | 100ms max for 0 score | Tighter (mesh should be fast) |
| LoRa | 5000ms max for 0 score | Wider (LoRa is inherently high-latency) |
| Satellite | Not scored by latency | Skip latency score; return 0.5 neutral |

### 13.4 Bridge Node Scoring (Specialized)

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

## 14. Diversity Enforcement

### 14.1 Mandatory Constraints (Cross-Segment)

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

### 14.2 Default Constraints (Applied Unless Overridden)

```python
# Minimum uptime
assert all(n.uptime_pct >= 85.0 for n in path), "Node below minimum uptime"

# Region diversity (at least 2 different regions)
regions = set(n.region for n in all_path_nodes)
assert len(regions) >= 2, "Insufficient region diversity"
```

### 14.3 Constraint Violation Handling

On constraint violation during selection:

1. Remove one of the conflicting nodes from consideration (randomly chosen between conflicting pair).
2. Retry selection for that position.
3. After 10 retries for a position: attempt with reduced path length (if > minimum).
4. If minimum path length cannot be satisfied: fail with `ERR_SEGMENT_CONSTRUCTION_FAILED`.

---

## 15. Path Length Selection

### 15.1 Defaults (Not Enforced Limits)

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain (2 segments + 1 bridge) | 7 |
| Cross-domain (3 segments + 2 bridges) | 10 |
| High-security application-specified | Any |

### 15.2 N-Hop Unlimited Policy

The protocol imposes **no global maximum**. Default configuration maximum: 15 hops (operator-settable). Soft warning is logged for paths exceeding 15 hops (high latency expected). Implementations MUST NOT enforce a maximum hop count at the protocol level.

### 15.3 Automatic Adjustment

| Condition | Adjustment |
|---|---|
| Network has < 50 nodes in database | Reduce to 3 hops (minimum) |
| Application requests high-security mode | Increase to 5-7 hops |
| Path latency exceeds application budget | Reduce by 1 hop (minimum 3) |
| Cover traffic enabled | No length adjustment needed |

### 15.4 Application-Specified Path Length

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

## 16. Path Selection Algorithm

### 16.1 Input

- Available nodes from DHT (filtered to valid, non-expired, non-blacklisted)
- Segment configuration (pool per segment, hops per segment)
- Diversity constraints (ASN, country, operator)
- Application-level constraints (ECH, jurisdiction, logging policy)
- path_id: generated AFTER successful path construction

### 16.2 Cross-Domain Path Construction Entry Point

```python
function build_path(config):
    if config.domain_segments:
        # Cross-domain path
        return build_cross_domain_path(config.domain_segments, config.constraints)
    elif config.segments:
        # Single-domain pool-based path (backward compatible)
        return build_single_domain_path(config.segments, config.constraints)
    else:
        # Default: single-domain clearnet, 4 hops, default pool
        return build_single_domain_path(
            [{ pool = "bgpx-default", hops = 4, exit = true }],
            config.constraints
        )
```

### 16.3 Pool-Aware Segment Selection (Single Domain)

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

### 16.4 Full Cross-Domain Construction

```python
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

## 17. Weighted Random Selection

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

## 18. Multi-Path Routing

For high-throughput or high-reliability applications, BGP-X supports building multiple paths simultaneously.

### 18.1 Path Isolation

When building multiple paths, the routing algorithm enforces cross-path isolation:

- No node appears in more than one path simultaneously
- No operator appears in more than one path simultaneously
- No ASN appears in more than one path simultaneously

### 18.2 Load Distribution

Streams are distributed across available paths using round-robin assignment. Sensitive streams MAY be pinned to a specific path.

---

## 19. path_id Generation

After successful path construction:

```python
path_id = CSPRNG.generate_bytes(8)
while path_id == bytes(8):  # Never all-zeros
    path_id = CSPRNG.generate_bytes(8)
```

path_id is included in every onion layer including DOMAIN_BRIDGE layers. The same value is used throughout the entire cross-domain path.

path_id changes on path rebuild. Old path_id mappings at relay nodes expire with the TTL.

---

## 20. Path Caching and Rebuild

### 20.1 Default Cache TTL

Default: **10 minutes**

### 20.2 Cache Invalidation

Path invalidation triggers:

- Any node in path goes offline (KEEPALIVE timeout)
- Any node in path blacklisted
- Node advertisement expires
- Cache TTL exceeded
- Client explicitly requests rebuild
- Pool curator key rotation (re-verify pool members)
- Domain bridge node goes offline
- Mesh island becomes unreachable
- DOMAIN_ADVERTISE record for required bridge pair expires
- MESH_ISLAND_ADVERTISE record expires

Bridge-level discovery cache: 5-minute TTL (shorter than path cache — bridge availability changes more frequently).

### 20.3 Rebuild Triggers (MUST rebuild)

- Any node in path goes offline (KEEPALIVE timeout)
- Any node in path blacklisted
- Path age exceeds TTL
- Pool curator key rotation (re-verify pool members)
- Any path segment fails (all nodes in segment unresponsive)

### 20.4 Rebuild Triggers (SHOULD rebuild)

- Path quality score below rebuild_quality_threshold (default 0.20)
- Measured RTT exceeds 3× baseline
- Packet loss exceeds 5% sustained over 30 seconds

### 20.5 Segment-Level Rebuild

When one segment fails, attempt segment-only rebuild before full path rebuild:

```
1. Identify failed segment
2. Select replacement nodes for that segment only
3. Establish new sessions with replacement segment nodes
4. Migrate in-flight streams to new segment
5. If segment rebuild fails: rebuild entire path
```

### 20.6 Cross-Domain Segment Rebuild

1. Identify failed segment or bridge transition
2. Attempt to rebuild only the failed part
3. If bridge transition fails: attempt alternative bridge node
4. If no alternative: rebuild affected portion
5. If full path fails: `FAIL(ERR_NO_CROSS_DOMAIN_PATH)`

### 20.7 Seamless Rebuild

Client maintains path pool (2-3 pre-built paths). When current path retires, replacement is immediately available:

- Pre-build starts when current path is assigned
- Different pool configurations pre-build separately
- Isolated paths (for sensitive streams) pre-built independently

Pool pre-building: different domain segment configurations pre-build separately. A client may maintain:
- 2 pre-built clearnet-only paths
- 1 pre-built path for each commonly used mesh island
- 1 pre-built path for each commonly used satellite bridge

---

## 21. Path Construction Timing

Path construction is not instantaneous. Clients SHOULD pre-build paths before they are needed to avoid latency at connection time.

Recommended behavior:

- Maintain 2–3 pre-built paths ready for use
- Begin rebuilding a path as soon as the previous one is used
- For latency-sensitive applications, accept a pre-built path from the pool immediately rather than waiting for fresh construction

For cross-domain paths, pre-building is especially valuable because:
- Domain bridge discovery adds latency
- Mesh island path segments may require additional DHT queries
- A pre-built cross-domain path can be accepted immediately

---

## 22. Retry Logic

If path construction fails (no valid set of nodes satisfying all constraints):

1. **Retry with fresh DHT query** (up to 3 times): The node set may be stale. Wait 2 seconds between retries.
2. **Relax optional constraints** (in order of least privacy impact): Increase max_latency_ms, reduce min_bandwidth_mbps, reduce min_reputation_score.
3. **Reduce path length** if > minimum (try preferred-1 hops).
4. **Segment fallback**: if `fallback_to_default = true` and pool is insufficient, use default pool for this segment.
5. **Fail and report**: If mandatory constraints cannot be satisfied, path construction MUST fail with specific error. Do NOT relax mandatory constraints.

Applications SHOULD retry path construction after a delay (default: 5 seconds) when it fails.

---

## 23. Pool Curator Key Rotation Effects

When a pool curator key rotation is accepted:

1. New curator key cached
2. All pool members signed under old curator key flagged for re-verification
3. 24-hour grace period: old-key members still eligible (with warning)
4. After grace period: old-key members excluded until re-signed by curator under new key
5. Curator re-signs all members within 24 hours as part of rotation procedure

---

## 24. Offline Island Handling

When a mesh island's bridge nodes are all offline:
- Path construction to destinations within that island: FAIL with ERR_MESH_ISLAND_UNREACHABLE
- If `store_and_forward = true` in path config: queue request for delivery when island reconnects
- Client retries periodically (configurable interval, default 60 seconds)
- When bridge reconnects: island advertisement published to unified DHT; path construction succeeds

---

## 25. Bootstrap Path

When a client first joins with no DHT knowledge, it needs a bootstrap path to query the DHT.

Bootstrap sequence:

1. Connect to a well-known bootstrap node (hardcoded in client distribution)
2. Send DHT_FIND_NODE to discover nearby nodes
3. Use the discovered nodes to construct a proper BGP-X path
4. Re-route DHT queries through the constructed path

For cross-domain capability: after populating DHT routing table, additionally prefetch DOMAIN_ADVERTISE records for commonly needed bridge pairs.

Bootstrap nodes MUST NOT be used as relay/entry/exit nodes in constructed paths. They serve only to provide initial DHT access.

---

## 26. Mesh Island Path Construction

Within a mesh island segment, path construction queries the unified DHT with domain filter:

```python
function get_nodes_in_domain_and_pool(domain, pool):
    if domain.type == "mesh":
        candidates = dht.find_nodes(
            target = pool_id_or_random,
            domain_filter = domain.id,
            max_results = 50
        )
        return [n for n in candidates
                if n.serves_domain(domain) and verify_individually(n)]
    else:
        return dht.find_nodes(target=pool_id_or_random, max_results=50)
```

---

## 27. Return Path Consideration

Return traffic uses `path_id` routing (relay nodes store `path_id → predecessor` mapping). The routing algorithm does not need to construct a separate return path — `path_id` enables bidirectional use of the same constructed path across all domain boundaries.

For long-lived bidirectional sessions, a separate return path MAY be constructed using different nodes. This reduces correlation between inbound and outbound timing patterns.

---

## 28. Path Quality Monitoring

Clients SHOULD monitor path quality during use:

- Measure round-trip latency via KEEPALIVE timing
- Track packet loss (inferred from sequence number gaps)
- Track throughput

If path quality falls below acceptable thresholds:

- Attempt to rebuild a new path
- Report path quality data (without identifying information) to the reputation system

---

## 29. Complete Path Construction Pseudocode (Single-Domain)

```python
function build_single_domain_path(segment_configs, constraints, fallback_to_default):

    all_selected = []
    segment_results = []

    for segment in segment_configs:
        candidates = get_pool_members(segment.pool)
        candidates = verify_individuals(candidates)
        candidates = filter_mandatory(candidates, already_selected=all_selected)

        if len(candidates) < segment.hops:
            if fallback_to_default and segment.pool != "bgpx-default":
                candidates = get_default_pool_members()
                candidates = filter_mandatory(candidates, already_selected=all_selected)
                if len(candidates) < segment.hops:
                    return FAIL(ERR_POOL_INSUFFICIENT_NODES)
            else:
                return FAIL(ERR_POOL_INSUFFICIENT_NODES)

        selected = []
        for i in range(segment.hops):
            remaining = [n for n in candidates if n not in selected and n not in all_selected]
            eligible = [n for n in remaining if diversity_ok(n, selected + all_selected)]
            if not eligible:
                return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)
            node = weighted_random_select(eligible)
            selected.append(node)
            all_selected.append(node)

        segment_results.append(selected)

    # Final cross-path diversity check
    if not check_all_diversity_constraints(all_selected):
        return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)

    # Generate path_id
    path_id = generate_path_id()

    return (all_selected, segment_results, path_id)
```

---

## 30. Metrics

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
| `path_cache_hits` | Paths reused from cache |
| `path_cache_misses` | Paths requiring fresh construction |
| `segment_rebuilds` | Segment-only rebuild attempts |
| `full_path_rebuilds` | Complete path rebuilds |

No node identifiers, path identifiers, or client data in metrics.
