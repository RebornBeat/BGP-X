# BGP-X Path Construction Specification

**Version**: 0.1.0-draft

This document specifies how BGP-X clients select and construct relay paths through the overlay network.

---

## 1. Overview

Path construction is the process by which a BGP-X client selects an ordered sequence of relay nodes to form a path from the client to its destination. BGP-X uses **client-side path selection** — the sending party constructs the complete path before transmission. No relay node participates in path selection.

This is a fundamental design choice. Delegating path selection to the network (as BGP does) would allow network-level adversaries to influence routing. By keeping path selection at the client, BGP-X ensures that no single network entity can force all clients to route through a surveillance point.

---

## 2. Path Anatomy

A path consists of three categories of nodes:

```
[Entry Node] → [Relay Nodes × 0..N] → [Exit Node]
```

| Role | Count | Minimum | Description |
|---|---|---|---|
| Entry | 1 | 1 | First hop. Knows client IP. |
| Relay | 0..N | 0 | Middle hops. No knowledge of origin or destination. |
| Exit | 1 | 1 | Last hop for clearnet. Knows destination. |

Minimum total path length: **3 hops** (entry + 0 relays + exit, where entry ≠ exit)
Default total path length: **4 hops** (entry + 2 relays + exit)

For BGP-X native service connections (no clearnet exit needed), the last node in the path is the service's entry node. The exit node role is replaced by a delivery node.

---

## 3. Path Selection Algorithm

### 3.1 Input

The path selection algorithm takes:

- The set of available nodes from the DHT (filtered to valid, non-expired advertisements)
- The path length N (default: 4)
- Path constraints (see Section 4)
- The current node blacklist (nodes flagged by reputation system)

### 3.2 Output

An ordered list of N nodes: [n_1, n_2, ..., n_N] where n_1 is entry and n_N is exit (for clearnet).

### 3.3 Algorithm

```
function select_path(nodes, length, constraints, blacklist):

    # Step 1: Filter to eligible nodes
    eligible = [n for n in nodes
                if n not in blacklist
                and n.signature_valid
                and n.advertisement_not_expired
                and n.uptime_pct >= MIN_UPTIME_PCT]

    # Step 2: Filter for entry-capable nodes
    entry_candidates = [n for n in eligible if "entry" in n.roles]

    # Step 3: Filter for relay-capable nodes
    relay_candidates = [n for n in eligible if "relay" in n.roles]

    # Step 4: Filter for exit-capable nodes
    exit_candidates = [n for n in eligible
                       if "exit" in n.roles
                       and exit_policy_acceptable(n.exit_policy, constraints)]

    # Step 5: Select entry node
    entry = weighted_random_select(entry_candidates,
                                   weight_fn = score_node)

    # Step 6: Select relay nodes (length - 2 of them)
    relays = []
    for i in range(length - 2):
        eligible_relays = [n for n in relay_candidates
                           if n != entry
                           and n not in relays
                           and diversity_ok(n, [entry] + relays, constraints)]
        relay = weighted_random_select(eligible_relays, weight_fn = score_node)
        relays.append(relay)

    # Step 7: Select exit node
    exit_candidates_filtered = [n for n in exit_candidates
                                  if n != entry
                                  and n not in relays
                                  and diversity_ok(n, [entry] + relays, constraints)]
    exit_node = weighted_random_select(exit_candidates_filtered,
                                        weight_fn = score_node)

    return [entry] + relays + [exit_node]
```

### 3.4 Node Scoring Function

```
function score_node(node):
    score = 0.0
    score += 0.40 * normalize(node.uptime_pct, 0, 100)
    score += 0.30 * normalize(1 / (node.latency_ms + 1), 0, 1)
    score += 0.20 * normalize(node.bandwidth_mbps, 0, MAX_BANDWIDTH)
    score += 0.10 * normalize(node.reputation_score, 0, 100)
    return score
```

Weighted random selection: nodes with higher scores are proportionally more likely to be selected, but all eligible nodes have a non-zero probability. This prevents deterministic path selection that could be exploited by a node with artificially inflated scores.

---

## 4. Path Constraints

Path constraints are enforced during node selection. Violating a constraint is a hard failure — the algorithm retries rather than relaxes the constraint.

### 4.1 Mandatory Constraints

These constraints MUST be enforced for all paths:

#### 4.1.1 ASN Diversity

No two nodes in the same path may have the same ASN (Autonomous System Number).

```
for all pairs (n_i, n_j) in path where i ≠ j:
    assert n_i.asn ≠ n_j.asn
```

Rationale: An adversary who controls a major ASN (e.g., a large transit provider) could observe traffic at multiple points in the path if two hops share the same AS.

#### 4.1.2 Country Diversity

No two nodes in the same path may be in the same country.

```
for all pairs (n_i, n_j) in path where i ≠ j:
    assert n_i.country ≠ n_j.country
```

Rationale: Legal coercion operates at national boundaries. Two hops in the same country may both be subject to the same legal order.

#### 4.1.3 Operator Diversity

No two nodes in the same path may share an operator_id.

```
for all pairs (n_i, n_j) in path where i ≠ j:
    if n_i.operator_id is not null and n_j.operator_id is not null:
        assert n_i.operator_id ≠ n_j.operator_id
```

Rationale: An adversary who operates multiple nodes could place them in the same path if operator identity were not checked. This constraint prevents a single operator from controlling both entry and exit.

#### 4.1.4 Node Uniqueness

A node may not appear more than once in a path.

```
assert len(path) == len(set(node.node_id for node in path))
```

### 4.2 Default Constraints (Applied Unless Overridden)

#### 4.2.1 Minimum Uptime

```
assert node.uptime_pct >= 85.0
```

Nodes with low uptime reduce path reliability and anonymity (frequent circuit rebuilds are observable).

#### 4.2.2 Region Diversity

At least two different regions (NA, EU, AP, SA, AF, ME) must be represented in the path.

```
regions = set(n.region for n in path)
assert len(regions) >= 2
```

### 4.3 Optional Constraints (Configurable per Application)

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

## 5. Retry Logic

If path construction fails (no valid set of nodes satisfying all constraints):

1. **Retry with fresh DHT query** (up to 3 times): The node set may be stale.
2. **Relax optional constraints** (in order of least privacy impact): Increase max_latency_ms, reduce min_bandwidth_mbps, reduce min_reputation_score.
3. **Fail and report**: If mandatory constraints cannot be satisfied, path construction MUST fail. Do not relax mandatory constraints.

Applications SHOULD retry path construction after a delay (default: 5 seconds) when it fails.

---

## 6. Path Caching and Reuse

Constructed paths MAY be cached and reused for subsequent connections within the same session window.

### Cache TTL

Default path cache TTL: **10 minutes**

After TTL expiry, a new path MUST be constructed. Using the same path for extended periods increases the risk of traffic correlation.

### Cache invalidation

Cached paths MUST be invalidated if:

- Any node in the path goes offline (detected via KEEPALIVE timeout)
- A node in the path is flagged by the reputation system
- The node's advertisement expires
- The client explicitly requests a new path

### Path isolation between applications

Different applications or streams with different privacy requirements SHOULD use different paths, even within the same time window. High-sensitivity connections MUST NOT share a path with low-sensitivity connections.

---

## 7. Guard Node Concept (Optional Extension)

Tor uses a "guard node" concept where a small set of trusted entry nodes is maintained over time, reducing the probability that the entry node is adversary-controlled.

BGP-X supports an optional guard mode:

- The client selects a small set of trusted entry nodes (default: 3)
- Entry nodes are chosen from this set for all paths
- The guard set is refreshed every 30 days or when a guard goes offline

Guard mode MUST be explicitly enabled. It is off by default because guard nodes learn more about a client's activity over time than random entry selection would allow.

---

## 8. Path Quality Monitoring

Clients SHOULD monitor path quality during use:

- Measure round-trip latency via KEEPALIVE timing
- Track packet loss (inferred from sequence number gaps)
- Track throughput

If path quality falls below acceptable thresholds:

- Attempt to rebuild a new path
- Report path quality data (without identifying information) to the reputation system

---

## 9. Path Construction Timing

Path construction is not instantaneous. Clients SHOULD pre-build paths before they are needed to avoid latency at connection time.

Recommended behavior:

- Maintain 2–3 pre-built paths ready for use
- Begin rebuilding a path as soon as the previous one is used
- For latency-sensitive applications, accept a pre-built path from the pool immediately rather than waiting for a fresh construction

---

## 10. Bootstrap Path

When a client first joins the network with no DHT knowledge, it needs a bootstrap path to query the DHT.

Bootstrap sequence:

1. Connect to a well-known bootstrap node (hardcoded in client distribution)
2. Send DHT_FIND_NODE to discover nearby nodes
3. Use the discovered nodes to construct a proper BGP-X path
4. Re-route DHT queries through the constructed path

Bootstrap nodes MUST NOT be used as relay nodes in constructed paths. They serve only to provide initial DHT access.
