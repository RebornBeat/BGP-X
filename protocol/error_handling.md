# BGP-X Error Handling Specification

**Version**: 0.1.0-draft

---

## 1. Error Handling Philosophy

BGP-X is a privacy system. Error messages are a potential information leak. The error handling design follows these principles:

1.  **Silent drop over informative error**: When a packet cannot be processed (decryption failure, unknown session, invalid format), nodes SHOULD drop the packet silently rather than send an error response. Error responses confirm that a packet was received and processed, which is itself information.

2.  **No error amplification**: Error handling MUST NOT send multiple responses to a single malformed packet. Amplification can be exploited for DoS.

3.  **No session confirmation**: A node MUST NOT confirm whether a given session ID exists. Both "session not found" and "decryption failure" MUST produce identical behavior (silent drop, optional internal log).

4.  **Errors are for established sessions**: ERROR messages (0x10) are only sent within established sessions where the sender is already known. They are not sent in response to unauthenticated packets.

5.  **Fail toward privacy**: When in doubt about whether a packet is valid, drop it silently.

---

## 2. Error Codes

| Code | Name | Description | Response |
|---|---|---|---|
| 0x0000 | ERR_NONE | No error (informational) | — |
| 0x0001 | ERR_PROTOCOL_VERSION | Unsupported protocol version | ERROR or silent drop |
| 0x0002 | ERR_INVALID_HEADER | Malformed common header | Silent drop |
| 0x0003 | ERR_DECRYPTION_FAILURE | Onion layer decryption failed (Authentication tag mismatch) | Silent drop |
| 0x0004 | ERR_UNKNOWN_SESSION | Session ID not found | Silent drop |
| 0x0005 | ERR_REPLAY | Sequence number replay detected | Silent drop |
| 0x0006 | ERR_HANDSHAKE_TIMEOUT | Handshake not completed within timeout | Silent drop |
| 0x0007 | ERR_HANDSHAKE_INVALID | Handshake content failed validation | Silent drop |
| 0x0008 | ERR_SESSION_FULL | Node cannot accept new sessions | ERROR (within session if applicable) |
| 0x0009 | ERR_STREAM_NOT_FOUND | Stream ID does not exist in this session | ERROR |
| 0x000A | ERR_STREAM_ALREADY_OPEN | Stream ID already in use | ERROR |
| 0x000B | ERR_EXIT_POLICY_DENIED | Destination rejected by exit policy | ERROR |
| 0x000C | ERR_DESTINATION_UNREACHABLE | Exit could not reach destination | ERROR |
| 0x000D | ERR_BANDWIDTH_EXCEEDED | Connection exceeds node's bandwidth limit | ERROR |
| 0x000E | ERR_INVALID_ADVERTISEMENT | Node advertisement failed validation | (internal) |
| 0x000F | ERR_DHT_FULL | DHT storage is full | (DHT response) |
| 0x0010 | ERR_UNKNOWN | Unknown error | ERROR |
| 0x0011 | ERR_POOL_NOT_FOUND | Pool ID not found in DHT | (path construction) |
| 0x0012 | ERR_POOL_SIGNATURE_INVALID | Pool signature verification failed | (path construction) |
| 0x0013 | ERR_POOL_INSUFFICIENT_NODES | Pool has too few active nodes | (path construction) |
| 0x0014 | ERR_POOL_ROLE_MISMATCH | Pool has no nodes with required role | (path construction) |
| 0x0015 | ERR_SEGMENT_CONSTRUCTION_FAILED | Segment could not be constructed | (path construction) |
| 0x0016 | ERR_ECH_REQUIRED_NOT_SUPPORTED | ECH required but destination doesn't support it | STREAM_CLOSE |
| 0x0017 | ERR_ECH_NEGOTIATION_FAILED | ECH negotiation failed | STREAM_CLOSE |
| 0x0018 | ERR_NODE_WITHDRAWN | Connection to withdrawn node attempted | (path construction) |
| 0x0019 | ERR_PATH_ID_UNKNOWN | Return packet references unknown path_id | Silent drop |
| 0x001A | ERR_MESH_FRAGMENT_TIMEOUT | Fragmented packet not fully received | Silent drop |
| 0x001B | ERR_TRANSPORT_UNAVAILABLE | Required mesh transport not available | (path construction) |
| 0x001C | ERR_POOL_KEY_ROTATION_INVALID | Key rotation record verification failed | Silent drop |
| 0x001D | ERR_SESSION_REHANDSHAKE_FAILED | 24hr session re-handshake failed | Session close |
| 0x001E | ERR_DOMAIN_NOT_FOUND | Target routing domain not reachable | (path construction) |
| 0x001F | ERR_DOMAIN_BRIDGE_UNAVAILABLE | No active bridge between specified domains | (path construction) |
| 0x0020 | ERR_MESH_ISLAND_UNREACHABLE | Mesh island has no active relay nodes | (path construction) |
| 0x0021 | ERR_NO_CROSS_DOMAIN_PATH | Path construction failed across domain boundary | (path construction) |
| 0x0022 | ERR_DOMAIN_POLICY_DENIED | Domain's routing policy rejected this path segment | STREAM_CLOSE |
| 0x0023 | ERR_DOMAIN_SCOPE_MISMATCH | Pool domain_scope does not match required segment domain | (path construction) |
| 0x0024 | ERR_DOMAIN_BRIDGE_POOL_INVALID | Domain-bridge pool does not match required bridge pair | (path construction) |

---

## 3. Malformed Onion Handling — Explicit Behavior Table

| Failure Condition | Behavior |
|---|---|
| Plaintext < 57 bytes (header too short) | DROP silently; log_internal "Layer too short" |
| Unknown hop_type (not 0x01-0x09) | DROP silently; log_internal "Unknown hop type" |
| Invalid next_hop encoding | DROP silently; log_internal "Invalid next hop" |
| path_id = 0x0000000000000000 | DROP silently; log_internal "Invalid path_id: all zeros" |
| stream_id = 0 | DROP silently; log_internal "Invalid stream_id: zero" |
| Client stream_id is even (should be odd) | Send STREAM_CLOSE RST; log_internal "Stream ID parity violation" |
| Service stream_id is odd (should be even) | Send STREAM_CLOSE RST; log_internal "Stream ID parity violation" |
| Hop type EXIT (0x02-0x04) but node has no exit role | DROP silently; log_internal "Exit hop received at non-exit node" |
| Hop type DELIVERY (0x05) but service not registered | Send STREAM_CLOSE ERR_DESTINATION_UNREACHABLE |
| Hop type DOMAIN_BRIDGE (0x06) but node not bridge-capable | DROP silently; log_internal "DOMAIN_BRIDGE at non-bridge node" |
| Hop type DOMAIN_BRIDGE with unknown domain_type | DROP silently; log_internal "Unknown target domain type" |
| Hop type DOMAIN_BRIDGE with unreachable target domain | DROP silently; client detects via KEEPALIVE timeout |
| Hop type MESH_ENTRY (0x07) but no mesh transport | DROP silently; log_internal "MESH_ENTRY at non-mesh node" |
| Hop type MESH_EXIT (0x08) but no mesh transport | DROP silently; log_internal "MESH_EXIT at non-mesh node" |
| Hop type MESH_RELAY (0x09) but no mesh transport | DROP silently; log_internal "MESH_RELAY at non-mesh node" |
| Fragment: total_fragments = 0 | DROP silently |
| Fragment: fragment_num >= total_fragments | DROP silently |
| Fragment timeout (10 seconds) | Discard fragment buffer; log_internal "Fragment timeout" |
| Fragment reassembly produces packet > 1280 bytes | DROP the reassembled packet; log_internal "Reassembled packet exceeds MTU" |
| DOMAIN_ADVERTISE: invalid bridge pair | DROP silently; log_internal "Invalid bridge pair" |
| MESH_ISLAND_ADVERTISE: invalid island_id | DROP silently; log_internal "Invalid island_id format" |

---

## 4. Error Handling by Scenario

### 4.1 Malformed Common Header

**Trigger**: The common header cannot be parsed (too short, invalid version, non-zero reserved fields).

**Action**: Drop packet silently. Log internally with: timestamp, source transport address, reason. Do NOT send ERROR.

### 4.2 Decryption Failure

**Trigger**: ChaCha20-Poly1305 authentication tag verification fails.

**Action**: Drop packet silently. Log internally. This is expected behavior when the node receives a packet not intended for it (e.g., stray packets, probes). Do NOT send ERROR.

**Critical**: The node MUST NOT distinguish between "wrong session key" and "corrupted packet" in its external behavior. Both produce identical silent drops.

### 4.3 Unknown Session ID

**Trigger**: Session ID in the common header does not correspond to any established session.

**Action**: Drop packet silently. Do NOT send ERROR. Do NOT confirm or deny the existence of the session.

### 4.4 Replay Attack

**Trigger**: Sequence number has already been processed within the current window, or falls outside the window.

**Action**: Drop packet silently. Log internally. Do NOT send ERROR.

### 4.5 Handshake Timeout

**Trigger**: HANDSHAKE_DONE is not received within the domain-appropriate timeout of HANDSHAKE_RESP being sent (see Section 8).

**Action**: Destroy partial session state. Zeroize all derived key material. Log internally.

### 4.6 Handshake Validation Failure

**Trigger**: HANDSHAKE_DONE confirmation hash does not match expected value.

**Action**: Destroy partial session state. Zeroize all key material. Log internally. Do NOT send ERROR.

### 4.7 Exit Policy Denied

**Trigger**: Gateway receives a STREAM_OPEN with a destination that violates its exit policy (denied port, denied destination, denied protocol).

**Action**: Send STREAM_CLOSE with close reason ERR_EXIT_POLICY_DENIED. Log the denied destination category (but NOT the full destination — the gateway should log "port 25 denied" not "user tried to connect to mail.example.com").

### 4.8 ECH Required but Not Available

**Trigger**: Stream requires ECH (`require_ech = true`) but the destination does not publish ECH configuration.

**Action**: Send STREAM_CLOSE with ERR_ECH_REQUIRED_NOT_SUPPORTED (0x0016). This is a stream-close, not a session-close. If `require_ech = false`, the client MAY attempt one retry without ECH.

### 4.9 Unknown path_id

**Trigger**: Return traffic packet references a path_id not in the node's single-domain or cross-domain path table.

**Action**: Drop silently (Error 0x0019). Log internally. Path table entry may have expired (TTL = session idle timeout); client will rebuild path.

### 4.10 Withdrawn Node Contact

**Trigger**: Path construction attempts to use a node that has published NODE_WITHDRAW.

**Action**: Exclude node from path construction. Error 0x0018. Client rebuilds path using different nodes.

### 4.11 Pool Curator Key Rotation Invalid

**Trigger**: Received POOL_KEY_ROTATION record fails dual-signature verification.

**Action**: Drop the record silently. Log internally. Continue using current cached curator key.

### 4.12 Domain Bridge Node Failure Mid-Path

**Trigger**: Preceding relay's KEEPALIVE to a bridge node times out.

**Action**: The relay cannot inform the client (it only knows path_id). Client detects failure via end-to-end KEEPALIVE timeout. Client rebuilds path, selecting a different bridge node for the failed domain transition. path_id entries in both domains are orphaned and expire at TTL.

### 4.13 Mesh Island Unreachable During Construction

**Trigger**: All bridge nodes to a mesh island are offline when path construction is attempted.

**Action**: Return ERR_MESH_ISLAND_UNREACHABLE (0x0020). If `store_and_forward = true`: queue the path request and retry when island becomes reachable (bridge node publishes updated island advertisement to unified DHT).

### 4.14 No Bridge Nodes for Domain Transition

**Trigger**: Unified DHT has no valid DOMAIN_ADVERTISE records for the required domain pair.

**Action**: Return ERR_DOMAIN_BRIDGE_UNAVAILABLE (0x001F). Application may try alternative routing (longer path through intermediate domain, alternative bridge pool, etc.).

### 4.15 Cross-Domain path_id Unknown for Return Traffic

**Trigger**: Return traffic arrives at a bridge node with path_id not in cross-domain path table.

**Action**: ERR_PATH_ID_UNKNOWN — DROP silently. Same as single-domain behavior. Table entry may have expired if bridge was temporarily offline. Client rebuilds path.

---

## 5. Logging Requirements

### 5.1 What MUST NEVER Be Logged

- Client IP addresses at relay or exit nodes
- Destination addresses (full) at relay or exit nodes
- path_id values
- Path composition (which nodes appeared in the same path)
- Session identifiers
- Traffic volume per path or per session
- Pool query history (which pools a client queried)
- Application data content
- Specific denied destination addresses (log denial category, not specific destination)
- Which routing domains a specific session traversed
- Cross-domain path composition
- Mesh island identifiers associated with specific sessions

### 5.2 What MAY Be Logged (Operator Discretion)

- Aggregate statistics: packets per hour, sessions per hour, error rates
- Error categories: decryption failures per hour, replay attempts per hour (NOT session details)
- Node performance: per-session latency statistics (without session identifiers)
- Exit policy enforcement events: category and protocol, not specific destination
- Aggregate count of cross-domain path constructions and bridge transitions
- Mesh island reachability status (is/is not reachable; no connection details)

### 5.3 What SHOULD Be Logged (Internal, for Operational Purposes)

- Node daemon startup and shutdown events
- DHT join/leave events
- Advertisement publication timestamps
- Key rotation events (pool curator rotations accepted)
- Critical failures (out of memory, disk full, etc.)

### 5.4 Log Retention

Operational logs SHOULD be retained for no more than 7 days. Aggregate statistics MAY be retained indefinitely.

---

## 6. Fail-Safe Defaults

When in doubt, BGP-X components MUST fail toward privacy:

| Situation | Fail-Safe Behavior |
|---|---|
| Cannot determine if packet is valid | Drop silently |
| Cannot determine exit policy applicability | Deny |
| Cannot reach DHT | Fail path construction; do NOT attempt direct connection |
| Session key derivation produces unexpected result | Abort handshake; zeroize all material |
| Clock synchronization failure | Enter CLOCK_SUSPECT state; widen timestamp acceptance to 120s; attempt NTP sync |
| Memory allocation failure in crypto path | Abort session immediately (panic is safer than undefined behavior) |
| Fragment buffer full | Discard oldest incomplete fragmented packet |

---

## 7. Clock Drift Handling

```
Normal operation:       Timestamp acceptance window: ±60 seconds

Clock drift detected:   If timestamp validation fails by >300 seconds:
                        → Log "Clock drift detected: Δ=N seconds"
                        → Enter CLOCK_SUSPECT state
                        → Widen window to ±120 seconds
                        → Attempt NTP sync

NTP sync confirmed:     Return to ±60 second window
                        → Exit CLOCK_SUSPECT state

NTP sync fails:         Remain in CLOCK_SUSPECT state
                        → Alert operator via log
                        → Do NOT reject all handshakes (remain operational with wider window)
```

---

## 8. Timeout Values

| Event | Timeout | Action on Timeout |
|---|---|---|
| Handshake completion (clearnet) | 10 seconds | Destroy partial session |
| Handshake completion (WiFi mesh) | 15 seconds | Destroy partial session |
| Handshake completion (LoRa) | 60 seconds | Destroy partial session |
| Session idle (clearnet) | 90 seconds | Close session |
| Session idle (LoRa/satellite path) | Configurable, max 5 minutes | Close session |
| KEEPALIVE interval | 25s ± 5s (MUST randomize) | Send KEEPALIVE |
| KEEPALIVE interval (LoRa) | 60s ± 10s | Send KEEPALIVE |
| Bridge discovery DHT query | 10 seconds | ERR_DOMAIN_BRIDGE_UNAVAILABLE |
| Path construction (single domain) | 30 seconds | ERR_SEGMENT_CONSTRUCTION_FAILED |
| Path construction (cross-domain) | 60 seconds | ERR_NO_CROSS_DOMAIN_PATH |
| Store-and-forward retry | Configurable, default 60 seconds | Retry |
| Store-and-forward TTL | Configurable, default 3600 seconds | Drop queued request |
| Stream open acknowledgment | 15 seconds | ERR_DESTINATION_UNREACHABLE |
| Fragment reassembly | 10 seconds | Discard fragment buffer |
| Pool re-verification after rotation | 24 hours | Exclude old-key-signed members |
