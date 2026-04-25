# BGP-X Reputation System Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X reputation system — the mechanism by which node reliability, honesty, and performance are tracked over time and used to influence path selection.

---

## 1. Overview

The BGP-X reputation system serves two purposes:

1. **Path quality**: Prefer nodes that are reliable, fast, and available
2. **Security**: Detect and penalize nodes exhibiting malicious or unreliable behavior

The reputation system is **client-local by default** — each client maintains its own reputation scores based on its own observations. A distributed reputation aggregation layer is a planned extension.

Reputation scores are used by the routing algorithm as one factor in node scoring (15% weight by default). They do not override mandatory constraints — a node with a perfect reputation score that fails a diversity constraint is still excluded.

---

## 2. Reputation Score

Each node in the client's node database has a `reputation_score` in the range [0.0, 100.0].

### 2.1 Initial score

New nodes (never used before) start with a neutral score:

```
initial_reputation_score = 60.0
```

This allows new nodes to be used while their track record is established, without treating all unproven nodes as suspect.

### 2.2 Score bounds

- Minimum: 0.0 (blacklist threshold below this)
- Maximum: 100.0
- Blacklist threshold: below 15.0 after at least 10 observations

---

## 3. Reputation Events

Reputation events are observations that modify a node's score. Events are weighted and applied as increments or decrements to the current score.

### 3.1 Positive events

| Event | Score Change | Description |
|---|---|---|
| RELAY_SUCCESS | +0.5 | A packet was successfully relayed through this node |
| KEEPALIVE_SUCCESS | +0.2 | Node responded to a KEEPALIVE within expected latency |
| HANDSHAKE_SUCCESS | +1.0 | Handshake completed successfully |
| LATENCY_WITHIN_ADVERTISED | +0.3 | Measured latency is within 150% of advertised latency |
| BANDWIDTH_WITHIN_ADVERTISED | +0.3 | Measured throughput is within 50% of advertised bandwidth |
| UPTIME_CONSISTENT | +0.5 | Node is available when its advertisement indicates it should be |

### 3.2 Negative events

| Event | Score Change | Description |
|---|---|---|
| RELAY_FAILURE | -2.0 | A packet sent through this node was not forwarded (timeout) |
| KEEPALIVE_TIMEOUT | -3.0 | Node did not respond to KEEPALIVE within 90 seconds |
| HANDSHAKE_FAILURE | -5.0 | Handshake failed (timeout or invalid response) |
| LATENCY_EXCEEDED | -1.0 | Measured latency exceeds 200% of advertised latency |
| ADVERTISEMENT_INCONSISTENCY | -5.0 | ASN or country in advertisement inconsistent with observed routing |
| PROTOCOL_VIOLATION | -10.0 | Node sent a protocol-invalid message |
| SUSPICIOUS_BEHAVIOR | -15.0 | Behavior pattern matches known attack signatures |

### 3.3 Score decay

Reputation scores decay toward the neutral value (60.0) over time at a rate of 0.1 points per hour. This ensures:

- A node that was excellent years ago but has deteriorated does not retain its high score indefinitely
- A node that recovered from a bad period is not permanently penalized

```
score_at_time_t = 60.0 + (score_at_time_0 - 60.0) × exp(-λ × t)
λ = 0.1 / 3600  # decay constant (per second)
```

---

## 4. Blacklisting

A node is blacklisted when:

1. Its reputation score falls below **15.0** AND it has at least **10 observations**, OR
2. A SUSPICIOUS_BEHAVIOR event is recorded (immediate blacklist regardless of score), OR
3. A cryptographic violation is observed (e.g., packet authentication failure that indicates active manipulation), OR
4. The node is included in a signed blacklist record published to the DHT

### 4.1 Blacklist effects

- The node is excluded from all path construction
- The node's advertisement is not propagated by this client's DHT node
- A blacklist record MAY be published to the DHT (see Section 6)

### 4.2 Blacklist duration

- Reputation-based blacklists: 30 days minimum, then eligible for re-evaluation if fresh advertisement is published
- Cryptographic violation blacklists: permanent (never re-evaluated)
- Signed community blacklists: duration specified in the blacklist record

---

## 5. Observation Collection

### 5.1 What is observed

The reputation system collects observations only from the client's own use of nodes. Observations are:

- Whether KEEPALIVE responses arrive on time
- Whether handshakes complete successfully
- Path latency measurements (before and after each relay hop, inferred from timing)
- Whether the path delivered data successfully (session completion)

### 5.2 What is NOT observed

- Content of relayed traffic
- Session identifiers from other clients
- DHT query behavior

### 5.3 Observation attribution

Observations are attributed to the node that caused the event, as determined by the client's knowledge of path construction:

- A timeout at hop 2 attributes the KEEPALIVE_TIMEOUT to the node at hop 2
- A successful relay attributes RELAY_SUCCESS to all nodes in the path
- A handshake failure attributes HANDSHAKE_FAILURE to the specific node the handshake was attempted with

---

## 6. Distributed Reputation (Planned Extension)

The base reputation system is fully local. A planned extension adds distributed reputation aggregation.

### 6.1 Reputation records

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
    "handshake_failure": 0
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature of reporter_id>"
}
```

### 6.2 Aggregation policy

When consuming distributed reputation data:

- Weight each record by the reporter's own reputation score (circular reference handled by bootstrapping from local observations)
- Apply a skepticism discount (distributed reports are trusted less than local observations)
- Ignore reports from nodes with reputation < 50 (to prevent Sybil reputation attacks)
- Cap the influence of distributed reputation at 30% of the final score (local observations dominate)

### 6.3 Sybil resistance for distributed reputation

A Sybil attacker who creates many fake reporter nodes cannot effectively manipulate reputation because:

- Each reporter's influence is weighted by their own reputation
- New nodes start at neutral reputation (60.0) and earn influence slowly
- The local observation component (70% weight) cannot be manipulated by remote reports

---

## 7. Blacklist Records (DHT-Published)

Signed blacklist records allow one node to warn others about malicious behavior.

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
  "signature": "<Ed25519 signature of reporter_id>"
}
```

#### Reason codes

| Code | Description |
|---|---|
| `keepalive_failure` | Node consistently fails to respond to KeepalIves |
| `handshake_failure` | Node consistently fails handshakes |
| `active_mitm` | Evidence of active traffic manipulation |
| `advertisement_fraud` | ASN/country misrepresentation |
| `protocol_violation` | Systematic protocol violations |
| `exit_policy_violation` | Gateway violating its signed exit policy |

### 7.1 Blacklist record acceptance policy

Clients accept blacklist records only if:

- The reporter's local reputation score is >= 70.0
- The record is from the past 72 hours
- The signature is valid
- The reason code is known
- At least 3 independent reporters have published blacklist records for the same subject with compatible reasons (Sybil resistance)

A single blacklist record does not trigger automatic blacklisting. It triggers enhanced monitoring (more frequent KEEPALIVE checks, mandatory handshake verification).

---

## 8. Reputation Persistence

The client's reputation database MUST be persisted to disk across restarts.

Storage format:
- Encrypted with a key derived from the client's long-term identity
- JSON or binary serialization (implementation choice)
- Backed up before modification (atomic write via temp file + rename)

On startup:
- Load reputation database
- Decay all scores based on time elapsed since last run
- Remove nodes whose advertisements have expired AND whose last-seen time is > 30 days ago

---

## 9. Reputation System Metrics

| Metric | Description |
|---|---|
| `nodes_by_reputation_bucket` | Count of nodes in each reputation range (0–20, 20–40, 40–60, 60–80, 80–100) |
| `blacklisted_node_count` | Total blacklisted nodes |
| `positive_events_per_hour` | Rate of positive reputation events |
| `negative_events_per_hour` | Rate of negative reputation events |
| `distributed_reports_received` | Count of distributed reputation records received |
| `distributed_reports_accepted` | Count accepted after validation |

No node identifiers are included in published metrics.
