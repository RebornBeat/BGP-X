# BGP-X Routing Algorithm Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X routing is **client-driven inter-protocol domain source routing**. The client constructs the complete cross-domain path before transmitting. The routing algorithm:

1. Determines domain sequence required
2. Discovers bridge nodes for domain transitions
3. Queries domain-filtered DHT for relay candidates per domain segment
4. Filters candidates using mandatory constraints
5. Scores candidates (domain-aware weighted function)
6. Selects via weighted random sampling with cross-domain diversity enforcement
7. Generates path_id

**N-hop unlimited**: no maximum on total path hops, domain segment count, domain traversal count, or domain bridge transition count.

---

## 2. Node Database

```rust
struct NodeDatabaseEntry {
    node_id:                NodeId,
    public_key:             Ed25519PublicKey,
    roles:                  Vec<NodeRole>,
    routing_domains:        Vec<RoutingDomainEntry>,
    bridge_capable:         bool,
    bridges:                Vec<DomainBridge>,
    serves_domains:         Vec<DomainId>,
    asn:                    u32,
    country:                String,
    region:                 Region,
    operator_id:            Option<Ed25519PublicKey>,
    exit_policy:            Option<ExitPolicy>,
    exit_policy_version:    Option<u16>,
    bandwidth_mbps:         f32,
    latency_ms:             f32,
    uptime_pct:             f32,
    reputation_score:       f32,
    geo_plausibility_score: f32,
    last_seen:              Timestamp,
    advertisement_expires:  Timestamp,
    extensions:             ExtensionFlags,
    pool_memberships:       Vec<PoolId>,
    island_memberships:     Vec<String>,
    pt_supported:           Vec<String>,
    ech_capable:            bool,
    is_gateway:             bool,
    link_quality_profiles:  HashMap<TransportType, LinkQualityProfile>,
    withdrawn:              bool,
    blacklisted:            bool,
    blacklist_reason:       Option<String>,
    transport_types:        Vec<TransportType>,
}

struct DomainBridge {
    from_domain:          DomainId,
    to_domain:            DomainId,
    bridge_latency_ms:    f32,
    bridge_transport:     TransportType,
    available:            bool,
}
```

Maximum database size: 10,000 nodes. Eviction: least recently seen first. Withdrawn nodes: immediately removed from eligible pool; retained 7 days for propagation.

---

## 3. Filtering Stage

### Mandatory Filters (All Nodes, All Domains)

| Filter | Condition |
|---|---|
| Not blacklisted | `node.blacklisted == false` |
| Not withdrawn | `node.withdrawn == false` |
| Advertisement valid | `current_time < node.advertisement_expires` |
| Signature valid | Verified at database insertion time |
| Role appropriate | Node has required role for position in path |
| Minimum uptime | `node.uptime_pct >= 85.0` |
| Valid endpoints | Has reachable endpoint in required domain |
| Not already in path | Not selected for this path construction |

### Domain Membership Filter

For segments requiring nodes in a specific routing domain:
```python
candidates = [n for n in candidates if required_domain_id in n.serves_domains]
```

### Bridge Capability Filter (Bridge Positions)

```python
candidates = [n for n in candidates
              if n.bridge_capable
              and has_bridge(n, from_domain, to_domain)
              and get_bridge(n, from_domain, to_domain).available]
```

### ECH Capability Filter (Exit Segments)

```python
if segment_constraints.require_ech and is_exit_segment:
    candidates = [n for n in candidates if n.ech_capable]
```

### Exit Policy Filter (Exit Segments)

```python
if is_exit_segment:
    candidates = [n for n in candidates
                  if "exit" in n.roles
                  and n.exit_policy is not None
                  and satisfies_exit_constraints(n, segment_constraints)]
```

---

## 4. Scoring Function

```
score(node, domain) = 0.30 × S_uptime(node)
                    + 0.25 × S_latency(node, domain)
                    + 0.20 × S_bandwidth(node)
                    + 0.15 × S_reputation(node)
                    + 0.10 × S_geo_plausibility(node, domain)
```

### Component Functions

```python
def S_uptime(node):
    return node.uptime_pct / 100.0

def S_latency(node, domain):
    if domain.type == "clearnet":
        return max(0, 1.0 - (node.latency_ms / 500.0))
    elif domain.type == "mesh":
        if node.has_lora_transport_for_domain(domain):
            return max(0, 1.0 - (node.latency_ms / 5000.0))  # LoRa: wide range
        else:
            return max(0, 1.0 - (node.latency_ms / 100.0))   # WiFi mesh: tight
    elif domain.type == "satellite":
        return 0.5  # Neutral — satellite latency expected and not differentiating
    return max(0, 1.0 - (node.latency_ms / 500.0))  # Default

def S_bandwidth(node):
    return min(1.0, node.bandwidth_mbps / 1000.0)

def S_reputation(node):
    return node.reputation_score / 100.0

def S_geo_plausibility(node, domain):
    if domain.type == "satellite":
        return 0.5  # Exempt
    if node.rtt_measurement_count < 5:
        return 0.5  # Insufficient measurements
    return node.geo_plausibility_score  # Pre-computed by reputation system
```

### Bridge Node Scoring (Specialized)

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

## 5. Cross-Domain Diversity Enforcement

Applied to the ENTIRE path (all segments and bridge nodes combined):

```python
def check_cross_domain_diversity(all_path_nodes):
    asns = [n.asn for n in all_path_nodes]
    if len(asns) != len(set(asns)):
        return False

    countries = [n.country for n in all_path_nodes]
    if len(countries) != len(set(countries)):
        return False

    op_ids = [n.operator_id for n in all_path_nodes if n.operator_id]
    if len(op_ids) != len(set(op_ids)):
        return False

    node_ids = [n.node_id for n in all_path_nodes]
    if len(node_ids) != len(set(node_ids)):
        return False

    return True
```

Domain boundaries do NOT reset diversity counting. Bridge nodes participate in diversity constraints equally with relay nodes.

Default additional constraints (applied unless overridden):
```python
assert len(set(n.region for n in all_path_nodes)) >= 2  # At least 2 different regions
assert all(n.uptime_pct >= 85.0 for n in all_path_nodes)
```

---

## 6. Path Length Selection

Defaults (not enforced limits):

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain (2 segments + 1 bridge) | 7 |
| Cross-domain (3 segments + 2 bridges) | 10 |
| High-security application-specified | Any |

No protocol maximum enforced. Daemon logs a soft informational message for paths exceeding 15 hops.

---

## 7. Weighted Random Selection

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

Nodes are NOT selected deterministically by highest score. Weighted random ensures high-scoring nodes are proportionally likely, but low-scoring eligible nodes have non-zero probability.

---

## 8. Full Cross-Domain Construction

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

## 9. Path Caching and Rebuild

Default cache TTL: 10 minutes.

Additional cross-domain cache invalidation:
- Domain bridge node goes offline (KEEPALIVE timeout)
- Mesh island becomes unreachable (all bridges offline)
- DOMAIN_ADVERTISE record for required bridge pair expires
- MESH_ISLAND_ADVERTISE record expires
- Pool curator key rotation affecting any node in any domain segment

Bridge-level discovery cache: 5-minute TTL (shorter than path cache — bridge availability changes more frequently).

**Segment-level rebuild for cross-domain**:
1. Identify failed segment or bridge transition
2. Attempt to rebuild only the failed part
3. If bridge transition fails: attempt alternative bridge node
4. If no alternative: rebuild affected portion
5. If full path fails: FAIL with ERR_NO_CROSS_DOMAIN_PATH

---

## 10. Metrics

| Metric | Description |
|---|---|
| path_construction_attempts | Total attempts |
| path_construction_successes | Successes |
| path_construction_failures | Failures (with reason breakdown) |
| path_construction_latency_ms | Time to construct |
| cross_domain_constructions | Paths using domain_segments |
| domain_bridge_selections | Bridge node selection attempts |
| domain_bridge_failures | Bridge node selection failures |
| node_database_size | Current entries |
| node_database_by_region | Count per region |
| bridge_nodes_in_database | Count of bridge-capable nodes |
| island_advertisements_cached | Count of mesh island advertisement cache entries |
| pool_fallbacks | Count of pool insufficient → default fallback |

No node identifiers, path identifiers, or client data in metrics.
