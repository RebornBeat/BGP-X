# BGP-X Data Plane Forwarding Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X data plane moves packets through the overlay network in real time. Cross-domain routing extends the forwarding pipeline with new hop type handling (0x06–0x09) and cross-domain path_id routing.

Requirements:
- **High throughput**: forwarding at wire speed for available bandwidth
- **Low latency overhead**: no more than 1-2ms processing latency per hop; ≤2ms additional for DOMAIN_BRIDGE
- **Memory safety**: all parsing and decryption safe against malformed input; no raw pointer arithmetic
- **Constant-time operations**: no timing-channel leakage from crypto operations
- **Privacy by design**: no logging of prohibited information

---

## 2. Forwarding Pipeline

```
Transport (receive — any domain: UDP, WiFi mesh, LoRa, etc.)
        │
        ▼
┌───────────────────┐
│ 1. Packet intake  │  Parse outer header; validate format
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 2. Session lookup │  Constant-time lookup in session table (INBOUND sequence space)
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 3. Replay check   │  Verify sequence in inbound sliding window
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 4. Decryption     │  Decrypt onion layer; verify Poly1305 tag (constant-time)
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 5. Layer parse    │  Extract hop_type, next_hop, path_id, stream_id, flags
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 6. path_id record │  Store path_id → source in domain path table
│                   │  For bridge nodes: update cross-domain path table
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 7. Dispatch       │  Route by hop_type:
│  0x01 RELAY       │    Forward same domain (UDP)
│  0x02 EXIT_TCP    │    Clearnet exit IPv4
│  0x03 EXIT_UDP    │    Clearnet exit IPv6
│  0x04 EXIT_RESOLVE│    Resolve domain at exit
│  0x05 DELIVERY    │    Deliver to native service
│  0x06 DOMAIN_BRIDGE→   Cross-domain forward (NEW)
│  0x07 MESH_ENTRY  →    Enter mesh island (NEW)
│  0x08 MESH_EXIT   →    Exit mesh island (NEW)
│  0x09 MESH_RELAY  →    Relay within mesh (NEW)
│  0x11 COVER       │    Identical processing to 0x01 RELAY
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 8. Output queue   │  Enqueue for transmission on correct transport
└────────┬──────────┘
         ▼
Transport (send — appropriate domain transport)
```

---

## 3. Stage 1: Packet Intake

### UDP/IP (Clearnet)

`SO_REUSEPORT` socket bound to `0.0.0.0:7474`, multiple receiver threads share the socket.

Socket buffer sizes: `SO_RCVBUF = SO_SNDBUF = 16 MB`

### Mesh Transports

Transport-specific receive via MeshTransport interface. MESH_FRAGMENT reassembly before processing. Each mesh transport has its own receive goroutine/thread feeding the same intake pipeline.

### Header Validation

1. Datagram ≥ 32 bytes — if shorter: drop silently
2. Version = 0x01 — else: drop silently
3. Reserved = 0x0000 — else: drop silently
4. Payload Length ≤ (datagram_length - 32) — else: drop silently
5. Unknown Msg Type: drop silently (forward compatibility)

### Message Type Routing

| Msg Type | Handler |
|---|---|
| 0x01 RELAY | Forwarding pipeline |
| 0x02-0x04 HANDSHAKE | Handshake handler |
| 0x09 KEEPALIVE | Keepalive handler |
| 0x0A-0x0F DHT | DHT handler |
| 0x11 COVER | Identical to RELAY (same pipeline) |
| 0x15 NODE_WITHDRAW | Withdrawal handler |
| 0x16 PATH_QUALITY_REPORT | Return path handler (opaque forward via path_id) |
| 0x18 MESH_BEACON | Beacon handler |
| 0x19 MESH_FRAGMENT | Fragment reassembly → intake pipeline |
| 0x1A DOMAIN_ADVERTISE | DHT handler (store bridge record) |
| 0x1B MESH_ISLAND_ADVERTISE | DHT handler (store island record) |
| 0x1C POOL_KEY_ROTATION | Rotation handler |

---

## 4. Stage 2: Session Lookup

**Concurrent hash map (sharded, 64 shards)** keyed by Session ID (16 bytes).

Lookup MUST be constant-time with respect to session existence. Timing differences between "found" and "not found" can leak session existence information.

```
session = session_table.get(header.session_id)  // Constant-time lookup

if session is None:
    drop_silently()
    log_internal("Unknown session")
    return

if session.state != ESTABLISHED:
    drop_silently()
    return
```

### Session Table Sizing

Maximum: configurable, default 10,000 concurrent sessions. When full: new HANDSHAKE_INIT silently dropped. Existing sessions unaffected.

---

## 5. Stage 3: Replay Detection

BGP-X uses separate sliding windows for INBOUND and OUTBOUND sequence numbers.

**Inbound window** (for received packets):

```
function check_inbound_sequence(session, inbound_seq):

    if inbound_seq > session.inbound_max:
        advance = inbound_seq - session.inbound_max
        session.inbound_window <<= advance
        session.inbound_window.set_bit(0)
        session.inbound_max = inbound_seq
        return ACCEPT

    offset = session.inbound_max - inbound_seq
    if offset >= 64:
        return REJECT("Too old")
    if session.inbound_window.get_bit(offset):
        return REJECT("Replay")

    session.inbound_window.set_bit(offset)
    return ACCEPT
```

The header carries the OUTBOUND sequence number from the sender's perspective (which is the INBOUND sequence number from the receiver's perspective).

---

## 6. Stage 4: Decryption

```
nonce = 0x00000000 || header.sequence_number_BE8   (12 bytes)

plaintext = ChaCha20-Poly1305-Decrypt(
    key        = session.session_key,
    nonce      = nonce,
    aad        = header.serialize(),      // 32 bytes
    ciphertext = packet.payload
)

if plaintext is None:
    drop_silently()
    log_internal("Auth failure")
    increment session.decryption_failures

    if session.decryption_failures > 10 in last 60 seconds:
        flag session for review (possible MITM attempt)

    return
```

Authentication tag verification MUST be constant-time. MUST occur before any plaintext processing.

### Key Material Protection

Session keys stored with:
- Guard page protection
- `mlock()` to prevent swap to disk (Linux)
- Automatic zeroization on session close via `Drop` implementation

---

## 7. Stage 5: Layer Parsing

Parse 57-byte onion layer header:

```
hop_type  = plaintext[0]          // 1 byte
next_hop  = plaintext[1:41]       // 40 bytes (encoding depends on hop_type)
path_id   = plaintext[41:49]      // 8 bytes — return traffic routing identifier
stream_id = plaintext[49:53]      // 4 bytes
flags     = plaintext[53:55]      // 2 bytes
reserved  = plaintext[55:57]      // 2 bytes (MUST be 0x0000)
remaining = plaintext[57:]        // subsequent hops (still encrypted)
```

**Total layer header: 57 bytes.**

Validation:
- `hop_type` not in {0x01..0x09}: drop silently, log "Unknown hop type"
- `path_id` = 0x0000000000000000: drop silently, log "Invalid path_id: all zeros"
- Plaintext length < 57: drop silently, log "Layer too short"
- `reserved` ≠ 0x0000: drop silently

---

## 8. Stage 6: path_id Recording

**Standard relay nodes**:

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

**Domain bridge nodes** (when hop_type == 0x06 DOMAIN_BRIDGE):

```
// Single-domain path table (this domain's side)
path_table.insert(
    key = path_id,
    value = PathTableEntry {
        source_addr:   packet.source_addr,
        domain:        current_domain_id,
        ...
    }
)

// Cross-domain path table (for return routing across domain boundary)
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

Both path tables: **in-memory only**. NEVER logged. NEVER persisted to disk.

Background cleanup task every 30 seconds removes expired entries from both tables.

---

## 9. Stage 7: Dispatch

### RELAY (0x01) — Standard Forwarding

```python
next_node = parse_next_hop(next_hop)  // NodeID + IPv4 address + UDP port

new_header = CommonHeader {
    msg_type:        0x01,
    session_id:      outbound_session_to_next_hop.session_id,
    sequence_number: outbound_session.next_sequence(),
    payload_length:  len(remaining)
}

transport.send_udp(next_node.addr, new_header || remaining)
```

The outbound session to the next hop is a separate session established during path construction.

### DOMAIN_BRIDGE (0x06) — Cross-Domain Forward (NEW)

```python
target_domain_id = next_hop[0:8]    // 8 bytes: type (4B BE) + instance (4B BE)
bridge_node_id   = next_hop[8:40]   // 32 bytes NodeID

# Validate target domain type
if target_domain_id.type not in KNOWN_DOMAIN_TYPES:
    drop_silently("Unknown target domain type")
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

### MESH_ENTRY (0x07)

```python
island_domain_id = next_hop[0:8]
first_mesh_relay = next_hop[8:40]
mesh_addr = mesh_transport.resolve_node_addr(first_mesh_relay, island_domain_id)
if mesh_addr is None:
    drop_silently("First mesh relay not discoverable")
    return
mesh_transport.send(mesh_addr, new_header || remaining)
```

### MESH_EXIT (0x08)

```python
exit_domain_id   = next_hop[0:8]
exit_bridge_node = next_hop[8:40]
exit_transport   = transport_manager.get_transport_for_domain(exit_domain_id)
exit_node_addr   = resolve_node_in_domain(exit_bridge_node, exit_domain_id)
exit_transport.send(exit_node_addr, new_header || remaining)
```

### MESH_RELAY (0x09)

```python
next_relay_id  = next_hop[0:32]
lora_addr_hint = next_hop[32:36]  // 0x00000000 = WiFi mesh auto-resolve
mesh_addr = resolve_mesh_addr(next_relay_id, lora_addr_hint)
mesh_transport.send(mesh_addr, new_header || remaining)
```

### EXIT_TCP (0x02), EXIT_UDP (0x03), EXIT_RESOLVE (0x04), DELIVERY (0x05)

See `/gateway/gateway_spec.md` for complete exit dispatch including ECH, DoH DNS, and ECS stripping.

### COVER Traffic (0x11)

COVER packets processed identically to RELAY (0x01) through the same complete pipeline. If the decrypted payload produces uninterpretable content (random bytes formatted as onion layer), the node drops silently at the dispatch stage. This is expected and correct behavior — external observers cannot distinguish COVER from RELAY in any routing domain.

---

## 10. Cross-Domain Return Traffic Routing

When return traffic arrives at a domain bridge node from the target domain:

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

Intermediate relays and bridge nodes do NOT decrypt return traffic. The encrypted blob is forwarded using path_id lookups at each hop.

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
- `clearnet` (type 0x00000001): UDP socket on 0.0.0.0:7474
- `mesh:<island_id>` (type 0x00000003): WiFi mesh raw socket, LoRa serial, or BLE
- `lora-regional:<region_id>` (type 0x00000004): LoRa serial for that region
- `satellite:<orbit_id>` (type 0x00000005): satellite modem interface

---

## 12. Stage 8: Output Queue

**Per-destination bounded queue** with dedicated sender thread pool.

Parameters:
- Queue depth per destination: 256 packets
- Sender threads: 4
- Send buffer size: 4 MB per socket

### Backpressure

If output queue full:
- Drop current packet (tail drop)
- Increment congestion counter for session
- Signal congestion control module

### Batched Sends

Linux `sendmmsg`: sender threads batch multiple packets per syscall. Significantly reduces per-packet overhead at high packet rates.

---

## 13. Keepalive Handling

```python
def handle_keepalive(session, header, source_addr):

    if session is None or session.state != ESTABLISHED:
        return  # silent drop

    session.last_seen = current_time()
    geo_plausibility.record_rtt(session.remote_node_id, measure_rtt(source_addr))

    response = CommonHeader {
        msg_type:        0x09,
        session_id:      header.session_id,
        sequence_number: session.next_outbound_sequence(),
        payload_length:  0
    }
    transport.send(source_addr, response)
```

### Session Liveness Monitoring (Background Task, Every 10 Seconds)

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

---

## 15. Geographic Plausibility Measurement

```python
# On KEEPALIVE send:
keepalive_sent[session_id] = current_time()

# On KEEPALIVE receive:
if session_id in keepalive_sent:
    rtt = current_time() - keepalive_sent[session_id]
    geo_plausibility.record_rtt(session.remote_node_id, rtt)
    del keepalive_sent[session_id]
```

Domain-specific thresholds applied in the geo plausibility system.

---

## 16. Concurrency Model

```
Receiver Threads (N = CPU_CORES / 2)
  ├── SO_REUSEPORT binding for UDP
  ├── Each handles: intake, session lookup, replay check, decryption
  └── Produces work items → Worker Threads

Worker Threads (N = CPU_CORES / 2)
  ├── Layer parse, path_id recording
  ├── Dispatch (RELAY, DOMAIN_BRIDGE, EXIT, DELIVERY, MESH_*)
  └── Produces send items → Output Queues

Sender Threads (4)
  ├── Output queue drain
  ├── sendmmsg batching
  └── Per-domain transport selection

Background Tasks (Tokio async)
  ├── Session liveness monitor (every 10s)
  ├── Path table cleanup — single-domain (every 30s)
  ├── Cross-domain path table cleanup (every 30s)
  ├── DHT routing table maintenance (every 60min bucket refresh)
  └── Advertisement re-publication (every 12h)

Concurrent Data Structures
  ├── Session Table: DashMap<SessionId, Arc<Session>> (64 shards)
  ├── Single-Domain Path Table: DashMap<PathId, PathTableEntry> (64 shards)
  └── Cross-Domain Path Table: DashMap<PathId, CrossDomainPathEntry> (64 shards)
```

All critical paths (session lookup, replay detection, decryption, path_id recording) are wait-free or lock-free using sharded maps and atomic operations.

---

## 17. Performance Targets

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

## 18. Forwarding Statistics

Per-session (aggregate, no identifiers):
- bytes_relayed, packets_relayed
- decryption_failures, replay_attempts

Aggregate daemon metrics:
- packets_per_second (current rate)
- bytes_per_second (current throughput)
- session_count (active)
- path_table_size (active path_id entries, single-domain)
- cross_domain_path_table_size (active cross-domain entries)
- queue_depth (output queue)
- drop_rate (congestion drops per second)
- domain_bridge_transitions_total (counter)
- return_traffic_forwarded_total (single-domain path_id)
- return_traffic_forwarded_cross_domain (cross-domain path_id)
- pt_active (boolean)
- mesh_active (boolean)

**Prohibited from statistics**: any session-level identifiers, node IDs, path compositions, which routing domains a session traversed, mesh island identifiers associated with sessions.
