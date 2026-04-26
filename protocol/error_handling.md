# BGP-X Error Handling Specification

**Version**: 0.1.0-draft

---

## 1. Error Handling Philosophy

BGP-X is a privacy system. Error messages are a potential information leak. Design principles:

1. **Silent drop over informative error**: malformed packets, unknown sessions, decryption failures → drop silently
2. **No error amplification**: no multiple responses to one malformed packet
3. **No session confirmation**: identical behavior for session-not-found and decryption-failure
4. **Errors only within established sessions**: ERROR messages (0x10) sent only to established sessions
5. **Fail toward privacy**: when in doubt, drop silently

---

## 2. Error Codes

| Code | Name | Description | Response |
|---|---|---|---|
| 0x0000 | ERR_NONE | No error | — |
| 0x0001 | ERR_PROTOCOL_VERSION | Unsupported protocol version | ERROR or silent drop |
| 0x0002 | ERR_INVALID_HEADER | Malformed common header | Silent drop |
| 0x0003 | ERR_DECRYPTION_FAILURE | Authentication tag mismatch | Silent drop |
| 0x0004 | ERR_UNKNOWN_SESSION | Session ID not found | Silent drop |
| 0x0005 | ERR_REPLAY | Sequence number replay | Silent drop |
| 0x0006 | ERR_HANDSHAKE_TIMEOUT | Handshake not completed within timeout | Silent drop |
| 0x0007 | ERR_HANDSHAKE_INVALID | Handshake content failed validation | Silent drop |
| 0x0008 | ERR_SESSION_FULL | Node cannot accept new sessions | ERROR (if possible) |
| 0x0009 | ERR_STREAM_NOT_FOUND | Stream ID does not exist | ERROR |
| 0x000A | ERR_STREAM_ALREADY_OPEN | Stream ID already in use | ERROR |
| 0x000B | ERR_EXIT_POLICY_DENIED | Destination rejected by exit policy | ERROR |
| 0x000C | ERR_DESTINATION_UNREACHABLE | Exit could not reach destination | ERROR |
| 0x000D | ERR_BANDWIDTH_EXCEEDED | Connection exceeds bandwidth limit | ERROR |
| 0x000E | ERR_INVALID_ADVERTISEMENT | Advertisement failed validation | (internal) |
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
| 0x0022 | ERR_DOMAIN_POLICY_DENIED | Domain's routing policy rejected this segment | STREAM_CLOSE |
| 0x0023 | ERR_DOMAIN_SCOPE_MISMATCH | Pool domain_scope does not match required segment domain | (path construction) |
| 0x0024 | ERR_DOMAIN_BRIDGE_POOL_INVALID | Domain-bridge pool does not match required bridge pair | (path construction) |

---

## 3. Malformed Onion Handling

| Failure Condition | Behavior |
|---|---|
| Plaintext < 57 bytes | DROP silently; log_internal "Layer too short" |
| Unknown hop_type (not 0x01-0x09) | DROP silently; log_internal "Unknown hop type" |
| Invalid next_hop encoding | DROP silently; log_internal "Invalid next hop" |
| path_id = 0x0000000000000000 | DROP silently; log_internal "Invalid path_id: all zeros" |
| stream_id = 0 | DROP silently; log_internal "Invalid stream_id: zero" |
| Client stream_id is even | Send STREAM_CLOSE RST; log_internal "Stream ID parity violation" |
| Service stream_id is odd | Send STREAM_CLOSE RST; log_internal "Stream ID parity violation" |
| hop_type EXIT but node has no exit role | DROP silently; log_internal "Exit hop at non-exit node" |
| hop_type DELIVERY but service not registered | Send STREAM_CLOSE ERR_DESTINATION_UNREACHABLE |
| hop_type DOMAIN_BRIDGE but node not bridge-capable | DROP silently; log_internal "DOMAIN_BRIDGE at non-bridge node" |
| hop_type DOMAIN_BRIDGE with unknown domain_type | DROP silently; log_internal "Unknown target domain type" |
| hop_type DOMAIN_BRIDGE with unreachable target domain | DROP silently; client detects via KEEPALIVE timeout |
| hop_type MESH_ENTRY but no mesh transport | DROP silently; log_internal "MESH_ENTRY at non-mesh node" |
| hop_type MESH_RELAY but no mesh transport | DROP silently |
| Fragment: total_fragments = 0 | DROP silently |
| Fragment: fragment_num >= total_fragments | DROP silently |
| Fragment timeout (10 seconds) | Discard fragment buffer; log_internal "Fragment timeout" |
| Reassembled packet > 1280 bytes | DROP; log_internal "Reassembled packet exceeds MTU" |
| DOMAIN_ADVERTISE: invalid bridge pair | DROP silently; log_internal "Invalid bridge pair" |
| MESH_ISLAND_ADVERTISE: invalid island_id | DROP silently; log_internal "Invalid island_id format" |

---

## 4. Cross-Domain Error Handling

### 4.1 Domain Bridge Node Failure Mid-Path

Preceding relay's KEEPALIVE to bridge node times out. The relay cannot inform the client (it only knows path_id). Client detects via end-to-end KEEPALIVE timeout.

**Action**: client rebuilds path, selecting a different bridge node for the failed domain transition.

path_id entries in both domains are orphaned; they expire at TTL (session idle timeout) without further action.

### 4.2 Mesh Island Unreachable During Construction

**Trigger**: all bridge nodes to a mesh island are offline when path construction is attempted.

**Action**: return ERR_MESH_ISLAND_UNREACHABLE. If `store_and_forward = true`: queue the path request and retry when island becomes reachable (bridge node publishes updated island advertisement to unified DHT).

### 4.3 No Bridge Nodes for Domain Transition

**Trigger**: unified DHT has no valid DOMAIN_ADVERTISE records for the required domain pair.

**Action**: return ERR_DOMAIN_BRIDGE_UNAVAILABLE. Application may try alternative routing (longer path through intermediate domain, alternative bridge pool, etc.).

### 4.4 Cross-Domain path_id Unknown for Return Traffic

**Trigger**: return traffic arrives at a bridge node with path_id not in cross-domain path table.

**Action**: ERR_PATH_ID_UNKNOWN — DROP silently. Same as single-domain behavior. Table entry may have expired if bridge was temporarily offline. Client rebuilds path.

---

## 5. Logging Requirements

### MUST NEVER Be Logged

- Client IP addresses at relay or exit nodes
- Destination addresses (full) at relay or exit nodes
- path_id values
- Path composition (which nodes appeared in same path)
- Session identifiers
- Traffic volume per path or per session
- Pool query history
- Application data content
- Specific denied destination addresses (log denial category only)
- Which routing domains a specific session traversed
- Cross-domain path composition
- Mesh island identifiers associated with specific sessions

### MAY Be Logged (Operational)

- Aggregate statistics: packets per hour, sessions per hour, error rates
- Error categories and rates (no session details)
- Node performance: aggregate latency statistics
- Aggregate count of cross-domain path constructions and bridge transitions
- Mesh island reachability status (is/is not reachable; no connection details)

---

## 6. Timeout Values

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
| Fragment reassembly | 10 seconds | Discard |
| Pool re-verification after rotation | 24 hours | Exclude old-key-signed members |
