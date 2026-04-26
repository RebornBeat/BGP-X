# BGP-X Reputation System Specification

**Version**: 0.1.0-draft

---

## 1. Overview and Design Principles

The BGP-X reputation system is a dual-layer system:

**Layer 1 (Primary): Local Reputation** — based entirely on this client's own observations. Authoritative. Never shared in identifiable form.

**Layer 2 (Advisory): Global Reputation** — aggregated observations from other nodes and clients. Advisory only. Does NOT automatically affect path selection. Opt-in enforcement.

### Non-Negotiable Rules

- Path composition NEVER shared
- Per-hop timing NEVER shared in identifiable form
- path_id NEVER shared
- Session identifiers NEVER shared
- Client IP addresses NEVER shared
- Traffic volume per path NEVER shared
- Pool query history NEVER shared
- Cross-domain traversal details NEVER shared
- Global blacklists are advisory by default — no automatic exclusion unless explicitly enabled

---

## 2. Reputation Score

Each node in the client's node database has a `reputation_score` in range [0.0, 100.0].

Initial score for new, unseen nodes: **60.0** (neutral)

Blacklist threshold: below **15.0** after at least 10 observations.

---

## 3. Local Reputation Events

### Positive Events

| Event | Score Change | Description |
|---|---|---|
| RELAY_SUCCESS | +0.5 | Packet successfully relayed through this node |
| KEEPALIVE_SUCCESS | +0.2 | Node responded to KEEPALIVE within expected latency |
| HANDSHAKE_SUCCESS | +1.0 | Handshake completed successfully |
| LATENCY_WITHIN_ADVERTISED | +0.3 | Measured latency ≤ 150% of advertised |
| BANDWIDTH_WITHIN_ADVERTISED | +0.3 | Measured throughput ≥ 50% of advertised |
| UPTIME_CONSISTENT | +0.5 | Node available when advertisement indicates it should be |
| GEO_PLAUSIBLE | +0.3 | RTT consistent with claimed region |
| CLEAN_WITHDRAWAL | 0 | Node properly published NODE_WITHDRAW before going offline |
| DOMAIN_BRIDGE_SUCCESS | +0.5 | Successfully bridged traffic between two routing domains |
| MESH_ISLAND_STABLE | +0.3 | Mesh island consistently reachable via this bridge node |
| BRIDGE_LOW_LATENCY | +0.3 | Bridge transition latency within advertised range |

### Negative Events

| Event | Score Change | Description |
|---|---|---|
| RELAY_FAILURE | -2.0 | Packet sent through node not forwarded (timeout) |
| KEEPALIVE_TIMEOUT | -3.0 | Node didn't respond to KEEPALIVE within 90 seconds |
| HANDSHAKE_FAILURE | -5.0 | Handshake failed (timeout or invalid response) |
| LATENCY_EXCEEDED | -1.0 | Measured latency > 200% of advertised |
| ADVERTISEMENT_INCONSISTENCY | -5.0 | ASN or country inconsistent with observed routing |
| PROTOCOL_VIOLATION | -10.0 | Node sent protocol-invalid message |
| SUSPICIOUS_BEHAVIOR | -15.0 | Behavior pattern matches known attack signatures |
| GEO_SUSPICIOUS | -1.5 | RTT 1.5-2× expected for claimed region |
| GEO_IMPLAUSIBLE | -3.0 | RTT 2-3× expected for claimed region |
| PROTOCOL_VIOLATION_GEO | -10.0 | ≥3 GEO_IMPLAUSIBLE events within 7 days |
| ECH_CAPABLE_BUT_FAILING | -2.0 | Node claims ECH capability but consistently fails ECH negotiation |
| WITHDRAWAL_WITHOUT_NOTICE | -1.0 | Node went offline without publishing NODE_WITHDRAW |
| DOMAIN_BRIDGE_FAILURE | -3.0 | Domain bridge failed (timeout; couldn't reach other domain) |
| MESH_ISLAND_UNREACHABLE | -2.0 | Mesh island became unreachable via this bridge node |
| BRIDGE_LATENCY_EXCEEDED | -1.0 | Bridge transition latency > 200% of advertised value |
| BRIDGE_CLAIMED_AVAILABLE_OFFLINE | -4.0 | Node advertised bridge as available but couldn't deliver |
| FAKE_DOMAIN_ADVERTISEMENT | -15.0 | Node claimed domain endpoint it cannot actually reach |

### Score Decay

Scores decay toward neutral (60.0) over time:

```
score_t = 60.0 + (score_0 - 60.0) × exp(-λ × t)
λ = 0.1 / 3600  # decay per second
```

This ensures historically excellent nodes that deteriorate lose their high score, and recovering nodes are not permanently penalized.

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

When a bridge node has poor reliability for its mesh domain but good reliability for clearnet relaying, the per-domain event record helps clients identify which role is problematic. Per-domain adjustment is used only for bridge node selection scoring, not for overall reputation score computation.

---

## 5. Blacklisting

### Automatic Blacklist Triggers

1. `reputation_score < 15.0` AND `observation_count >= 10`
2. `SUSPICIOUS_BEHAVIOR` event recorded (immediate, regardless of score)
3. Cryptographic violation detected (active MITM attempt, forged authentication tags)
4. Persistent `FAKE_DOMAIN_ADVERTISEMENT` events (≥3 occurrences in 7 days): permanent blacklist
5. Persistent `BRIDGE_CLAIMED_AVAILABLE_OFFLINE` (≥5 occurrences): temporary blacklist (30 days)
6. Node in signed global blacklist with 3+ independent reporters AND local observations confirm (if enforce_global_blacklist = true)

### Blacklist Effects

- Node excluded from all path construction
- Node's advertisement not propagated by this client's DHT node
- May publish blacklist record to DHT (see Section 6)

### Local Blacklist Expiry

| Type | Default Duration | Behavior |
|---|---|---|
| Reputation-based | 30 days | Re-evaluated at expiry |
| Manual operator | Configurable | Cleared via Control API |
| Cryptographic violation | Permanent | Never re-evaluated |
| Fake domain advertisement | Permanent | Never re-evaluated |
| Community blacklist (opt-in) | Per record (max 30 days) | Re-evaluated at expiry |

---

## 6. Global Reputation (Advisory Layer)

### Global Record Format

```json
{
  "version": 1,
  "reporter_id": "<reporting NodeID>",
  "subject_id": "<subject NodeID>",
  "score": 72.5,
  "observation_count": 142,
  "observation_period_days": 30,
  "events_summary": {
    "relay_success": 1240,
    "keepalive_timeout": 3,
    "handshake_failure": 0,
    "geo_suspicious": 0,
    "domain_bridge_failure": 1
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature of reporter_id>"
}
```

### Advisory-Only Default

By default, global reputation records are:
- Fetched and stored
- Displayed in monitoring tools
- Used to increase local observation frequency on suspicious nodes
- NOT used to automatically exclude nodes from paths

### Opt-In Global Enforcement

```toml
[reputation]
enforce_global_blacklist = true  # Default: false
```

When enabled: global blacklist records from 3+ independent reporters with local observation confirmation trigger local blacklisting.

### Global Aggregation Policy

- Weight each record by reporter's own local reputation score
- Apply skepticism discount: global records weighted at 30% maximum of final score
- Ignore reports from reporters with reputation < 50
- Local observations (70% weight) always dominate
- Local observations always win in conflicts

---

## 7. Global Blacklist Records

Signed records published to DHT to warn others:

```json
{
  "version": 1,
  "record_type": "blacklist",
  "subject_id": "<NodeID>",
  "reason": "active_mitm",
  "evidence_hash": "<BLAKE3 hash of supporting evidence>",
  "duration_hours": 720,
  "reporter_id": "<NodeID of reporter>",
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature>"
}
```

### Reason Codes

| Code | Description |
|---|---|
| `keepalive_failure` | Consistently fails to respond to KeepalIves |
| `handshake_failure` | Consistently fails handshakes |
| `active_mitm` | Evidence of active traffic manipulation |
| `advertisement_fraud` | ASN/country/domain misrepresentation |
| `protocol_violation` | Systematic protocol violations |
| `exit_policy_violation` | Gateway violating signed exit policy |
| `geo_implausibility` | Persistent geographic plausibility failure |
| `fake_domain_bridge` | Claims bridge capability it cannot deliver |
| `mesh_island_fraud` | False mesh island membership claims |

---

## 8. Pool-Related Reputation

Pool curators with many blacklisted members accumulate reputation penalties:

| Condition | Score Change |
|---|---|
| >20% of pool members blacklisted | Curator reputation -5.0 per occurrence |
| >50% of pool members blacklisted | Curator reputation -10.0 per occurrence |
| Domain-bridge pool with >50% unavailable bridges | Curator reputation -5.0 per occurrence |

---

## 9. Reputation Persistence

Location: `data_dir/reputation.db`. Encrypted with key derived from client's long-term identity. Atomic writes (temp file + rename).

On startup: load database, apply decay based on elapsed time since last run, remove expired entries.

On shutdown: flush reputation database to disk.

---

## 10. What Is NEVER Shared

All prohibited items from Section 1, plus:
- Which routing domains a specific session traversed
- Cross-domain path composition
- Mesh island identifiers associated with specific sessions
- Per-connection domain bridge transition records
- Domain-specific timing correlation data

---

## 11. Metrics

| Metric | Description |
|---|---|
| nodes_by_reputation_bucket | Count per range (0-20, 20-40, 40-60, 60-80, 80-100) |
| blacklisted_node_count | Total currently blacklisted |
| positive_events_per_hour | Rate of positive reputation events |
| negative_events_per_hour | Rate of negative reputation events |
| geo_suspicious_events_per_hour | Geographic plausibility events |
| domain_bridge_failure_events_per_hour | Domain bridge failure events |
| global_reports_received | Count of global records received |
| global_reports_accepted | Count accepted after validation |
| pool_curator_reputation_by_pool | Curator reputation per pool |

No node identifiers in published metrics.
