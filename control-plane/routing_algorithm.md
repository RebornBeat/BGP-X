# BGP-X Routing Algorithm Specification

**Version**: 0.1.0-draft

This document specifies the complete routing algorithm used by BGP-X clients to select paths, score nodes, and make routing decisions.

---

## 1. Overview

BGP-X routing is **client-driven source routing**. The client constructs the complete path before transmitting any data. No relay node participates in path selection decisions.

This is fundamentally different from BGP (where ISP routers make per-hop routing decisions) and from Tor (where the client selects nodes but the directory authority influences which nodes are available). In BGP-X, the client:

1. Queries the DHT for available nodes
2. Filters nodes using hard constraints
3. Scores remaining nodes using a weighted function
4. Selects nodes using weighted random sampling
5. Verifies the assembled path meets diversity requirements
6. Constructs onion-encrypted packets for the selected path

---

## 2. Node Database

The routing algorithm operates on a local **node database** maintained by the client.

### 2.1 Node database contents

For each known node, the database stores:

```
{
  node_id:            bytes[32]
  public_key:         bytes[32]
  roles:              set<string>
  endpoints:          list<Endpoint>
  asn:                uint32
  country:            string
  region:             string
  operator_id:        bytes[32] | null
  exit_policy:        ExitPolicy | null
  bandwidth_mbps:     float
  latency_ms:         float
  uptime_pct:         float
  reputation_score:   float  # 0.0–100.0, maintained by reputation system
  last_seen:          timestamp
  advertisement_expires: timestamp
  extensions:         ExtensionFlags
  blacklisted:        bool
  blacklist_reason:   string | null
}
```

### 2.2 Node database maintenance

- Nodes are added to the database as their advertisements are retrieved from the DHT
- Nodes are removed from the database when their advertisement expires AND they have not been seen within the past 2 hours
- Blacklisted nodes remain in the database (with `blacklisted = true`) for 30 days for blacklist propagation purposes, then are removed
- The database is persisted to disk (encrypted) across client restarts

### 2.3 Database size limits

Maximum node database size: **10,000 nodes**

When the limit is reached, the least recently seen nodes are evicted first, subject to the constraint that at least 5 nodes from each region are retained if available.

---

## 3. Filtering Stage

Before scoring, the routing algorithm filters nodes to those eligible for use in the current path.

### 3.1 Mandatory filters (always applied)

| Filter | Condition to pass |
|---|---|
| Not blacklisted | `node.blacklisted == false` |
| Advertisement valid | `current_time < node.advertisement_expires` |
| Signature valid | Advertisement signature verified (done at database insertion time) |
| Role appropriate | Node's roles include the required role for this position in the path |
| Minimum uptime | `node.uptime_pct >= 85.0` |
| No private endpoints | All endpoints are publicly routable addresses |
| Not already in path | Node not already selected for this path |

### 3.2 Version compatibility filter

```
node.protocol_versions_min <= session_version <= node.protocol_versions_max
```

Only nodes that support the client's selected protocol version are eligible.

### 3.3 Exit policy filter (for exit nodes only)

When selecting an exit node, the filter additionally requires:

- `"exit"` in node.roles
- `node.exit_policy` is not null
- `node.exit_policy.allow_protocols` contains the required protocol
- Destination port is in `node.exit_policy.allow_ports` (or `[0]` meaning all)
- Destination address is not in `node.exit_policy.deny_destinations`
- `node.exit_policy.logging_policy` satisfies client's logging policy requirement (if specified)
- `node.exit_policy.jurisdiction` satisfies client's jurisdiction constraints (if specified)

---

## 4. Scoring Function

After filtering, each candidate node is assigned a score used for weighted random selection.

### 4.1 Score components

```
score(node) = w_uptime    × S_uptime(node)
            + w_latency   × S_latency(node)
            + w_bandwidth × S_bandwidth(node)
            + w_reputation× S_reputation(node)
```

Default weights:

| Component | Weight | Symbol |
|---|---|---|
| Uptime | 0.35 | w_uptime |
| Latency | 0.30 | w_latency |
| Bandwidth | 0.20 | w_bandwidth |
| Reputation | 0.15 | w_reputation |

Weights sum to 1.0.

### 4.2 Component functions

#### Uptime score

```
S_uptime(node) = node.uptime_pct / 100.0
```

Range: [0.0, 1.0]

#### Latency score

Lower latency is better. The score is an inverse function, normalized to the range of observed latencies:

```
S_latency(node) = 1.0 - (node.latency_ms / MAX_LATENCY_MS)
MAX_LATENCY_MS = 500.0
```

Nodes with latency > 500ms receive S_latency = 0.0 and are effectively excluded.

#### Bandwidth score

```
S_bandwidth(node) = min(node.bandwidth_mbps, MAX_SCORE_BANDWIDTH) / MAX_SCORE_BANDWIDTH
MAX_SCORE_BANDWIDTH = 1000.0
```

Bandwidth above 1000 Mbps does not improve the score further (diminishing returns for relay purposes).

#### Reputation score

```
S_reputation(node) = node.reputation_score / 100.0
```

The reputation score is maintained by the reputation system (see `/control-plane/reputation_system.md`). Range: [0.0, 100.0].

### 4.3 Weighted random selection

Nodes are not selected deterministically by highest score. Instead, **weighted random sampling without replacement** is used:

```
function weighted_random_select(candidates):
    total_weight = sum(score(n) for n in candidates)
    r = random_float(0.0, total_weight)
    cumulative = 0.0
    for node in candidates:
        cumulative += score(node)
        if r <= cumulative:
            return node
```

This ensures that high-scoring nodes are more likely to be selected, but low-scoring nodes still have a non-zero probability. This prevents deterministic path selection that would make traffic patterns predictable and prevent any node from dominating the network simply by manipulating performance metrics.

---

## 5. Diversity Enforcement

After scoring-based selection, the assembled path is verified against diversity constraints. If any constraint is violated, the selection is retried.

### 5.1 ASN diversity check

```
asns = [node.asn for node in path]
assert len(asns) == len(set(asns))
# All ASNs must be unique
```

### 5.2 Country diversity check

```
countries = [node.country for node in path]
assert len(countries) == len(set(countries))
# All countries must be unique
```

### 5.3 Operator diversity check

```
operator_ids = [node.operator_id for node in path if node.operator_id is not None]
assert len(operator_ids) == len(set(operator_ids))
# All operator IDs must be unique
```

### 5.4 Region diversity check (default constraint)

```
regions = set(node.region for node in path)
assert len(regions) >= 2
# At least 2 different regions represented
```

### 5.5 Constraint violation handling

If any mandatory diversity constraint is violated:

1. Identify which constraint failed and which nodes are in conflict
2. Remove one of the conflicting nodes from consideration for this path (randomly chosen between the two conflicts)
3. Retry selection for that position
4. If no valid node can be found after 10 retries, attempt with a reduced path length (if length > minimum)
5. If minimum path length constraints cannot be met, fail path construction

---

## 6. Path Length Selection

Default path length: **4 hops**

### 6.1 Automatic path length adjustment

The client may automatically adjust path length based on:

| Condition | Adjustment |
|---|---|
| Network has < 50 nodes in database | Reduce to 3 hops (minimum) |
| Application requests high-security mode | Increase to 5–7 hops |
| Path latency exceeds application budget | Reduce by 1 hop (minimum 3) |
| Cover traffic enabled | No length adjustment needed |

### 6.2 Application-specified path length

Applications using the BGP-X SDK can specify a minimum and maximum path length:

```rust
PathConfig {
    min_hops: 3,
    max_hops: 7,
    preferred_hops: 4,
    // ...
}
```

The routing algorithm selects the preferred_hops length if possible, falls back to any value in [min_hops, max_hops] if necessary.

---

## 7. Multi-Path Routing

For high-throughput or high-reliability applications, BGP-X supports building multiple paths simultaneously.

### 7.1 Path isolation

When building multiple paths, the routing algorithm enforces cross-path isolation:

- No node appears in more than one path simultaneously
- No operator appears in more than one path simultaneously
- No ASN appears in more than one path simultaneously

This prevents an adversary who controls one node from appearing in multiple paths used by the same client.

### 7.2 Load distribution

Streams are distributed across available paths using a round-robin assignment. Sensitive streams MAY be pinned to a specific path.

---

## 8. Path Rebuilding

A path MUST be rebuilt when:

- Any node in the path goes offline (KEEPALIVE timeout)
- Any node in the path is blacklisted
- The path's age exceeds the maximum reuse window (default: 10 minutes)
- The application explicitly requests a new path

A path SHOULD be rebuilt when:

- Measured path latency exceeds 3× the initial measured latency
- Packet loss exceeds 5% sustained over 30 seconds

### 8.1 Seamless rebuilding

The client SHOULD pre-build a replacement path before the current path's TTL expires. When the current path is retired, the replacement is immediately available.

### 8.2 Stream migration

When a path is rebuilt while streams are in flight:

- In-flight data on the old path is allowed to drain (up to 10 seconds)
- New streams are immediately opened on the new path
- Existing streams are migrated to the new path if they are still active after draining

---

## 9. Return Path Routing

For request-response traffic (HTTP, API calls, etc.), the return path from the exit node back to the client is the reverse of the forward path.

For long-lived bidirectional sessions, a separate return path MAY be constructed using different nodes. This reduces correlation between inbound and outbound traffic patterns.

---

## 10. Routing Algorithm Pseudocode (Complete)

```
function build_path(destination, config):

    # Fetch candidate nodes
    candidates = node_database.get_valid_nodes()

    # Determine path length
    n_hops = config.preferred_hops or DEFAULT_PATH_LENGTH

    max_attempts = 20
    for attempt in range(max_attempts):

        # Select entry node
        entry_candidates = filter_by_role(candidates, "entry")
        entry_candidates = apply_mandatory_filters(entry_candidates)
        if len(entry_candidates) == 0:
            raise PathConstructionError("No eligible entry nodes")
        entry = weighted_random_select(entry_candidates)

        # Select relay nodes
        relays = []
        used = {entry}
        relay_candidates = filter_by_role(candidates, "relay")
        relay_candidates = apply_mandatory_filters(relay_candidates)

        success = true
        for i in range(n_hops - 2):
            eligible = [n for n in relay_candidates
                        if n not in used
                        and diversity_compatible(n, list(used), config)]
            if len(eligible) == 0:
                success = false
                break
            relay = weighted_random_select(eligible)
            relays.append(relay)
            used.add(relay)

        if not success:
            continue  # retry

        # Select exit node
        exit_candidates = filter_by_role(candidates, "exit")
        exit_candidates = apply_mandatory_filters(exit_candidates)
        exit_candidates = apply_exit_policy_filter(exit_candidates, destination, config)
        exit_candidates = [n for n in exit_candidates
                           if n not in used
                           and diversity_compatible(n, list(used), config)]

        if len(exit_candidates) == 0:
            continue  # retry

        exit_node = weighted_random_select(exit_candidates)

        # Assemble path
        path = [entry] + relays + [exit_node]

        # Final diversity check (belt and suspenders)
        if not check_all_diversity_constraints(path, config):
            continue  # retry

        return path

    raise PathConstructionError(
        "Could not construct path satisfying all constraints after max_attempts"
    )
```

---

## 11. Routing Metrics and Observability

The routing algorithm SHOULD expose the following metrics for operational monitoring (no user-identifiable data):

| Metric | Description |
|---|---|
| `path_construction_attempts` | Total path construction attempts |
| `path_construction_successes` | Successful path constructions |
| `path_construction_failures` | Failed path constructions (with reason) |
| `path_construction_latency_ms` | Time to construct a path |
| `node_database_size` | Current number of nodes in database |
| `node_database_by_region` | Node count per region |
| `path_age_at_rebuild_seconds` | Age of path when it was rebuilt |
| `diversity_constraint_violations` | Constraint violations by type |

These metrics MUST NOT include any information about which nodes were selected, which destinations were accessed, or any session identifiers.
