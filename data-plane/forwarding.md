# BGP-X Data Plane Forwarding Specification

**Version**: 0.1.0-draft

This document specifies the complete packet forwarding behavior of BGP-X relay nodes — how packets are received, validated, decrypted, and forwarded through the overlay network.

---

## 1. Overview

The BGP-X data plane is the component responsible for moving packets through the overlay network in real time. It operates independently of the control plane (node discovery, path construction) and must satisfy strict requirements:

- **High throughput**: The forwarding path must be efficient enough to relay traffic at wire speed for the node's available bandwidth
- **Low latency overhead**: The per-packet processing cost must be minimal — the goal is to add no more than 1–2ms of processing latency per hop
- **Memory safety**: The forwarding path processes untrusted input; all parsing and decryption must be safe against malformed input
- **Constant-time operations**: Cryptographic operations must not leak timing information
- **Privacy by design**: The forwarding path must not log, store, or expose any information about packet origin, destination, or content beyond what is required to forward the packet

---

## 2. Forwarding Architecture

The BGP-X node forwarding pipeline consists of the following stages, executed in order for each received packet:

```
UDP Socket (receive)
        │
        ▼
┌───────────────────┐
│  1. Packet intake │  Parse outer header, validate fields
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  2. Session lookup│  Look up session ID in session table
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  3. Replay check  │  Verify sequence number not replayed
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  4. Decryption    │  Decrypt onion layer with session key
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  5. Layer parse   │  Extract hop type, next hop, payload
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  6. Dispatch      │  Route to relay, exit, or delivery handler
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  7. Output queue  │  Enqueue for transmission
└────────┬──────────┘
         │
         ▼
UDP Socket (send)
```

Each stage is described in detail below.

---

## 3. Stage 1: Packet Intake

### 3.1 UDP receive

The node maintains a UDP socket bound to `0.0.0.0:7474` (default port). The socket SHOULD use `SO_REUSEPORT` to allow multiple worker threads to receive packets in parallel without contention.

Recommended socket buffer sizes:

```
SO_RCVBUF = 16 MB
SO_SNDBUF = 16 MB
```

### 3.2 Outer header parsing

Upon receiving a UDP datagram, the forwarding pipeline:

1. Verifies the datagram is at least 32 bytes (minimum header size). If shorter, drop silently.
2. Parses the common header fields (Version, Msg Type, Reserved, Session ID, Sequence Number, Payload Length).
3. Verifies Version == 0x01 (current version). If unknown version, drop silently.
4. Verifies Reserved == 0x0000. If non-zero, drop silently.
5. Verifies Payload Length <= (datagram_length - 32). If the stated payload length exceeds available bytes, drop silently.
6. Verifies Msg Type is a known type. If unknown, drop silently (forward compatibility — a future version may define new types).

### 3.3 Message type routing

After header parsing, the packet is routed based on Msg Type:

| Msg Type | Handler |
|---|---|
| 0x01 RELAY | Forwarding pipeline (this document) |
| 0x02–0x04 HANDSHAKE | Handshake handler (see `/protocol/handshake.md`) |
| 0x09 KEEPALIVE | Keepalive handler |
| 0x0A–0x0F DHT | DHT handler (see `/control-plane/discovery.md`) |
| 0x11 COVER | Cover traffic handler (discard payload, update session liveness) |
| All others | Drop silently |

---

## 4. Stage 2: Session Lookup

### 4.1 Session table

The node maintains a session table: a concurrent hash map keyed by Session ID (16 bytes).

Each session table entry contains:

```
SessionEntry {
    session_id:       bytes[16]
    session_key:      bytes[32]      // ChaCha20-Poly1305 key
    sequence_window:  BitSet[64]     // Replay detection window
    sequence_max:     uint64         // Highest seen sequence number
    last_seen:        timestamp
    state:            SessionState   // HANDSHAKING | ESTABLISHED | CLOSING
    next_hop:         Option<NodeEndpoint>  // Cached from last RELAY
    bytes_relayed:    uint64
    packets_relayed:  uint64
}
```

### 4.2 Lookup procedure

```
session = session_table.get(header.session_id)

if session is None:
    drop_silently()
    log_internal("Unknown session ID, dropped")
    return

if session.state != ESTABLISHED:
    drop_silently()
    return
```

The session table lookup MUST be constant-time with respect to whether the session exists or not. Timing differences between "session not found" and "session found but state wrong" can leak session existence information.

Implementation note: Use a constant-time comparison for Session ID lookup rather than a standard hash map get, or add a timing equalizer after the lookup.

### 4.3 Session table sizing

Maximum sessions per node: configurable, default 10,000

When the session table is full, new handshakes are rejected (ERROR ERR_SESSION_FULL). In-flight sessions are not affected.

---

## 5. Stage 3: Replay Detection

### 5.1 Sliding window algorithm

BGP-X uses a 64-bit sliding window for replay detection per session.

```
function check_and_update_sequence(session, seq):

    if seq > session.sequence_max:
        # New highest sequence number — advance window
        advance = seq - session.sequence_max
        session.sequence_window = session.sequence_window << advance
        session.sequence_window.set_bit(0)
        session.sequence_max = seq
        return ACCEPT

    else:
        # Sequence number is within or below the window
        offset = session.sequence_max - seq

        if offset >= 64:
            # Too old — outside window
            return REJECT("Sequence number too old")

        if session.sequence_window.get_bit(offset):
            # Already received
            return REJECT("Replay detected")

        # New packet within window
        session.sequence_window.set_bit(offset)
        return ACCEPT
```

### 5.2 Sequence number overflow

Sequence numbers are uint64. At 1 million packets per second, overflow occurs after approximately 584,000 years. No overflow handling is required in the current specification.

### 5.3 Initial sequence number

The initial sequence number for a session is a random uint64 generated during handshake. It is communicated to the peer during HANDSHAKE_DONE. This prevents sequence number prediction for new sessions.

---

## 6. Stage 4: Decryption

### 6.1 Decryption algorithm

```
function decrypt_onion_layer(session, header, payload):

    # Construct nonce: 4 zero bytes + 8-byte sequence number
    nonce = bytes([0, 0, 0, 0]) + header.sequence_number.to_bytes(8, 'big')

    # Construct AAD: the 32-byte common header
    aad = header.serialize()

    # Attempt decryption
    plaintext = chacha20_poly1305_decrypt(
        key      = session.session_key,
        nonce    = nonce,
        aad      = aad,
        ciphertext = payload  # includes 16-byte Poly1305 tag at end
    )

    if plaintext is None:
        # Authentication tag mismatch
        return (FAILURE, None)

    return (SUCCESS, plaintext)
```

### 6.2 Authentication tag verification

The Poly1305 authentication tag is 16 bytes appended to the end of the ciphertext. Verification is performed atomically with decryption by ChaCha20-Poly1305 — the plaintext is never returned if the tag does not match.

Tag verification MUST be constant-time. A timing-variable comparison allows an attacker to determine whether a forged packet partially matches.

### 6.3 Decryption failure handling

If decryption fails:

- Drop the packet silently
- Do NOT send an error response
- Increment the session's decryption_failure counter
- If decryption_failure_count > 10 in a 60-second window, flag the session for review

A high rate of decryption failures on a specific session may indicate:
- A replay attack
- A packet delivered to the wrong node (routing error)
- An active MITM attempting to inject packets

### 6.4 Key zeroization

Session keys are stored in memory allocated with a guard page and cleared (zeroized) when:
- The session is closed
- The process exits (via atexit handler)
- A signal is received (SIGTERM, SIGINT)

On Linux, session keys SHOULD be stored in memory locked with `mlock()` to prevent swapping to disk.

---

## 7. Stage 5: Layer Parsing

After successful decryption, the plaintext is the onion layer content. The layer is parsed to extract:

- Hop Type (1 byte)
- Next Hop (40 bytes)
- Remaining Ciphertext (payload for subsequent hops)

```
function parse_onion_layer(plaintext):

    if len(plaintext) < 49:
        return (FAILURE, "Layer too short")

    hop_type = plaintext[0]
    next_hop_raw = plaintext[1:41]
    flags = plaintext[41:43]
    stream_id = plaintext[43:47]
    reserved = plaintext[47:49]
    remaining = plaintext[49:]

    if hop_type not in VALID_HOP_TYPES:
        return (FAILURE, "Unknown hop type")

    next_hop = parse_next_hop(hop_type, next_hop_raw)
    if next_hop is None:
        return (FAILURE, "Invalid next hop")

    return (SUCCESS, LayerContent{
        hop_type:  hop_type,
        next_hop:  next_hop,
        flags:     flags,
        stream_id: stream_id,
        remaining: remaining
    })
```

### 7.1 Hop type dispatch

| Hop Type | Action |
|---|---|
| 0x01 RELAY | Forward remaining ciphertext to next BGP-X node |
| 0x02 EXIT_IPV4 | Connect to IPv4 destination and forward |
| 0x03 EXIT_IPV6 | Connect to IPv6 destination and forward |
| 0x04 EXIT_DOMAIN | Resolve domain and connect to destination |
| 0x05 DELIVERY | Deliver to local BGP-X native service |

---

## 8. Stage 6: Dispatch

### 8.1 Relay dispatch (Hop Type 0x01)

For relay hops, the node constructs a new RELAY packet and sends it to the next BGP-X node:

```
function relay_dispatch(layer_content, original_sequence):

    next_node = layer_content.next_hop  // NodeEndpoint {ip, port}

    # Construct new outer header
    new_header = CommonHeader {
        version:         0x01,
        msg_type:        0x01,  // RELAY
        reserved:        0x0000,
        session_id:      <outbound session ID for this path segment>,
        sequence_number: <next sequence number for outbound session>,
        payload_length:  len(layer_content.remaining)
    }

    # Send to next node
    udp_send(next_node.ip, next_node.port, new_header + layer_content.remaining)
```

**Important**: The outbound session ID is the session established between this relay node and the next node — not the session ID from the original packet. Each hop has independent session IDs and session keys. This is what prevents the next hop from correlating this packet with the original sender.

### 8.2 Exit dispatch (Hop Types 0x02–0x04)

For exit hops, the gateway connects to the clearnet destination and forwards the plaintext payload (which at this point is the actual application data, decrypted by all onion layers):

```
function exit_dispatch(layer_content):

    destination = layer_content.next_hop  // IP/domain + port

    # Check exit policy
    if not exit_policy.allows(destination):
        send_stream_close(ERR_EXIT_POLICY_DENIED)
        return

    # Establish connection to destination
    conn = tcp_connect(destination)  // or udp if indicated by flags
    if conn is None:
        send_stream_close(ERR_DESTINATION_UNREACHABLE)
        return

    # Forward payload
    conn.send(layer_content.remaining)

    # Handle response (forward back through overlay)
    response = conn.receive()
    overlay_send_response(response)
```

### 8.3 Delivery dispatch (Hop Type 0x05)

For BGP-X native services, the gateway delivers the decrypted payload to the local service handler:

```
function delivery_dispatch(layer_content):

    service_id = layer_content.next_hop.service_id
    service = local_services.get(service_id)

    if service is None:
        send_stream_close(ERR_DESTINATION_UNREACHABLE)
        return

    service.deliver(layer_content.stream_id, layer_content.remaining)
```

---

## 9. Stage 7: Output Queue

### 9.1 Queue architecture

The output queue is a per-destination bounded queue. Packets are enqueued after dispatch and transmitted by a dedicated sender thread pool.

Queue parameters:

| Parameter | Default | Description |
|---|---|---|
| Queue depth per destination | 256 packets | Maximum enqueued packets before backpressure |
| Sender threads | 4 | Thread pool for async UDP sends |
| Send buffer size | 4 MB | Per-socket send buffer |

### 9.2 Backpressure

If the output queue for a destination is full:

- Drop the current packet (tail drop)
- Increment the congestion counter for this session
- Signal the congestion control module (see `/data-plane/congestion_control.md`)

### 9.3 Batched sends

Where the OS supports it (`sendmmsg` on Linux), the sender thread SHOULD batch multiple packets into a single syscall. This significantly reduces per-packet overhead at high packet rates.

---

## 10. Keepalive Handling

KEEPALIVE messages (Msg Type 0x09) are processed outside the main forwarding pipeline:

```
function handle_keepalive(session, header):

    # Verify session exists and is ESTABLISHED
    if session is None or session.state != ESTABLISHED:
        return  # silent drop

    # Update last_seen timestamp
    session.last_seen = current_time()

    # Send KEEPALIVE response
    response_header = CommonHeader {
        version:         0x01,
        msg_type:        0x09,
        session_id:      header.session_id,
        sequence_number: session.next_outbound_sequence(),
        payload_length:  0
    }
    udp_send(source_addr, response_header)
```

### 10.1 Session liveness monitoring

A background task runs every 10 seconds and checks all sessions:

```
for session in session_table.values():
    if current_time() - session.last_seen > 90 seconds:
        close_session(session, reason="keepalive_timeout")
        reputation_system.record(session.remote_node_id, KEEPALIVE_TIMEOUT)
```

---

## 11. Concurrency Model

The BGP-X forwarding pipeline is designed for multi-core execution.

### 11.1 Thread model

```
┌──────────────────────────────────────────────────────┐
│  Receiver Threads (N = CPU cores / 2)                │
│  Each binds to same socket via SO_REUSEPORT          │
│  Handles: intake, session lookup, replay, decryption │
└──────────────────────────┬───────────────────────────┘
                           │ (via channel)
┌──────────────────────────▼───────────────────────────┐
│  Worker Threads (N = CPU cores / 2)                  │
│  Handles: layer parse, dispatch, output queue        │
└──────────────────────────┬───────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────┐
│  Sender Threads (4)                                   │
│  Handles: output queue drain, UDP send               │
└──────────────────────────────────────────────────────┘
```

### 11.2 Session table concurrency

The session table uses a sharded concurrent hash map. The key space is divided into 64 shards, each protected by its own read-write lock. Session lookups (reads) are concurrent; session creation and deletion (writes) acquire the shard write lock.

### 11.3 Receiver thread affinity

Receiver threads SHOULD be pinned to specific CPU cores using `pthread_setaffinity_np` (Linux) to reduce cache misses and context switching.

---

## 12. Performance Targets

| Metric | Target |
|---|---|
| Per-packet forwarding latency | < 1 ms (p99) |
| Maximum relay throughput (single core) | 500 Mbps |
| Maximum relay throughput (8 cores) | 2 Gbps |
| Maximum concurrent sessions | 10,000 |
| Session table lookup time | < 100 ns (p99) |
| Decryption throughput (ChaCha20-Poly1305) | > 4 Gbps (hardware) |
| Memory per session | < 256 bytes |

---

## 13. Zero-Copy Packet Processing

For high-throughput nodes, the implementation SHOULD minimize data copying:

- Receive buffer is pre-allocated and reused (ring buffer)
- Decryption is performed in-place where ChaCha20-Poly1305 permits
- The remaining onion ciphertext is passed by reference to the output queue
- UDP send uses `sendmsg` with `iovec` to avoid copying header + payload separately

---

## 14. Forwarding Statistics

The forwarding pipeline maintains per-session and aggregate statistics:

Per-session:
- `bytes_relayed` — total bytes forwarded
- `packets_relayed` — total packets forwarded
- `decryption_failures` — count of failed decryptions
- `replay_attempts` — count of replay-detected packets

Aggregate (no session identifiers):
- `packets_per_second` — current forwarding rate
- `bytes_per_second` — current throughput
- `session_count` — active sessions
- `queue_depth` — current output queue depth
- `drop_rate` — packets dropped (congestion) per second

These are exposed via the Control API (`get_stats` method).
