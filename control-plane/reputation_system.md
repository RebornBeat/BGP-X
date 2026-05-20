# BGP-X Reputation System Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X reputation system — the mechanism by which node reliability, honesty, and performance are tracked over time and used to influence path selection.

---

## 1. Overview and Design Principles

The BGP-X reputation system is a **dual-layer system**:

**Layer 1 (Primary): Local Reputation**
- Based entirely on this client's own observations
- Authoritative — local observations are always trusted over global signals
- Applied directly to path selection
- Never shared in identifiable form

**Layer 2 (Advisory): Global Reputation**
- Aggregated observations from other nodes and clients
- Advisory only — does NOT automatically affect path selection
- Opt-in enforcement available (`enforce_global_blacklist = true`)
- Shared as aggregate signals via DHT

### 1.1 Purpose

The BGP-X reputation system serves two purposes:

1. **Path quality**: Prefer nodes that are reliable, fast, and available
2. **Security**: Detect and penalize nodes exhibiting malicious or unreliable behavior

The reputation system is **client-local by default** — each client maintains its own reputation scores based on its own observations. A distributed reputation aggregation layer provides advisory signals.

Reputation scores are used by the routing algorithm as one factor in node scoring (15% weight by default). They do not override mandatory constraints — a node with a perfect reputation score that fails a diversity constraint is still excluded.

### 1.2 Non-Negotiable Rules

The following are absolute constraints on the reputation system. These data items are NEVER shared in any reputation record (local or global):

- **Path composition** — which nodes appeared in the same path, in what order, or at what time
- **Per-hop timing** — individual hop latency in identifiable form
- **path_id values**
- **Session identifiers**
- **Client IP addresses**
- **Traffic volume per path**
- **Pool query history**
- **Cross-domain traversal details per session**
- **Mesh island identifiers associated with specific sessions**
- **Per-connection domain bridge transition records**
- **Domain-specific timing correlation data**

### 1.3 Global Blacklists are Advisory by Default

No automatic exclusion from paths occurs based on global blacklist records unless explicitly enabled by the client:

```toml
[reputation]
enforce_global_blacklist = false   # Default: advisory only
```

When disabled (default): global blacklist records trigger enhanced monitoring but not exclusion.

When enabled: global blacklist records from 3+ independent reporters with local confirmation trigger local blacklisting.

---

## 2. Reputation Score

Each node in the client's node database has a `reputation_score` in range [0.0, 100.0].

### 2.1 Initial Score

New nodes (never used before) start with a neutral score:

```
initial_reputation_score = 60.0
```

This allows new nodes to be used while their track record is established, without treating all unproven nodes as suspect.

### 2.2 Score Bounds

- **Minimum**: 0.0 (hard floor)
- **Maximum**: 100.0 (hard ceiling)
- **Blacklist threshold**: below 15.0 after at least 10 observations

### 2.3 Score Use in Path Selection

Reputation score is one of five components in the node scoring function (15% weight by default):

```
score_total = 0.30 × uptime_score
            + 0.25 × latency_score
            + 0.20 × bandwidth_score
            + 0.15 × reputation_score
            + 0.10 × geo_plausibility_score
```

Reputation does not override mandatory constraints (diversity, signature validity, advertisement validity).

---

## 3. Local Reputation Events

Reputation events are observations that modify a node's score. Events are weighted and applied as increments or decrements to the current score.

### 3.1 Positive Events

| Event | Score Change | Description |
|---|---|---|
| RELAY_SUCCESS | +0.5 | A packet was successfully relayed through this node |
| KEEPALIVE_SUCCESS | +0.2 | Node responded to a KEEPALIVE within expected latency |
| HANDSHAKE_SUCCESS | +1.0 | Handshake completed successfully |
| LATENCY_WITHIN_ADVERTISED | +0.3 | Measured latency is within 150% of advertised latency |
| BANDWIDTH_WITHIN_ADVERTISED | +0.3 | Measured throughput is at least 50% of advertised bandwidth |
| UPTIME_CONSISTENT | +0.5 | Node is available when its advertisement indicates it should be |
| GEO_PLAUSIBLE | +0.3 | RTT consistent with claimed region (if jurisdiction declared) |
| CLEAN_WITHDRAWAL | 0 | Node properly published NODE_WITHDRAW before going offline |
| DOMAIN_BRIDGE_SUCCESS | +0.5 | Successfully bridged traffic between two routing domains |
| MESH_ISLAND_STABLE | +0.3 | Mesh island consistently reachable via this bridge node |
| BRIDGE_LOW_LATENCY | +0.3 | Bridge transition latency within advertised range |

### 3.2 Negative Events

| Event | Score Change | Description |
|---|---|---|
| RELAY_FAILURE | -2.0 | A packet sent through this node was not forwarded (timeout) |
| KEEPALIVE_TIMEOUT | -3.0 | Node did not respond to KEEPALIVE within 90 seconds |
| HANDSHAKE_FAILURE | -5.0 | Handshake failed (timeout or invalid response) |
| LATENCY_EXCEEDED | -1.0 | Measured latency exceeds 200% of advertised latency |
| ADVERTISEMENT_INCONSISTENCY | -5.0 | ASN or country in advertisement inconsistent with observed routing |
| PROTOCOL_VIOLATION | -10.0 | Node sent a protocol-invalid message |
| SUSPICIOUS_BEHAVIOR | -15.0 | Behavior pattern matches known attack signatures |
| GEO_SUSPICIOUS | -1.5 | RTT is 1.5-2× expected for claimed region |
| GEO_IMPLAUSIBLE | -3.0 | RTT is 2-3× expected for claimed region |
| PROTOCOL_VIOLATION_GEO | -10.0 | Three or more GEO_IMPLAUSIBLE events within 7 days |
| ECH_CAPABLE_BUT_FAILING | -2.0 | Node claims ECH capability but consistently fails ECH negotiation |
| WITHDRAWAL_WITHOUT_NOTICE | -1.0 | Node went offline without publishing NODE_WITHDRAW |
| DOMAIN_BRIDGE_FAILURE | -3.0 | Domain bridge failed (timeout; couldn't reach other domain) |
| MESH_ISLAND_UNREACHABLE | -2.0 | Mesh island became unreachable via this bridge node |
| BRIDGE_LATENCY_EXCEEDED | -1.0 | Bridge transition latency exceeds 200% of advertised value |
| BRIDGE_CLAIMED_AVAILABLE_OFFLINE | -4.0 | Node advertised bridge as available but couldn't deliver |
| FAKE_DOMAIN_ADVERTISEMENT | -15.0 | Node claimed domain endpoint it cannot actually reach |

### 3.3 Score Decay

Reputation scores decay toward the neutral value (60.0) over time at a rate of 0.1 points per hour. This ensures:

- A node that was excellent years ago but has deteriorated does not retain its high score indefinitely
- A node that recovered from a bad period is not permanently penalized

Decay formula:

```
score_at_time_t = 60.0 + (score_at_time_0 - 60.0) × exp(-λ × t)
λ = 0.1 / 3600  # decay constant (per second)
```

Implementation note: decay is applied lazily on access or during periodic maintenance, not continuously.

---

## 4. Per-Domain Reputation Tracking

Domain bridge nodes may serve multiple routing domains. Reputation events are tagged with the relevant domain:

```rust
struct ReputationEvent {
    event_type:   ReputationEventType,
    domain:       Option<DomainId>,    // Which domain this event relates to
    timestamp:    Timestamp,
    score_change: f32,
}
```

When a bridge node has poor reliability for its mesh domain but good reliability for clearnet relaying, the per-domain event record helps clients identify which role is problematic.

**Per-domain adjustment** is used only for bridge node selection scoring, not for overall reputation score computation. The overall node reputation score aggregates across all domains, but bridge selection considers domain-specific reliability.

Example: A bridge node with:
- Clearnet relay reputation: 85.0
- Mesh bridge reputation: 45.0 (frequent MESH_ISLAND_UNREACHABLE events)

This node may be selected as a clearnet relay but deprioritized for mesh bridge paths.

---

## 5. Blacklisting

### 5.1 Automatic Blacklist Triggers

A node is automatically blacklisted when:

1. `reputation_score < 15.0` AND `observation_count >= 10`
2. `SUSPICIOUS_BEHAVIOR` event recorded (immediate, regardless of score)
3. Cryptographic violation detected (active MITM attempt, forged authentication tags)
4. Persistent `FAKE_DOMAIN_ADVERTISEMENT` events (≥3 occurrences in 7 days) — permanent blacklist
5. Persistent `BRIDGE_CLAIMED_AVAILABLE_OFFLINE` (≥5 occurrences) — 30-day blacklist
6. Node is included in a signed global blacklist record with 3+ independent reporters AND local observations confirm the report (if `enforce_global_blacklist = true`)

### 5.2 Blacklist Effects

When a node is blacklisted:

- The node is excluded from all path construction
- The node's advertisement is not propagated by this client's DHT node
- A blacklist record MAY be published to the DHT (see Section 7)
- For domain bridge nodes: all bridge pair records are marked unavailable

### 5.3 Local Blacklist Expiry

| Blacklist Type | Default Duration | Behavior |
|---|---|---|
| Reputation-based | 30 days | Re-evaluated at expiry; re-admitted if behavior improves |
| Manual operator blacklist | Configurable | Cleared via Control API |
| Cryptographic violation | Permanent (no expiry) | Never re-evaluated |
| Fake domain advertisement | Permanent | Never re-evaluated |
| Community blacklist (opt-in) | Per record (max 30 days) | Re-evaluated at expiry |

Configuration:

```toml
[reputation]
blacklist_default_duration_days = 30
blacklist_auto_review = true     # Re-evaluate on expiry
```

### 5.4 Blacklist Management via Control API

Operators can manage the local blacklist:

```bash
# Add a node to the blacklist
bgpx-cli nodes blacklist <node_id> --reason "active_mitm"

# Remove a node from the blacklist (if not permanent)
bgpx-cli nodes unblacklist <node_id>

# View blacklist status
bgpx-cli nodes reputation <node_id>
```

---

## 6. Global Reputation (Advisory Layer)

### 6.1 Global Record Format

Nodes publish signed reputation records to the DHT:

```json
{
  "version": 1,
  "reporter_id": "<NodeID of reporting node>",
  "subject_id": "<NodeID of node being rated>",
  "score": 75.5,
  "observation_count": 142,
  "observation_period_days": 30,
  "events_summary": {
    "relay_success": 1240,
    "keepalive_timeout": 3,
    "handshake_failure": 0,
    "geo_suspicious": 0,
    "geo_implausible": 1,
    "domain_bridge_failure": 1
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature of reporter_id>"
}
```

### 6.2 Advisory-Only Default

By default, global reputation records are:

- Fetched and stored
- Displayed in monitoring tools
- Used to increase local observation frequency on suspicious nodes
- NOT used to automatically exclude nodes from paths

Global records are advisory signals only.

### 6.3 Opt-In Global Enforcement

```toml
[reputation]
enforce_global_blacklist = true   # Default: false
```

When enabled: global blacklist records from 3+ independent reporters with local observation confirmation trigger local blacklisting.

### 6.4 Global Aggregation Policy

When consuming global reputation:

- Weight each record by the reporter's own local reputation score
- Apply a skepticism discount: global records weighted at 30% maximum of final score
- Ignore reports from reporters with reputation < 50 (to prevent Sybil reputation attacks)
- Cap the influence of distributed reputation at 30% of the final score (local observations dominate at 70%)
- Local observations always win in conflicts — if global says "bad" but local says "good", local wins

### 6.5 Sybil Resistance for Global Reputation

A Sybil attacker who creates many fake reporter nodes cannot effectively manipulate reputation because:

- Each reporter's influence is weighted by their own reputation
- New nodes start at neutral reputation (60.0) and earn influence slowly
- The local observation component (70% weight) cannot be manipulated by remote reports
- Reporters with reputation < 50 are ignored entirely

---

## 7. Global Blacklist Records

Signed records published to the DHT to warn others about malicious behavior:

```json
{
  "version": 1,
  "record_type": "blacklist",
  "subject_id": "<NodeID of malicious node>",
  "reason": "active_mitm",
  "evidence_hash": "<BLAKE3 hash of supporting evidence>",
  "duration_hours": 720,
  "reporter_id": "<NodeID of reporting node>",
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature of canonical JSON>"
}
```

### 7.1 Reason Codes

| Code | Description |
|---|---|
| `keepalive_failure` | Node consistently fails to respond to KEEPALIVEs |
| `handshake_failure` | Node consistently fails handshakes |
| `active_mitm` | Evidence of active traffic manipulation |
| `advertisement_fraud` | ASN/country/domain misrepresentation |
| `protocol_violation` | Systematic protocol violations |
| `exit_policy_violation` | Gateway violating its signed exit policy |
| `geo_implausibility` | Persistent geographic plausibility failure |
| `fake_domain_bridge` | Claims bridge capability it cannot deliver |
| `mesh_island_fraud` | False mesh island membership claims |

### 7.2 Blacklist Record Acceptance Policy

Clients accept global blacklist records only if:

- The reporter's local reputation score is >= 70.0
- The record is from the past 72 hours
- The signature is valid
- The reason code is known

**Default action**: MONITOR (log the record, increase local observation frequency on that node)

**With `enforce_global_blacklist = true`**: If 3+ independent reporters have valid blacklist records for the same subject with compatible reasons → trigger local blacklist

**Single blacklist record**: Does NOT trigger automatic blacklisting. It triggers enhanced monitoring (more frequent KEEPALIVE checks, mandatory handshake verification).

### 7.3 Evidence Hash

The `evidence_hash` field contains a BLAKE3 hash of supporting evidence (packet captures, timing logs, etc.). The evidence itself is NOT published to the DHT to protect privacy. The hash allows the reporter to prove evidence exists if challenged.

---

## 8. Pool-Related Reputation

### 8.1 Pool Curator Reputation

Pool curators that publish pools with many blacklisted members accumulate reputation penalties:

| Condition | Score Change |
|---|---|
| >20% of pool members blacklisted | Curator reputation -5.0 per occurrence |
| >50% of pool members blacklisted | Curator reputation -10.0 per occurrence |
| Domain-bridge pool with >50% unavailable bridges | Curator reputation -5.0 per occurrence |

Note: Curator reputation is tracked separately from node reputation. It is applied to the curator's `operator_id`, not to any node the curator operates.

### 8.2 After Pool Curator Key Rotation

When a pool curator rotates their signing key:

- Nodes signed under the old curator key have a 24-hour grace period for re-verification
- Reputation continuity is maintained through rotation (reputation data is tied to NodeID, not pool membership signature)
- Curators should re-sign all pool members promptly after rotation

### 8.3 Pool Curator Compromise

If a pool curator's private key is compromised:

- The curator publishes an emergency key rotation record (dual-signed if possible)
- Clients re-verify all pool members under the new key
- Curator reputation may be penalized depending on the severity of the compromise

---

## 9. Domain Bridge Reputation

### 9.1 Bridge-Specific Metrics

Domain bridge nodes are scored independently for each bridge pair they serve:

| Metric | Description |
|---|---|
| `bridge_latency_ms` | Transition latency between domains |
| `bridge_success_rate` | Percentage of successful cross-domain deliveries |
| `bridge_availability` | Uptime of the bridge capability |

### 9.2 Bridge Reliability Scoring

Bridge nodes with high `DOMAIN_BRIDGE_FAILURE` or `MESH_ISLAND_UNREACHABLE` events receive lower scores for bridge selection, even if their overall reputation is high for clearnet relaying.

### 9.3 Fake Domain Advertisement Detection

If a node advertises bridge capability for a domain it cannot actually reach:

- `FAKE_DOMAIN_ADVERTISEMENT` event: -15.0 penalty
- Three occurrences within 7 days: permanent blacklist
- This protects clients from nodes claiming bridge capability they cannot deliver

---

## 10. What Is NEVER Shared in Reputation Data

The following items are explicitly prohibited from any reputation record (local or global). They are never transmitted, logged, or stored in any shared format:

| Prohibited Item | Reason |
|---|---|
| Path composition | Reveals which nodes appear together in paths |
| Per-hop timing (identifiable) | Enables timing correlation attacks |
| path_id values | Session identifier proxy |
| Session identifiers | Directly identifies communication sessions |
| Client IP addresses | Identifies users directly |
| Traffic volume per path | Reveals behavior patterns |
| Pool query history | Reveals trust relationships |
| Cross-domain traversal details | Reveals routing patterns |
| Mesh island identifiers (session-associated) | Links users to specific communities |
| Per-connection domain bridge records | Reveals cross-domain behavior |
| Domain-specific timing correlation data | Enables correlation across domains |

Only aggregate counts (e.g., "relay_success: 1240") are shared in global records — never individual event instances with identifying context.

---

## 11. Observation Collection

### 11.1 What Is Observed

The reputation system collects observations only from the client's own use of nodes:

- Whether KEEPALIVE responses arrive on time
- Whether handshakes complete successfully
- Path latency measurements (before and after each relay hop, inferred from timing)
- Whether the path delivered data successfully (session completion)
- Domain bridge transition success/failure
- Mesh island reachability

### 11.2 What Is NOT Observed

- Content of relayed traffic
- Session identifiers from other clients
- DHT query behavior from other clients
- Traffic patterns of other users

### 11.3 Observation Attribution

Observations are attributed to the node that caused the event, as determined by the client's knowledge of path construction:

- A timeout at hop 2 attributes KEEPALIVE_TIMEOUT to the node at hop 2
- A successful relay attributes RELAY_SUCCESS to all nodes in the path
- A handshake failure attributes HANDSHAKE_FAILURE to the specific node the handshake was attempted with
- A domain bridge failure attributes DOMAIN_BRIDGE_FAILURE to the bridge node

---

## 12. Reputation Persistence

The client's reputation database MUST be persisted to disk across restarts.

### 12.1 Storage

- **Location**: `data_dir/reputation.db`
- **Format**: implementation choice (JSON, SQLite, or binary serialization)
- **Encryption**: encrypted with a key derived from the client's long-term identity key
- **Write**: atomic (temp file + rename) to prevent corruption

### 12.2 Startup Procedure

On startup:

1. Load reputation database from disk
2. Apply decay based on time elapsed since last run
3. Remove nodes whose advertisements have expired AND last-seen > 30 days ago
4. Initialize per-domain tracking structures for bridge nodes

### 12.3 Shutdown Procedure

On shutdown:

1. Flush reputation database to disk before exit
2. Ensure all pending events are recorded
3. Sync any pending global reputation cache updates

---

## 13. Reputation System Metrics

| Metric | Description |
|---|---|---|
| `nodes_by_reputation_bucket` | Count of nodes in each reputation range (0-20, 20-40, 40-60, 60-80, 80-100) |
| `blacklisted_node_count` | Total currently blacklisted nodes |
| `positive_events_per_hour` | Rate of positive reputation events |
| `negative_events_per_hour` | Rate of negative reputation events |
| `geo_suspicious_events_per_hour` | Geographic plausibility warning events |
| `domain_bridge_failure_events_per_hour` | Domain bridge failure events |
| `global_reports_received` | Count of global reputation records received |
| `global_reports_accepted` | Count accepted after validation |
| `pool_curator_reputation_by_pool` | Curator reputation per pool |

No node identifiers are included in published metrics. All metrics are aggregate counts only.

---

## 14. Implementation Requirements

### 14.1 MUST

- Maintain per-node reputation scores in persistent storage
- Apply score decay based on elapsed time
- Implement all positive and negative event types defined in Section 3
- Implement automatic blacklisting for triggers defined in Section 5.1
- Respect `enforce_global_blacklist` configuration
- Verify signatures on global reputation records before use
- Weight local observations over global signals (70%/30% max)
- Track per-domain events for bridge nodes
- Implement geo plausibility scoring for nodes that declare jurisdiction
- Clear blacklist entries on expiry (if not permanent)
- Provide Control API for blacklist management
- NEVER share prohibited items listed in Section 11

### 14.2 MUST NOT

- Share any prohibited items listed in Section 11 in any record format
- Allow global reputation to override local observations in conflicts
- Accept unsigned global reputation records
- Accept global records from reporters with reputation < 50
- Blacklist based on a single global blacklist record
- Store evidence (only evidence_hash) in global blacklist records
- Maintain separate per-domain overall reputation scores (only per-domain event tracking)

### 14.3 SHOULD

- Publish global reputation records for nodes with significant observation history
- Publish blacklist records for confirmed malicious behavior
- Implement skepticism discount for global reputation (30% max weight)
- Use per-domain event records for bridge node selection
- Implement pool curator reputation tracking
- Monitor for fake domain advertisements
- Rotate pool curator keys promptly after compromise

---

## 15. Cross-Domain Path Construction Integration

When constructing cross-domain paths, the reputation system provides:

1. **Overall node reputation** — general reliability score
2. **Per-domain event history** — domain-specific reliability for bridge nodes
3. **Bridge latency measurements** — historical transition latency

The path construction algorithm uses:

```
bridge_score = overall_reputation × 0.5
             + domain_specific_reliability × 0.3
             + bridge_latency_score × 0.2
```

This ensures bridges that are excellent clearnet relays but unreliable mesh bridges are deprioritized for mesh-bound traffic.

---

## 16. Session Re-Handshake Integration

When a session approaches 24 hours of age, the client initiates a re-handshake. If re-handshake fails:

- `HANDSHAKE_FAILURE` event is recorded against the node
- Path is marked for rebuild
- Alternative nodes are selected

This feeds back into reputation: nodes that cannot complete re-handshakes accumulate penalties and may be blacklisted.
