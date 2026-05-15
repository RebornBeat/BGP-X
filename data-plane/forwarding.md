# BGP-X Data Plane Forwarding Specification

**Version**: 0.1.0-draft

This document specifies the complete packet forwarding behavior of BGP-X relay nodes — how packets are received, validated, decrypted, and forwarded through the overlay network, including cross-domain routing through domain bridge nodes.

---

## 1. Overview

The BGP-X data plane is the component responsible for moving packets through the overlay network in real time. It operates independently of the control plane (node discovery, path construction) and must satisfy strict requirements:

- **High throughput**: The forwarding path must be efficient enough to relay traffic at wire speed for the node's available bandwidth
- **Low latency overhead**: The per-packet processing cost must be minimal — the goal is to add no more than 1–2ms of processing latency per hop; ≤2ms additional for DOMAIN_BRIDGE hop type
- **Memory safety**: The forwarding path processes untrusted input; all parsing and decryption must be safe against malformed input; no raw pointer arithmetic
- **Constant-time operations**: Cryptographic operations must not leak timing information
- **Privacy by design**: The forwarding path must not log, store, or expose any information about packet origin, destination, or content beyond what is required to forward the packet

---

## 2. Forwarding Architecture

The BGP-X node forwarding pipeline consists of the following stages, executed in order for each received packet:

```
Transport (receive — any domain: UDP, WiFi mesh, LoRa, BLE, Satellite)
        │
        ▼
┌───────────────────┐
│  1. Packet intake │  Parse outer header, validate fields
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 2. Session lookup │  Constant-time lookup in session table
│                   │  (INBOUND sequence space)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  3. Replay check  │  Verify sequence in inbound sliding window
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 4. Decryption     │  Decrypt onion layer; verify Poly1305 tag
│                   │  (constant-time verification)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 5. Layer parse    │  Extract hop_type, next_hop, path_id,
│                   │  stream_id, flags
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 6. path_id record │  Store path_id → source in domain path table
│                   │  For bridge nodes: update cross-domain path table
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 7. Dispatch       │  Route by hop_type:
│                   │
│  0x01 RELAY       │    Forward same domain (UDP or mesh)
│  0x02 EXIT_TCP    │    Clearnet exit IPv4
│  0x03 EXIT_UDP    │    Clearnet exit IPv6
│ 0x04 EXIT_RESOLVE │    Resolve domain at exit
│ 0x05 DELIVERY     │    Deliver to native service
│ 0x06 DOMAIN_BRIDGE│    Cross-domain forward
│ 0x07 MESH_ENTRY   │    Enter mesh island
│ 0x08 MESH_EXIT    │    Exit mesh island
│ 0x09 MESH_RELAY   │    Relay within mesh island
│ 0x11 COVER        │    Identical processing to 0x01 RELAY
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 8. Output queue   │  Enqueue for transmission on correct transport
└────────┬──────────┘
         │
         ▼
Transport (send — appropriate domain transport)
```

Each stage is described in detail below.

---

## 3. Stage 1: Packet Intake

### 3.1 Transport Receive

The node receives packets on multiple transports depending on its configuration:

**UDP/IP (Clearnet)**:
- `SO_REUSEPORT` socket bound to `0.0.0.0:7474`
- Multiple receiver threads share the socket for parallel processing
- Socket buffer sizes: `SO_RCVBUF = SO_SNDBUF = 16 MB`

**Mesh Transports**:
- Transport-specific receive via `MeshTransport` interface
- MESH_FRAGMENT reassembly before processing
- Each mesh transport has its own receive goroutine/thread feeding the same intake pipeline

**Satellite Transports**:
- USB modem interface (Starlink Gen 3, Iridium Certus, Inmarsat)
- High-latency link; extended keepalive intervals
- From BGP-X's perspective, satellite is clearnet domain — same processing

### 3.2 Outer Header Parsing

Upon receiving a datagram (UDP or reassembled mesh packet), the forwarding pipeline:

1. **Verifies the datagram is at least 32 bytes** (minimum header size). If shorter, drop silently.
2. **Parses the common header fields** (Version, Msg Type, Reserved, Session ID, Outbound Sequence Number, Payload Length).
3. **Verifies Version == 0x01** (current version). If unknown version, drop silently.
4. **Verifies Reserved == 0x0000**. If non-zero, drop silently.
5. **Verifies Payload Length <= (datagram_length - 32)**. If the stated payload length exceeds available bytes, drop silently.
6. **Verifies Msg Type is a known type**. If unknown, drop silently (forward compatibility — a future version may define new types).

### 3.3 Message Type Routing

After header parsing, the packet is routed based on Msg Type:

| Msg Type | Handler |
|---|---|
| 0x01 RELAY | Forwarding pipeline (this document) |
| 0x02-0x04 HANDSHAKE | Handshake handler (see `/protocol/handshake.md`) |
| 0x05 DATA | Data handler |
| 0x06-0x08 STREAM_* | Stream multiplexing handler |
| 0x09 KEEPALIVE | Keepalive handler |
| 0x0A-0x0F DHT | DHT handler (see `/control-plane/discovery.md`) |
| 0x10 ERROR | Error handler |
| 0x11 COVER | Identical to RELAY (same pipeline) |
| 0x12 DHT_POOL_ADVERTISE | DHT handler |
| 0x13 DHT_POOL_GET | DHT handler |
| 0x14 DHT_POOL_GET_RESP | DHT handler |
| 0x15 NODE_WITHDRAW | Withdrawal handler |
| 0x16 PATH_QUALITY_REPORT | Return path handler (opaque forward via path_id) |
| 0x17 PT_HANDSHAKE | Pluggable transport handler |
| 0x18 MESH_BEACON | Beacon handler |
| 0x19 MESH_FRAGMENT | Fragment reassembly → intake pipeline |
| 0x1A DOMAIN_ADVERTISE | DHT handler (store bridge record) |
| 0x1B MESH_ISLAND_ADVERTISE | DHT handler (store island record) |
| 0x1C POOL_KEY_ROTATION | Rotation handler |
| All others | Drop silently |

---

## 4. Stage 2: Session Lookup

### 4.1 Session Table

The node maintains a session table: a concurrent hash map keyed by Session ID (16 bytes). The table is sharded into 64 segments for concurrent access.

Each session table entry contains:

```
SessionEntry {
    session_id:       bytes[16]
    session_key:      bytes[32]      // ChaCha20-Poly1305 key
    inbound_window:   BitSet[64]     // INBOUND replay detection window
    inbound_max:      uint64         // Highest INBOUND sequence seen
    outbound_seq:     uint64         // Next OUTBOUND sequence number
    last_seen:        timestamp
    state:            SessionState   // HANDSHAKING | ESTABLISHED | CLOSING
    remote_node_id:   NodeID         // Peer's NodeID
    source_addr:      SocketAddr     // Transport address of peer
    domain_id:        DomainId       // Which routing domain this session is in
    bytes_relayed:    uint64
    packets_relayed:  uint64
    decryption_failures: uint64
    created:          timestamp
}
```

### 4.2 Lookup Procedure

```
session = session_table.get(header.session_id)  // Constant-time lookup

if session is None:
    drop_silently()
    log_internal("Unknown session ID, dropped")
    return

if session.state != ESTABLISHED:
    drop_silently()
    return
```

The session table lookup MUST be constant-time with respect to whether the session exists or not. Timing differences between "session not found" and "session found but state wrong" can leak session existence information.

**Implementation note**: Use a constant-time comparison for Session ID lookup rather than a standard hash map get, or add a timing equalizer after the lookup.

### 4.3 Session Table Sizing

Maximum sessions per node: configurable, default 10,000

When the session table is full, new handshakes are rejected (ERROR ERR_SESSION_FULL). In-flight sessions are not affected.

---

## 5. Stage 3: Replay Detection

### 5.1 Bidirectional Sequence Numbers

BGP-X uses **separate sliding windows for INBOUND and OUTBOUND sequence numbers**.

- **Outbound window**: tracks sequence numbers this node sends (prevents outbound replay)
- **Inbound window**: tracks sequence numbers this node receives (prevents inbound replay)

The common header carries the OUTBOUND sequence number from the sender's perspective. This becomes the INBOUND sequence number from the receiver's perspective.

This separation prevents nonce reuse even though the same session key is used for both directions.

### 5.2 Inbound Sliding Window Algorithm

```
function check_inbound_sequence(session, inbound_seq):

    if inbound_seq > session.inbound_max:
        # New highest sequence number — advance window
        advance = inbound_seq - session.inbound_max
        session.inbound_window = session.inbound_window << advance
        session.inbound_window.set_bit(0)
        session.inbound_max = inbound_seq
        return ACCEPT

    else:
        # Sequence number is within or below the window
        offset = session.inbound_max - inbound_seq

        if offset >= 64:
            # Too old — outside window
            return REJECT("Sequence number too old")

        if session.inbound_window.get_bit(offset):
            # Already received
            return REJECT("Replay detected")

        # New packet within window
        session.inbound_window.set_bit(offset)
        return ACCEPT
```

### 5.3 Sequence Number Overflow

Sequence numbers are uint64. At 1 million packets per second, overflow occurs after approximately 584,000 years. No overflow handling is required in the current specification.

### 5.4 Initial Sequence Number

The initial sequence number for a session is a random uint64 generated during handshake. It is communicated to the peer during HANDSHAKE_DONE. This prevents sequence number prediction for new sessions.

---

## 6. Stage 4: Decryption

### 6.1 Decryption Algorithm

```
function decrypt_onion_layer(session, header, payload):

    # Construct nonce: 4 zero bytes + 8-byte sequence number
    nonce = bytes([0, 0, 0, 0]) + header.sequence_number.to_bytes(8, 'big')

    # Construct AAD: the 32-byte common header
    aad = header.serialize()

    # Attempt decryption
    plaintext = ChaCha20-Poly1305-Decrypt(
        key        = session.session_key,
        nonce      = nonce,
        aad        = aad,
        ciphertext = payload  # includes 16-byte Poly1305 tag at end
    )

    if plaintext is None:
        # Authentication tag mismatch
        return (FAILURE, None)

    return (SUCCESS, plaintext)
```

### 6.2 Authentication Tag Verification

The Poly1305 authentication tag is 16 bytes appended to the end of the ciphertext. Verification is performed atomically with decryption by ChaCha20-Poly1305 — the plaintext is never returned if the tag does not match.

Tag verification MUST be constant-time. A timing-variable comparison allows an attacker to determine whether a forged packet partially matches.

**Critical**: Tag verification MUST occur before any plaintext processing.

### 6.3 Decryption Failure Handling

If decryption fails:

- Drop the packet silently
- Do NOT send an error response
- Increment the session's `decryption_failures` counter
- If `decryption_failures > 10` in a 60-second window, flag the session for review

A high rate of decryption failures on a specific session may indicate:
- A replay attack
- A packet delivered to the wrong node (routing error)
- An active MITM attempting to inject packets

### 6.4 Key Material Protection

Session keys are stored in memory with:
- Guard page protection
- `mlock()` to prevent swap to disk (Linux)
- Automatic zeroization on session close via `Drop` implementation

---

## 7. Stage 5: Layer Parsing

After successful decryption, the plaintext is the onion layer content. The layer is parsed to extract the 57-byte header:

### 7.1 Layer Header Structure

```
Byte offset:
  0        Hop Type (1 byte)
  1-40     Next Hop (40 bytes) — encoding depends on hop_type
  41-48    path_id (8 bytes) — return traffic routing identifier
  49-52    Stream ID (4 bytes)
  53-54    Flags (2 bytes)
  55-56    Reserved (2 bytes, MUST be 0x0000)
  57+      Remaining Ciphertext (for subsequent hops)
```

**Total layer header: 57 bytes.**

### 7.2 Field Definitions

| Field | Size | Description |
|---|---|---|
| hop_type | 1 byte | Determines dispatch action |
| next_hop | 40 bytes | Destination for this hop; format depends on hop_type |
| path_id | 8 bytes | Random identifier for return routing |
| stream_id | 4 bytes | Multiplexed stream identifier |
| flags | 2 bytes | Hop-specific flags |
| reserved | 2 bytes | Must be 0x0000 |
| remaining | variable | Onion layers for subsequent hops |

### 7.3 Hop Type Values

| Value | Meaning |
|---|---|
| 0x01 | RELAY — forward to another BGP-X node |
| 0x02 | EXIT_TCP — forward to clearnet destination (IPv4) |
| 0x03 | EXIT_UDP — forward to clearnet destination (IPv6) |
| 0x04 | EXIT_RESOLVE — forward to domain name (resolve at exit) |
| 0x05 | DELIVERY — this is the final hop (BGP-X native service) |
| 0x06 | DOMAIN_BRIDGE — transition to another routing domain |
| 0x07 | MESH_ENTRY — enter a mesh island from overlay |
| 0x08 | MESH_EXIT — leave a mesh island to overlay |
| 0x09 | MESH_RELAY — relay within a mesh island |

### 7.4 Validation

- `hop_type` not in {0x01..0x09}: drop silently, log "Unknown hop type"
- `path_id` = 0x0000000000000000: drop silently, log "Invalid path_id: all zeros"
- Plaintext length < 57: drop silently, log "Layer too short"
- `reserved` ≠ 0x0000: drop silently

---

## 8. Stage 6: path_id Recording

The path_id is an 8-byte random identifier generated by the client per path instance. It enables return traffic routing without requiring any relay to know the full path structure.

### 8.1 Standard Relay Nodes

```
path_table.insert(
    key = path_id,
    value = PathTableEntry {
        source_addr: packet.source_addr,
        session_id:  header.session_id,
        created:     current_time(),
        ttl:         SESSION_IDLE_TIMEOUT,   // 90 seconds
    }
)
```

### 8.2 Domain Bridge Nodes (when hop_type == 0x06)

Domain bridge nodes maintain TWO path tables:

**Single-domain path table** (this domain's side):
```
path_table.insert(
    key = path_id,
    value = PathTableEntry {
        source_addr:   packet.source_addr,
        domain:        current_domain_id,
        session_id:    header.session_id,
        created:       current_time(),
        ttl:           SESSION_IDLE_TIMEOUT,
    }
)
```

**Cross-domain path table** (for return routing across domain boundary):
```
cross_domain_path_table.insert(
    key = path_id,
    value = CrossDomainPathEntry {
        source_domain:  current_domain_id,
        source_addr:    packet.source_addr,
        target_domain:  decode_target_domain(next_hop[0:8]),
        created:        current_time(),
        ttl:            SESSION_IDLE_TIMEOUT,
    }
)
```

### 8.3 Path Table Properties

**In-memory only**. NEVER logged. NEVER persisted to disk.

**Background cleanup**: Task runs every 30 seconds and removes expired entries from both tables.

**path_id requirements**:
- Generated by the client using CSPRNG (8 bytes, 64 bits)
- Unique per path instance
- Included in EVERY onion layer of the same path
- The same value at every hop of the same path
- Never reused within a session
- Not linked to any client identity
- Not logged by any relay node

---

## 9. Stage 7: Dispatch

After recording the path_id, dispatch routes the packet based on hop_type.

### 9.1 RELAY (0x01) — Standard Forwarding

Forward remaining ciphertext to next BGP-X node within the current routing domain.

```python
next_node = parse_next_hop(next_hop)  # NodeID + IPv4/IPv6 + UDP port

# Construct new outer header with this node's OUTBOUND sequence
new_header = CommonHeader {
    version:         0x01,
    msg_type:        0x01,
    reserved:        0x0000,
    session_id:      outbound_session_to_next_hop.session_id,
    sequence_number: outbound_session.next_sequence(),
    payload_length:  len(remaining)
}

# Transmit remaining onion (already encrypted for subsequent hops)
transport.send_udp(next_node.addr, new_header || remaining)
```

**Important**: The outbound session to the next hop is a SEPARATE session established during path construction handshake. Each hop has independent session IDs and session keys. This prevents the next hop from correlating this packet with the original sender.

### 9.2 DOMAIN_BRIDGE (0x06) — Cross-Domain Forwarding

Domain bridge nodes handle transitions between routing domains.

```python
target_domain_id = next_hop[0:8]    # 8 bytes: type (4B BE) + instance (4B BE)
bridge_node_id   = next_hop[8:40]   # 32 bytes NodeID

# Validate target domain type
if target_domain_id.type not in KNOWN_DOMAIN_TYPES:
    drop_silently("Unknown target domain type")
    return

# Reject reserved domain type 0x00000005 (bgpx-satellite)
if target_domain_id.type == 0x00000005:
    drop_silently("Reserved domain type not active")
    return

# Get transport for target domain
target_transport = transport_manager.get_transport_for_domain(target_domain_id)
if target_transport is None:
    drop_silently("No transport for target domain")
    log_internal("DOMAIN_BRIDGE: missing transport for " + str(target_domain_id))
    return

# Resolve bridge node's address in target domain
bridge_node_addr = resolve_node_in_domain(bridge_node_id, target_domain_id)
if bridge_node_addr is None:
    drop_silently("Bridge node not reachable in target domain")
    return

# Forward remaining onion payload via target domain transport
# NO re-encryption: remaining payload already encrypted for subsequent hops
new_header = construct_outbound_header(outbound_session_to_bridge_node)
target_transport.send(bridge_node_addr, new_header || remaining)
```

**Critical**: The remaining onion payload is forwarded WITHOUT re-encryption. The bridge node decrypts only its own layer. Inner layers remain encrypted for subsequent mesh/target-domain hops.

### 9.3 MESH_ENTRY (0x07)

Enter a mesh island from the overlay or clearnet.

```python
island_domain_id = next_hop[0:8]
first_mesh_relay = next_hop[8:40]

mesh_addr = mesh_transport.resolve_node_addr(first_mesh_relay, island_domain_id)
if mesh_addr is None:
    drop_silently("First mesh relay not discoverable")
    return

mesh_transport.send(mesh_addr, new_header || remaining)
```

### 9.4 MESH_EXIT (0x08)

Exit a mesh island to the overlay or clearnet.

```python
exit_domain_id   = next_hop[0:8]
exit_bridge_node = next_hop[8:40]

exit_transport   = transport_manager.get_transport_for_domain(exit_domain_id)
if exit_transport is None:
    drop_silently("No transport for exit domain")
    return

exit_node_addr = resolve_node_in_domain(exit_bridge_node, exit_domain_id)
if exit_node_addr is None:
    drop_silently("Exit bridge node not reachable")
    return

exit_transport.send(exit_node_addr, new_header || remaining)
```

### 9.5 MESH_RELAY (0x09)

Relay within a mesh island using radio transport.

```python
next_relay_id  = next_hop[0:32]
lora_addr_hint = next_hop[32:36]  # 0x00000000 = WiFi mesh auto-resolve

mesh_addr = resolve_mesh_addr(next_relay_id, lora_addr_hint)
if mesh_addr is None:
    drop_silently("Mesh relay not discoverable")
    return

mesh_transport.send(mesh_addr, new_header || remaining)
```

### 9.6 EXIT_TCP (0x02), EXIT_UDP (0x03), EXIT_RESOLVE (0x04)

Exit nodes only. See `/gateway/gateway_spec.md` for complete exit dispatch including:
- ECH support
- DoH DNS resolution
- DNSSEC validation
- ECS stripping
- Exit policy enforcement

### 9.7 DELIVERY (0x05)

Deliver to a BGP-X native service running on this node.

```python
service_id = parse_service_id(next_hop)
service = local_services.get(service_id)

if service is None:
    send_stream_close(ERR_DESTINATION_UNREACHABLE)
    return

service.deliver(stream_id, remaining)
```

### 9.8 COVER Traffic (0x11)

COVER packets are processed identically to RELAY (0x01) through the same complete pipeline. 

If the decrypted payload produces uninterpretable content (random bytes formatted as onion layer), the node drops silently at the dispatch stage. This is expected and correct behavior — external observers cannot distinguish COVER from RELAY in any routing domain.

**Important**: COVER packets use the **same session_key as RELAY packets**. There is no separate cover_key. Both use ChaCha20-Poly1305 encryption and are externally indistinguishable.

---

## 10. Cross-Domain Return Traffic Routing

When return traffic (DATA, PATH_QUALITY_REPORT, or encrypted response blob) arrives at a domain bridge node from the target domain:

```python
def handle_cross_domain_return(packet, source_domain):

    path_id = extract_path_id_from_return_traffic(packet)

    entry = cross_domain_path_table.get(path_id)
    if entry is None:
        drop_silently()
        log_internal("Unknown path_id for cross-domain return")
        return

    if entry.source_domain != source_domain:
        drop_silently()
        log_internal("Cross-domain path_id domain mismatch")
        return

    # Forward opaque blob to source domain predecessor
    source_transport = transport_manager.get_transport_for_domain(entry.source_domain)
    source_transport.send(entry.source_addr, packet)
    # NO decryption — bridge node is opaque forwarder for return traffic
```

**Intermediate relays and domain bridge nodes do NOT decrypt return traffic.** The encrypted blob is forwarded using path_id lookups at each hop. Only the exit node (who encrypted the response for the client) and the client can decrypt the content.

This preserves the privacy property: no intermediate node can link source to destination.

---

## 11. Domain Transport Selector

```rust
pub struct TransportManager {
    // Key: DomainId, Value: owned transport implementation
    transports: HashMap<DomainId, Box<dyn Transport>>,
}

impl TransportManager {
    pub fn get_transport_for_domain(&self, domain: &DomainId) -> Option<&dyn Transport>;
    pub fn register_transport(&mut self, domain: DomainId, transport: Box<dyn Transport>);
}
```

Transport types per domain:

| Domain Type | Transport |
|---|---|
| clearnet (0x00000001) | UDP socket on 0.0.0.0:7474 |
| mesh (0x00000003) | WiFi mesh raw socket, LoRa serial, or BLE |
| lora-regional (0x00000004) | LoRa serial for that region |
| satellite (0x00000005) | RESERVED — not active |

**Note**: Commercial satellite internet (Starlink, Iridium, etc.) is clearnet domain (0x00000001) with satellite latency class. Domain type 0x00000005 is reserved for future BGP-X-native satellite infrastructure and is NOT active.

---

## 12. Stage 8: Output Queue

### 12.1 Queue Architecture

Per-destination bounded queue with dedicated sender thread pool.

**Parameters**:
- Queue depth per destination: 256 packets
- Sender threads: 4
- Send buffer size: 4 MB per socket

### 12.2 Backpressure

If output queue full:
- Drop current packet (tail drop)
- Increment congestion counter for session
- Signal congestion control module

### 12.3 Batched Sends

Linux `sendmmsg`: sender threads batch multiple packets per syscall. Significantly reduces per-packet overhead at high packet rates.

---

## 13. Keepalive Handling

### 13.1 KEEPALIVE Processing

```python
def handle_keepalive(session, header, source_addr):

    if session is None or session.state != ESTABLISHED:
        return  # silent drop

    session.last_seen = current_time()
    session.update_rtt_measurement(source_addr)  # For geo plausibility

    # Send KEEPALIVE response
    response = CommonHeader {
        version:         0x01,
        msg_type:        0x09,
        reserved:        0x0000,
        session_id:      header.session_id,
        sequence_number: session.next_outbound_sequence(),
        payload_length:  0
    }
    transport.send(source_addr, response)
```

### 13.2 KEEPALIVE Timing

Nodes MUST send KEEPALIVE at 25-second intervals on idle sessions, **randomized within ±5 seconds**.

This randomization is **MANDATORY** — not optional — to prevent KEEPALIVE timing fingerprinting.

### 13.3 Session Liveness Monitoring

Background task runs every 10 seconds:

```python
for session in session_table.values():
    if current_time() - session.last_seen > session.idle_timeout:
        close_session(session, "keepalive_timeout")
        reputation_system.record(session.remote_node_id, KEEPALIVE_TIMEOUT)

# Expire standard path table entries
for entry in path_table.values():
    if current_time() - entry.created > entry.ttl:
        path_table.remove(entry.path_id)

# Expire cross-domain path table entries
for entry in cross_domain_path_table.values():
    if current_time() - entry.created > entry.ttl:
        cross_domain_path_table.remove(entry.path_id)
```

### 13.4 Extended Timeouts for High-Latency Domains

For high-latency domain segments (LoRa, satellite GEO): KEEPALIVE interval and session dead timeout SHOULD be extended:

- **LoRa**: up to 120 seconds interval, 300 seconds dead timeout
- **Satellite GEO**: up to 300 seconds interval, 600 seconds dead timeout

These are configuration parameters, not protocol changes.

---

## 14. Pluggable Transport Integration

When PT is enabled:

```
Transport receive:
  Physical (UDP/radio) → PT deobfuscation → BGP-X forwarding pipeline

Transport send:
  BGP-X output queue → PT obfuscation → Physical (UDP/radio)
```

PT subprocess communicates via binary pipe. The BGP-X core does not see PT internals — it writes/reads BGP-X packets; PT handles obfuscation transparently.

**PT is below BGP-X protocol layer**. All hop types (including DOMAIN_BRIDGE, MESH_RELAY) are processed identically regardless of PT status.

---

## 15. Geographic Plausibility Measurement

Forwarding pipeline measures RTT to each node it communicates with:

```python
# On KEEPALIVE send:
keepalive_sent[session_id] = current_time()

# On KEEPALIVE receive:
if session_id in keepalive_sent:
    rtt = current_time() - keepalive_sent[session_id]
    geo_plausibility.record_rtt(session.remote_node_id, rtt)
    del keepalive_sent[session_id]
```

RTT measurements are fed to the geographic plausibility scoring system.

**Satellite nodes are exempt** from geo plausibility scoring — satellite terminal IP addresses may be geographically distant from the ground station.

**Mesh nodes use domain-specific thresholds** — WiFi mesh RTT expectations differ from internet RTT baselines.

---

## 16. Mesh Transport Handling

### 16.1 Fragmentation

For mesh transports with small MTU (particularly LoRa), BGP-X packets that exceed the transport MTU are fragmented using MESH_FRAGMENT (0x19):

- Fragmentation occurs BEFORE BGP-X header construction for the original packet
- Maximum 16 fragments per packet
- Fragment timeout: 10 seconds — if not all fragments received, discard and log
- Reassembly produces a complete BGP-X packet for normal processing
- packet_id is 4 bytes random per original packet (unique for reassembly tracking)

### 16.2 MTU Limits by Transport

| Transport | MTU Available | Notes |
|---|---|---|
| UDP/IP | 1280 bytes | BGP-X target MTU |
| WiFi 802.11s | 1280 bytes | Same as UDP/IP |
| LoRa SF7 | ~200 bytes usable | Requires MESH_FRAGMENT |
| LoRa SF12 | ~50 bytes usable | Requires MESH_FRAGMENT |
| Bluetooth BLE | 100-200 bytes | Requires MESH_FRAGMENT |
| Ethernet P2P | 1500 bytes | BGP-X target MTU still applies |

### 16.3 Mesh Transport Selection

When a node has multiple mesh transports available:
1. Prefer highest bandwidth for data traffic
2. Prefer LoRa for DHT queries (low bandwidth acceptable)
3. Use domain bridge nodes' advertised transport for cross-domain paths

---

## 17. Concurrency Model

The BGP-X forwarding pipeline is designed for multi-core execution.

### 17.1 Thread Model

```
┌──────────────────────────────────────────────────────┐
│  Receiver Threads (N = CPU cores / 2)                │
│  SO_REUSEPORT binding for UDP                        │
│  Mesh transport receive goroutines                   │
│  Handles: intake, session lookup, replay, decryption │
└──────────────────────────┬───────────────────────────┘
                           │ (via channel)
┌──────────────────────────▼───────────────────────────┐
│  Worker Threads (N = CPU cores / 2)                  │
│  Handles: layer parse, path_id recording, dispatch  │
└──────────────────────────┬───────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────┐
│  Sender Threads (4)                                   │
│  Handles: output queue drain, sendmmsg batching      │
└──────────────────────────────────────────────────────┘
```

### 17.2 Background Tasks (Tokio async)

- Session liveness monitor (every 10s)
- Path table cleanup — single-domain (every 30s)
- Cross-domain path table cleanup (every 30s)
- DHT routing table maintenance (every 60min bucket refresh)
- Advertisement re-publication (every 12h)

### 17.3 Concurrent Data Structures

- **Session Table**: `DashMap<SessionId, Arc<Session>>` (64 shards)
- **Single-Domain Path Table**: `DashMap<PathId, PathTableEntry>` (64 shards)
- **Cross-Domain Path Table**: `DashMap<PathId, CrossDomainPathEntry>` (64 shards)

All critical paths (session lookup, replay detection, decryption, path_id recording) are wait-free or lock-free using sharded maps and atomic operations.

---

## 18. Performance Targets

| Metric | Target | Minimum Acceptable |
|---|---|---|
| Per-packet forwarding latency (RELAY) | <1ms (p99) | <3ms (p99) |
| DOMAIN_BRIDGE hop overhead vs. RELAY | <2ms additional | <5ms additional |
| Cross-domain path_id table lookup | <100ns (p99) | <500ns (p99) |
| Transport selection per domain | <50ns (p99) | <200ns (p99) |
| Maximum relay throughput (8 cores) | >1 Gbps | >100 Mbps |
| Maximum concurrent sessions | 10,000 | 1,000 |
| Session table lookup | <100ns (p99) | <500ns (p99) |
| Decryption throughput | >4 Gbps (ChaCha20-Poly1305) | >1 Gbps |
| Memory per session | <256 bytes | <1 KB |

---

## 19. Forwarding Statistics

### 19.1 Per-Session Statistics

- `bytes_relayed`: total bytes forwarded
- `packets_relayed`: total packets forwarded
- `decryption_failures`: count of failed decryptions
- `replay_attempts`: count of replay-detected packets

### 19.2 Aggregate Daemon Metrics

- `packets_per_second`: current forwarding rate
- `bytes_per_second`: current throughput
- `session_count`: active sessions
- `path_table_size`: active path_id entries (single-domain)
- `cross_domain_path_table_size`: active cross-domain entries
- `queue_depth`: output queue depth
- `drop_rate`: congestion drops per second
- `domain_bridge_transitions_total`: counter for cross-domain hops processed
- `return_traffic_forwarded_total`: opaque blobs forwarded via path_id (single-domain)
- `return_traffic_forwarded_cross_domain`: opaque blobs forwarded across domain boundary
- `pt_active`: boolean
- `mesh_active`: boolean

### 19.3 Prohibited from Statistics

- Any session-level identifiers
- Node IDs
- Path compositions
- Which routing domains a session traversed
- Mesh island identifiers associated with sessions
- Client IP addresses
- Destination addresses
- path_id values

---

## 20. Implementation Notes

### 20.1 Zero-Copy Packet Processing

For high-throughput nodes, the implementation SHOULD minimize data copying:

- Receive buffer is pre-allocated and reused (ring buffer)
- Decryption is performed in-place where ChaCha20-Poly1305 permits
- The remaining onion ciphertext is passed by reference to the output queue
- UDP send uses `sendmsg` with `iovec` to avoid copying header + payload separately

### 20.2 Domain-Agnostic Crypto

All cryptographic operations are identical regardless of routing domain:

- Same ChaCha20-Poly1305 for UDP, WiFi mesh, LoRa, BLE, satellite
- Same X25519 key exchange during handshake
- Same HKDF-SHA256 key derivation
- No domain-specific nonce spaces
- No domain-specific cipher variations

### 20.3 No Re-Encryption at Domain Boundaries

Domain bridge nodes do NOT re-encrypt the remaining onion payload. The payload is already encrypted for subsequent hops. The bridge node only decrypts its own layer and forwards the inner layers opaquely.

This is a critical privacy property: even a malicious bridge node cannot read inner layers.
