# BGP-X Path Construction Specification

**Version**: 0.1.0-draft

---

## 1. Overview

Path construction is the process by which a BGP-X client selects an ordered sequence of relay nodes to form a path from client to destination. BGP-X uses **client-side path selection** — the sending party constructs the complete path before transmission.

BGP-X path construction is **inter-protocol domain-aware**: paths may span multiple routing domains (clearnet, BGP-X overlay, mesh islands) in any order, any number of times, with no protocol-level maximum on total hops, domains, or transitions.

No relay node participates in path selection. The client:

1. Determines the domain sequence required for the path
2. Queries the unified DHT for candidate nodes per domain segment
3. Discovers domain bridge nodes for each required domain transition
4. Filters nodes using hard constraints
5. Scores remaining nodes using a weighted function (domain-aware)
6. Selects nodes using weighted random sampling with cross-domain diversity enforcement
7. Verifies the assembled path meets diversity requirements
8. Generates path_id (8 bytes, CSPRNG)
9. Constructs onion-encrypted packets for the selected cross-domain path

---

## 2. N-Hop Unlimited Policy

BGP-X path construction imposes **no maximum on total path hops, domain count, domain traversal count, or segment count**.

The routing algorithm will construct a path of any length the client requests, provided:
- Sufficient eligible nodes exist in each domain segment
- Mandatory diversity constraints can be satisfied
- The 3-hop absolute minimum is met

Practical defaults (per-path configuration, not enforced limits):

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain, 2 segments + 1 bridge | 7 (3+1+3) |
| Cross-domain, 3 segments + 2 bridges | 10 (3+1+3+1+2) |
| High-security application-specified | Any |

The daemon logs a soft informational message when total hops exceeds 15 (high latency expected) but does NOT reject or limit the path.

---

## 3. Path Anatomy

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

**Minimum total path length**: 3 hops across the full path regardless of domain count.

---

## 4. Domain Segment Configuration

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

## 5. Any-to-Any Connectivity

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

## 6. DHT Pools with Domain Scope

Pools may be scoped to a specific routing domain:

```json
{
  "pool_id": "...",
  "domain_scope": "mesh:lima-district-1",
  "members": ["node_id_1", "node_id_2"],
  ...
}
```

`domain_scope` values:
- `"any"` (default): pool contains nodes from any domain — backward compatible
- `"clearnet"`: clearnet-reachable nodes only
- `"mesh:<island_id>"`: nodes in specified mesh island only
- `"bridge:clearnet→mesh:<island_id>"`: bridge nodes for this specific pair

Domain-scoped pools are stored in the unified DHT. The `domain_scope` field identifies the pool's scope for client filtering.

---

## 7. Domain Bridge Discovery Algorithm

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

## 8. Cross-Domain Path Selection Algorithm

```python
function build_cross_domain_path(domain_segments, constraints):

    all_selected = []
    segment_results = []
    current_domain = None

    for i, segment in enumerate(domain_segments):

        if segment.type == "bridge":
            from_domain = get_previous_domain(domain_segments, i)
            to_domain = get_next_domain(domain_segments, i)

            bridge_candidates = discover_domain_bridges(from_domain, to_domain)
            if bridge_candidates is FAIL:
                if constraints.fallback_any_bridge:
                    bridge_candidates = discover_any_bridge_to(to_domain)
                if bridge_candidates is FAIL or not bridge_candidates:
                    return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

            eligible = [n for n in bridge_candidates
                        if n not in all_selected
                        and diversity_ok(n, all_selected)]

            if not eligible:
                return FAIL(ERR_NO_CROSS_DOMAIN_PATH)

            node = weighted_random_select(
                eligible,
                score_fn=score_bridge_node(from_domain, to_domain)
            )
            all_selected.append(node)
            segment_results.append(('bridge', [node]))
            current_domain = to_domain

        elif segment.type == "segment":
            domain = segment.domain or current_domain
            pool = segment.pool or "bgpx-default"
            hops = segment.hops

            candidates = get_nodes_in_domain_and_pool(domain, pool)
            candidates = [n for n in candidates
                          if verify_individually(n) and not is_blacklisted(n)]
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
                remaining = [n for n in candidates
                             if n not in selected and n not in all_selected]
                eligible = [n for n in remaining
                            if diversity_ok(n, selected + all_selected)]
                if not eligible:
                    return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)
                node = weighted_random_select(
                    eligible,
                    score_fn=score_node_in_domain(domain)
                )
                selected.append(node)
                all_selected.append(node)

            segment_results.append(('segment', selected))
            current_domain = domain

    # Final cross-domain diversity check
    if not check_cross_domain_diversity(all_selected):
        return FAIL(ERR_SEGMENT_CONSTRUCTION_FAILED)

    path_id = generate_path_id()
    return (all_selected, segment_results, path_id)
```

---

## 9. Node Scoring Function (Domain-Aware)

```
score(node, domain) = 0.30 × S_uptime(node)
                    + 0.25 × S_latency(node, domain)
                    + 0.20 × S_bandwidth(node)
                    + 0.15 × S_reputation(node)
                    + 0.10 × S_geo_plausibility(node, domain)
```

**S_latency(node, domain)**:
```
clearnet:    max(0, 1.0 - (node.latency_ms / 500.0))
WiFi mesh:   max(0, 1.0 - (node.latency_ms / 100.0))
LoRa:        max(0, 1.0 - (node.latency_ms / 5000.0))
satellite:   0.5  (neutral — expected high latency)
```

**S_geo_plausibility(node, domain)**:
```
satellite nodes: 0.5 (exempt)
mesh LoRa nodes: mesh-specific thresholds (100ms-5s expected)
others:          RTT-based ratio vs expected region range
```

**Bridge node scoring** (specialized):
```
score_bridge(node, from_domain, to_domain) =
    0.30 × min(uptime_in_domain(node, from_domain), uptime_in_domain(node, to_domain)) / 100.0
    + 0.25 × max(0, 1.0 - (bridge.bridge_latency_ms / 2000.0))
    + 0.20 × node.reputation_score / 100.0
    + 0.15 × min(1.0, len(node.routing_domains) / 5.0)
    + 0.10 × compute_geo_spread(node.routing_domains)
```

**Weighted random selection (MANDATORY — not deterministic)**:
```python
function weighted_random_select(candidates, score_fn):
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

---

## 10. Cross-Domain Diversity Enforcement

Diversity is enforced across the ENTIRE path — all segments and all domain bridge nodes combined:

```python
def check_cross_domain_diversity(all_path_nodes):

    # ASN diversity: no two nodes in same ASN across entire path
    asns = [n.asn for n in all_path_nodes]
    if len(asns) != len(set(asns)):
        return False

    # Country diversity: no two nodes in same country
    countries = [n.country for n in all_path_nodes]
    if len(countries) != len(set(countries)):
        return False

    # Operator diversity: no two nodes with same operator_id
    op_ids = [n.operator_id for n in all_path_nodes if n.operator_id]
    if len(op_ids) != len(set(op_ids)):
        return False

    # Node uniqueness: no node appears twice
    node_ids = [n.node_id for n in all_path_nodes]
    if len(node_ids) != len(set(node_ids)):
        return False

    return True
```

Domain boundaries do NOT reset diversity counting. A node in ASN 12345 in the clearnet segment means no other node in ASN 12345 may appear in any mesh segment or bridge position of the same path.

Default additional constraints (applied unless overridden):
```python
# At least 2 different regions across entire path
assert len(set(n.region for n in all_path_nodes)) >= 2

# All nodes minimum uptime
assert all(n.uptime_pct >= 85.0 for n in all_path_nodes)
```

---

## 11. path_id Generation

```python
path_id = CSPRNG.generate_bytes(8)
while path_id == bytes(8):  # Never all-zeros
    path_id = CSPRNG.generate_bytes(8)
```

path_id is included in every onion layer including DOMAIN_BRIDGE layers. The same value is used throughout the entire cross-domain path.

---

## 12. Mesh Island Path Construction

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

## 13. Double-Exit Architecture

Double-exit works within a single domain or across domains:

**Single-domain double-exit**:
```
[Entry + Relay, default pool] → [Exit Pool 1] → [Exit Pool 2] → Destination
```

**Cross-domain double-exit** (maximum isolation):
```
[clearnet relays] → [Bridge → mesh] → [mesh relays] → [Bridge → clearnet] → [clearnet exit]
```

The intermediate mesh segment acts as an opacity layer between the two clearnet segments.

---

## 14. Path Caching and Rebuild

Default cache TTL: **10 minutes**. Cross-domain cache invalidation additional triggers:
- Domain bridge node goes offline (KEEPALIVE timeout)
- Mesh island becomes unreachable (all bridges offline)
- DOMAIN_ADVERTISE record for required bridge pair expires
- MESH_ISLAND_ADVERTISE record expires

**Segment-level rebuild for cross-domain paths**:
1. Identify which segment or bridge transition failed
2. Attempt to rebuild only the failed segment/bridge
3. If domain transition fails: rebuild all segments in affected portion
4. If full rebuild fails: fail with ERR_NO_CROSS_DOMAIN_PATH

---

## 15. Offline Island Handling

When a mesh island's bridge nodes are all offline:
- Path construction to destinations within that island: FAIL with ERR_MESH_ISLAND_UNREACHABLE
- If `store_and_forward = true` in path config: queue request for delivery when island reconnects
- Client retries periodically (configurable interval, default 60 seconds)
- When bridge reconnects: island advertisement published to unified DHT; path construction succeeds

---

## 16. Bootstrap Path

When a client first joins with no DHT knowledge, same bootstrap procedure as before using well-known bootstrap nodes. For cross-domain capability: after populating DHT routing table, additionally prefetch DOMAIN_ADVERTISE records for commonly needed bridge pairs.

---

## 17. Complete Path Construction Entry Point

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
