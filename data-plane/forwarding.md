# BGP-X Data Plane Forwarding Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X data plane moves packets through the overlay network in real time. Cross-domain routing extends the forwarding pipeline with new hop type handling (0x06–0x09) and cross-domain path_id routing.

Requirements:
- High throughput: forwarding at wire speed
- Low latency: ≤2ms processing overhead per hop (≤2ms additional for DOMAIN_BRIDGE)
- Memory safety: all parsing and decryption safe against malformed input
- Constant-time crypto operations: no timing-channel leakage
- Privacy by design: no logging of prohibited information

---

## 2. Forwarding Pipeline

```
Transport (receive — any domain: UDP, WiFi mesh, LoRa, etc.)
        │
        ▼
┌───────────────────┐
│ 1. Packet intake  │  Parse outer header, validate
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 2. Session lookup │  Constant-time lookup in session table
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 3. Replay check   │  Inbound sliding window check
└────────┬──────────┘
         ▼
┌───────────────────┐
│ 4. Decryption     │  Decrypt; verify Poly1305 tag (constant-time)
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
│ 7. Dispatch       │  Route by hop_type 0x01-0x09:
│   0x01 RELAY      │    Forward same domain (UDP)
│   0x02 EXIT_TCP   │    Clearnet exit IPv4
│   0x03 EXIT_UDP   │    Clearnet exit IPv6
│   0x04 EXIT_RESOLVE│   Resolve domain at exit
│   0x05 DELIVERY   │    Deliver to native service
│   0x06 DOMAIN_BRIDGE → Cross-domain forward
│   0x07 MESH_ENTRY → Enter mesh island
│   0x08 MESH_EXIT  → Exit mesh island
│   0x09 MESH_RELAY → Relay within mesh (radio transport)
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

**UDP/IP**: `SO_REUSEPORT` socket on `0.0.0.0:7474`, multiple receiver threads.

**Mesh transports**: transport-specific receive via MeshTransport interface. MESH_FRAGMENT reassembly before processing.

Socket buffer sizes: `SO_RCVBUF = SO_SNDBUF = 16 MB`

Validation:
1. Datagram ≥ 32 bytes — if shorter: drop silently
2. Version = 0x01 — else drop silently
3. Reserved = 0x0000 — else drop silently
4. Payload Length ≤ (datagram_length - 32) — else drop silently
5. Unknown Msg Type: drop silently (forward compatibility)

---

## 4. Stage 2: Session Lookup

Concurrent hash map (sharded, 64 shards) keyed by Session ID.

Lookup MUST be constant-time with respect to session existence — timing differences between found/not-found can leak session existence information.

```
session = session_table.get(header.session_id)  // Constant-time

if session is None:
    drop_silently()
    log_internal("Unknown session")
    return
```

Maximum: configurable (default 10,000 sessions). When full: new HANDSHAKE_INIT dropped silently.

---

## 5. Stage 3: Replay Detection

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

---

## 6. Stage 4: Decryption

```
nonce = 0x00000000 || header.sequence_number_BE8  (12 bytes)

plaintext = ChaCha20-Poly1305-Decrypt(
    key        = session.session_key,
    nonce      = nonce,
    aad        = header.serialize(),    (32 bytes)
    ciphertext = packet.payload
)

if plaintext is None:
    drop_silently()
    log_internal("Auth failure")
    return
```

Authentication tag verification MUST be constant-time. MUST occur before any plaintext processing.

Session keys stored with:
- Guard page protection
- `mlock()` to prevent swap to disk
- Automatic zeroization on session close via `Drop` implementation

---

## 7. Stage 5: Layer Parsing

Parse 57-byte onion layer header:

```
hop_type  = plaintext[0]          // 1 byte
next_hop  = plaintext[1:41]       // 40 bytes
path_id   = plaintext[41:49]      // 8 bytes
stream_id = plaintext[49:53]      // 4 bytes
flags     = plaintext[53:55]      // 2 bytes
reserved  = plaintext[55:57]      // 2 bytes
remaining = plaintext[57:]        // subsequent hops
```

Validation:
- `hop_type` not in {0x01..0x09}: drop silently, log "Unknown hop type"
- `path_id` = 0x0000000000000000: drop silently, log "Invalid path_id"
- Plaintext length < 57: drop silently, log "Layer too short"
- `reserved` ≠ 0x0000: drop silently

---

## 8. Stage 6: path_id Recording

**Standard relay nodes**:
```
path_table[path_id] = PathTableEntry {
    source_addr: packet.source_addr,
    session_id:  header.session_id,
    created:     current_time(),
    ttl:         SESSION_IDLE_TIMEOUT,
}
```

**Domain bridge nodes** (when hop_type == 0x06):
```
// Standard path table (current domain side)
path_table[path_id] = PathTableEntry {
    source_addr:   packet.source_addr,
    domain:        current_domain_id,
    ...
}

// Cross-domain path table (for return routing across domain boundary)
cross_domain_path_table[path_id] = CrossDomainPathEntry {
    source_domain:  current_domain_id,
    source_addr:    packet.source_addr,
    target_domain:  decoded_target_domain_id,  // From bytes 0-7 of next_hop
    created:        current_time(),
    ttl:            SESSION_IDLE_TIMEOUT,
}
```

Path tables: in-memory only. NEVER logged. NEVER persisted to disk.

Background cleanup every 30 seconds removes expired entries.

---

## 9. Stage 7: Dispatch

### RELAY (0x01) — Unchanged

Forward remaining ciphertext to next relay within current domain using outbound session.

### EXIT_TCP, EXIT_UDP, EXIT_RESOLVE, DELIVERY (0x02–0x05) — Unchanged

See `/gateway/gateway_spec.md` for exit dispatch including ECH, DoH DNS, ECS stripping.

### DOMAIN_BRIDGE (0x06) — New

```python
elif hop_type == 0x06:

    target_domain_id = next_hop[0:8]   # 8 bytes domain ID
    bridge_node_id   = next_hop[8:40]  # 32 bytes NodeID

    # Validate target domain type
    if target_domain_id.type not in KNOWN_DOMAIN_TYPES:
        drop_silently("Unknown target domain type")
        return

    # Get transport for target domain
    target_transport = transport_manager.get_transport_for_domain(target_domain_id)
    if target_transport is None:
        drop_silently("No transport for target domain")
        log_internal("DOMAIN_BRIDGE: missing transport for " + target_domain_id)
        return

    # Record cross-domain path_id for return routing
    cross_domain_path_table[path_id] = CrossDomainPathEntry(
        source_domain = current_domain_id,
        source_addr   = packet.source_addr,
        target_domain = target_domain_id,
        created       = current_time(),
        ttl           = SESSION_IDLE_TIMEOUT,
    )

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

### MESH_ENTRY (0x07) — New

```python
elif hop_type == 0x07:
    island_domain_id = next_hop[0:8]
    first_mesh_relay = next_hop[8:40]
    mesh_addr = mesh_transport.resolve_node_addr(first_mesh_relay, island_domain_id)
    if mesh_addr is None:
        drop_silently("First mesh relay not discoverable")
        return
    mesh_transport.send(mesh_addr, new_header || remaining)
```

### MESH_EXIT (0x08) — New

```python
elif hop_type == 0x08:
    exit_domain_id   = next_hop[0:8]
    exit_bridge_node = next_hop[8:40]
    exit_transport   = transport_manager.get_transport_for_domain(exit_domain_id)
    exit_node_addr   = resolve_node_in_domain(exit_bridge_node, exit_domain_id)
    exit_transport.send(exit_node_addr, new_header || remaining)
```

### MESH_RELAY (0x09) — New

```python
elif hop_type == 0x09:
    next_relay_id  = next_hop[0:32]
    lora_addr_hint = next_hop[32:36]  # 0x00000000 for WiFi mesh auto-resolve
    mesh_addr = resolve_mesh_addr(next_relay_id, lora_addr_hint)
    mesh_transport.send(mesh_addr, new_header || remaining)
```

### COVER Traffic (0x11)

Processed identically to RELAY (0x01) through the same pipeline. If the decrypted payload produces uninterpretable content (random bytes), the node drops silently at the dispatch stage — this is expected and correct.

---

## 10. Cross-Domain Return Traffic Routing

When return traffic arrives at a domain bridge node from the target domain:

```python
def handle_return_traffic(packet, source_domain):

    path_id = extract_path_id_from_return_traffic(packet)

    entry = cross_domain_path_table.get(path_id)
    if entry is None:
        drop_silently("Unknown path_id for cross-domain return")
        return

    if entry.source_domain != source_domain:
        drop_silently("Cross-domain path_id domain mismatch")
        return

    # Forward opaque blob to source domain predecessor
    source_transport = transport_manager.get_transport_for_domain(entry.source_domain)
    source_transport.send(entry.source_addr, packet)
    # NO decryption — bridge node is opaque forwarder for return traffic
```

---

## 11. Domain Transport Selector

```rust
pub struct TransportManager {
    transports: HashMap<DomainId, Box<dyn Transport>>,
}

impl TransportManager {
    pub fn get_transport_for_domain(&self, domain: &DomainId) -> Option<&dyn Transport> {
        self.transports.get(domain)
    }
}
```

Transport types per domain:
- clearnet: UDP socket
- mesh:\<island_id\>: WiFi mesh raw socket, LoRa serial, or BLE
- lora-regional:\<region_id\>: LoRa serial for that region
- satellite:\<orbit_id\>: satellite modem interface

---

## 12. Stage 8: Output Queue

Per-destination bounded queue. Sender thread pool (4 threads). Queue depth: 256 packets per destination.

**Backpressure**: if output queue full → drop current packet (tail drop) → increment congestion counter.

**Batched sends** (Linux `sendmmsg`): sender threads batch multiple packets per syscall.

---

## 13. Keepalive Handling

```python
def handle_keepalive(session, header, source_addr):
    if session is None or session.state != ESTABLISHED:
        return  # silent drop

    session.last_seen = current_time()
    geo_plausibility.record_rtt(session.remote_node_id, measure_rtt(source_addr))

    response = CommonHeader(msg_type=0x09, session_id=header.session_id,
                            sequence_number=session.next_outbound_sequence(),
                            payload_length=0)
    transport.send(source_addr, response)
```

Session liveness monitoring (background task, every 10 seconds):
```python
for session in session_table.values():
    if current_time() - session.last_seen > session.idle_timeout:
        close_session(session, "keepalive_timeout")
        reputation_system.record(session.remote_node_id, KEEPALIVE_TIMEOUT)

for entry in path_table.values():
    if current_time() - entry.created > entry.ttl:
        path_table.remove(entry.path_id)

for entry in cross_domain_path_table.values():
    if current_time() - entry.created > entry.ttl:
        cross_domain_path_table.remove(entry.path_id)
```

---

## 14. Performance Targets

| Metric | Target |
|---|---|
| Per-packet forwarding latency (RELAY) | <1ms (p99) |
| DOMAIN_BRIDGE hop processing overhead | <2ms additional vs RELAY |
| Cross-domain path_id table lookup | <100ns (p99) |
| Transport selection for domain | <50ns (p99) |
| Maximum relay throughput (8 cores) | >1 Gbps |
| Maximum concurrent sessions | 10,000 |
| Session table lookup | <100ns (p99) |
| Decryption throughput | >4 Gbps (ChaCha20-Poly1305) |
| Memory per session | <256 bytes |

---

## 15. Forwarding Statistics

Per-session: bytes_relayed, packets_relayed, decryption_failures, replay_attempts.

Aggregate: packets_per_second, bytes_per_second, session_count, path_table_size, cross_domain_path_table_size, queue_depth, drop_rate, pt_active, mesh_active, domain_bridge_transitions_total, return_traffic_forwarded_cross_domain.

Prohibited: any session-level identifiers, node IDs, path compositions.
