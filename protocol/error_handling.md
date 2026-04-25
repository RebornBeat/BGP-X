# BGP-X Error Handling Specification

**Version**: 0.1.0-draft

---

## 1. Error Handling Philosophy

BGP-X is a privacy system. Error messages are a potential information leak. The error handling design follows these principles:

1. **Silent drop over informative error**: When a packet cannot be processed (decryption failure, unknown session, invalid format), nodes SHOULD drop the packet silently rather than send an error response. Error responses confirm that a packet was received and processed, which is itself information.

2. **No error amplification**: Error handling MUST NOT send multiple responses to a single malformed packet. Amplification can be exploited for DoS.

3. **No session confirmation**: A node MUST NOT confirm whether a given session ID exists. Both "session not found" and "decryption failure" MUST produce identical behavior (silent drop, optional internal log).

4. **Errors are for established sessions**: ERROR messages (0x10) are only sent within established sessions where the sender is already known. They are not sent in response to unauthenticated packets.

---

## 2. Error Codes

| Code | Name | Description | Response |
|---|---|---|---|
| 0x0000 | ERR_NONE | No error (informational) | — |
| 0x0001 | ERR_PROTOCOL_VERSION | Unsupported protocol version | ERROR or silent drop |
| 0x0002 | ERR_INVALID_HEADER | Malformed common header | Silent drop |
| 0x0003 | ERR_DECRYPTION_FAILURE | Onion layer decryption failed | Silent drop |
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

---

## 3. Error Handling by Scenario

### 3.1 Malformed Common Header

**Trigger**: The common header cannot be parsed (too short, invalid version, non-zero reserved fields).

**Action**: Drop packet silently. Log internally with: timestamp, source IP:port, received bytes, reason. Do NOT send ERROR.

### 3.2 Decryption Failure

**Trigger**: ChaCha20-Poly1305 authentication tag verification fails.

**Action**: Drop packet silently. Log internally. This is expected behavior when the node receives a packet not intended for it (e.g., stray packets, probes). Do NOT send ERROR.

**Critical**: The node MUST NOT distinguish between "wrong session key" and "corrupted packet" in its external behavior. Both produce identical silent drops.

### 3.3 Unknown Session ID

**Trigger**: Session ID in the common header does not correspond to any established session.

**Action**: Drop packet silently. Do NOT send ERROR. Do NOT confirm or deny the existence of the session.

### 3.4 Replay Attack

**Trigger**: Sequence number has already been processed within the current window, or falls outside the window.

**Action**: Drop packet silently. Log internally. Do NOT send ERROR.

### 3.5 Handshake Timeout

**Trigger**: HANDSHAKE_DONE is not received within 10 seconds of HANDSHAKE_RESP being sent.

**Action**: Destroy partial session state. Zeroize all derived key material. Log internally.

### 3.6 Handshake Validation Failure

**Trigger**: HANDSHAKE_DONE confirmation hash does not match expected value.

**Action**: Destroy partial session state. Zeroize all derived key material. Log internally. Do NOT send ERROR.

### 3.7 Exit Policy Denied

**Trigger**: Gateway receives a STREAM_OPEN with a destination that violates its exit policy (denied port, denied destination, denied protocol).

**Action**: Send STREAM_CLOSE with close reason ERR_EXIT_POLICY_DENIED. Log the denied destination category (but NOT the full destination — the gateway should log "port 25 denied" not "user tried to connect to mail.example.com").

### 3.8 Destination Unreachable

**Trigger**: Gateway cannot establish a connection to the clearnet destination (TCP connection refused, DNS resolution failure, timeout).

**Action**: Send STREAM_CLOSE with close reason ERR_DESTINATION_UNREACHABLE. Do NOT include the specific error details (e.g., DNS error vs TCP refused) in the STREAM_CLOSE — this leaks information about the destination.

### 3.9 Node at Capacity

**Trigger**: A node cannot accept a new session because it has reached its configured maximum concurrent sessions.

**Action**: During handshake: drop HANDSHAKE_INIT silently. After session established: send ERROR with code ERR_SESSION_FULL, then close the session.

---

## 4. Logging Requirements

### 4.1 What MUST NOT be logged

- Client source IP addresses at relay or exit nodes
- Full destination addresses at exit nodes
- Content of application data
- Session keys or any derived cryptographic material
- Stream-level correlation data (which streams appeared on which path at which time)

### 4.2 What MAY be logged (operator discretion)

- Aggregate statistics: packets forwarded per hour, bytes relayed, session count
- Error rates: decryption failures per hour, replay attempts per hour
- Node performance: per-session latency statistics (without session identifiers)
- Exit policy enforcement events: category and protocol, not specific destination

### 4.3 What SHOULD be logged (internal, for operational purposes)

- Node daemon startup and shutdown events
- DHT join/leave events
- Advertisement publication timestamps
- Critical failures (out of memory, disk full, etc.)

### 4.4 Log retention

Operational logs SHOULD be retained for no more than 7 days. Aggregate statistics MAY be retained indefinitely.

---

## 5. Fail-Safe Defaults

When in doubt, BGP-X components MUST fail toward privacy:

| Situation | Fail-Safe Behavior |
|---|---|
| Cannot determine if packet is valid | Drop silently |
| Cannot determine exit policy | Deny |
| Cannot reach DHT | Fail path construction, do not attempt direct connection |
| Session key derivation produces unexpected result | Abort handshake, destroy material |
| Clock synchronization failure | Reject handshakes until clock is synchronized |
| Memory allocation failure in crypto path | Panic (crash is safer than undefined behavior) |

---

## 6. Timeout Values

| Event | Timeout | Action on Timeout |
|---|---|---|
| Handshake completion | 10 seconds | Destroy partial session |
| Session idle (no traffic) | 90 seconds | Close session |
| KEEPALIVE interval | 25 seconds ± 5s | Send KEEPALIVE |
| DHT query response | 5 seconds | Retry or fail |
| Path construction | 30 seconds | Fail and report |
| Stream open acknowledgment | 15 seconds | Send STREAM_CLOSE |
| Exit connection establishment | 10 seconds | ERR_DESTINATION_UNREACHABLE |
