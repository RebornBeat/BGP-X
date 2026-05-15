# BGP-X Stream Multiplexing Specification

**Version**: 0.1.0-draft

This document specifies BGP-X stream multiplexing — the ability to carry multiple independent logical connections over a single overlay path. This specification is domain-agnostic; multiplexing behavior is identical whether the path traverses clearnet, mesh islands, or any combination of routing domains.

---

## 1. Overview

BGP-X supports **stream multiplexing** — the ability to carry multiple independent logical connections over a single BGP-X path simultaneously.

### 1.1 Why Multiplexing Matters

Without multiplexing:
- Each connection (HTTP request, DNS query, API call) requires its own path
- Path setup (handshake × N hops) cost is paid for every connection
- Many simultaneous connections mean many independent circuit constructions, each potentially observable

With multiplexing:
- One path handles all connections in a session window
- Path setup cost is paid once
- Connection patterns across streams are harder to observe individually
- A browser loading a web page (dozens of connections) uses one BGP-X path
- Reduced anonymity risk: multiple streams on one path create ambiguity vs. multiple separate paths

### 1.2 Comparison to Tor

Tor creates one circuit per TCP stream. A web page load requiring 30 TCP connections creates 30 Tor circuits (or shares circuits in ways that reduce anonymity). BGP-X multiplexes all 30 connections over a single path, improving both efficiency and anonymity.

---

## 2. Stream Concepts

### 2.1 Stream Definition

A **stream** is a logical bidirectional channel within a BGP-X session. Streams are identified by a 32-bit **Stream ID** within their session.

Stream IDs are assigned by the initiating party (client for client-initiated streams, service for service-initiated streams).

### 2.2 Stream ID Assignment (MANDATORY)

Stream IDs are 32-bit unsigned integers. Assignment follows strict rules to prevent collision without coordination:

- **Client-initiated streams**: ODD IDs (1, 3, 5, ...)
- **Service-initiated streams**: EVEN IDs (2, 4, 6, ...)
- **Bit 31**: Indicates direction (0 = client-initiated, 1 = service-initiated)

**Violation Handling**: If a sender uses an ID that violates the odd/even rule for their role, the recipient MUST send a RST (reset) stream response and log the error internally. This rule applies in all routing domains.

### 2.3 Stream ID Exhaustion

After 2^31 streams per session have been opened (all 32-bit odd or even IDs used), the session MUST be torn down and a new session established. In practice, 2^31 streams per session will never be reached in normal use.

### 2.4 Stream ID Reuse

Stream IDs MUST NOT be reused within a session. Once a stream is CLOSED or RESET, its ID is retired and never reused on the same session.

### 2.5 Cross-Domain Stream Continuity

A BGP-X stream maintains its stream_id across all domain boundaries in a cross-domain path. The stream is NOT re-identified at domain transitions.

```
Stream ID 7 (client-initiated, odd)
  → in clearnet segment: stream_id = 7
  → at domain bridge: stream_id = 7
  → in mesh segment: stream_id = 7
  → at service: stream_id = 7
```

One stream, one stream_id, one socket — regardless of how many domains the path traverses.

### 2.6 Stream Types

| Type | Delivery | Use Cases |
|---|---|---|
| ORDERED | Reliable, ordered (like TCP) | HTTP, API calls, file transfers |
| DATAGRAM | Unreliable, unordered (like UDP) | DNS, VoIP, gaming |

Stream type is specified at stream open time and cannot be changed.

### 2.7 path_id Association

Each stream is associated with a specific path (via path_id). When the underlying path is rebuilt (including cross-domain rebuilds):
- The path_id changes
- In-flight streams may be migrated to the new path
- Stream state records include the current path_id

---

## 3. Stream Lifecycle

### 3.1 Stream States

```
IDLE → OPEN → HALF_CLOSED_LOCAL → CLOSED
           └→ HALF_CLOSED_REMOTE → CLOSED
           └→ RESET (immediate close, no drain)
           └→ PAUSED (store-and-forward for mesh/intermittent transport)
```

| State | Description |
|---|---|
| IDLE | Stream ID not yet used |
| OPEN | Both parties can send and receive |
| HALF_CLOSED_LOCAL | Local party sent FIN; can still receive |
| HALF_CLOSED_REMOTE | Remote party sent FIN; can still send |
| CLOSED | Both parties have sent FIN; no more data |
| RESET | Stream abruptly closed (RST flag) |
| PAUSED | Stream temporarily suspended due to transport unavailability |

### 3.2 Stream Open

A stream is opened by sending a STREAM_OPEN message (Msg Type 0x06) or a DATA message with the SYN flag set.

STREAM_OPEN payload contains the destination address, allowing the exit node to establish the outbound connection before any data arrives. This reduces latency for the first data exchange.

```
STREAM_OPEN {
    stream_id:       uint32  (client-initiated: odd; service-initiated: even)
    stream_type:     uint8   (0x01 = ORDERED, 0x02 = DATAGRAM)
    stream_flags:    uint16  (see below)
    destination_type: uint8  (0x04 = IPv4, 0x06 = IPv6, 0x0A = domain, 0xBB = ServiceID)
    destination:     bytes   (address)
    port:            uint16
}
```

**Stream Flags (STREAM_OPEN):**

| Bit | Name | Description |
|---|---|---|
| 0 | ECH_REQUIRED | Fail stream if ECH not available at exit |
| 1 | ECH_PREFERRED | Use ECH when available; fallback allowed |

### 3.3 Stream Close

A stream is closed by sending a DATA message with the FIN flag set. After sending FIN, the sender MUST NOT send any more data on this stream. It MAY continue to receive data until it receives FIN from the remote party.

### 3.4 Stream Reset

A stream is reset by sending a DATA message with the RST flag set. Both parties MUST immediately stop sending on this stream and discard any buffered data.

### 3.5 Stream Timeout

A stream with no activity for 5 minutes (no data, no keepalives specific to the stream) SHOULD be closed with a normal FIN. The session itself remains open.

### 3.6 PAUSED State (Cross-Domain Specific)

For LoRa/satellite transports with duty cycle limitations or intermittent connectivity, streams enter the PAUSED state.

```rust
pub enum StreamState {
    Open,
    HalfClosedLocal,
    HalfClosedRemote,
    Closed,
    Reset,
    Paused {
        domain: DomainId,
        resume_condition: ResumeCondition,
    },
}

pub enum ResumeCondition {
    DomainReachable,
    BridgeNodeOnline,
    Timeout { at: Instant },
}
```

**PAUSED State Behavior:**
- For LoRa duty cycle: stream pauses during duty cycle wait; resumes when duty cycle resets
- For offline mesh island: stream pauses; resumes when island bridge node comes online
- Application receives `STREAM_PAUSED` event (not an error) indicating temporary suspension
- If the `ResumeCondition::Timeout` expires, the stream transitions to CLOSED

---

## 4. Ordered Stream (ORDERED type)

Ordered streams provide reliable, sequenced delivery semantics equivalent to TCP. Ordered stream behavior is **domain-agnostic** — identical over clearnet, mesh, or satellite links.

### 4.1 Per-Stream Sequence Numbers

Ordered streams maintain independent per-stream sequence numbers (separate from the session-level sequence numbers used for replay protection):

```
stream_sequence_number: uint32  (wraps at 2^32)
```

Stream sequence numbers start at a random value chosen by the sender during STREAM_OPEN.

### 4.2 Data Framing

Each DATA message on an ordered stream includes:

```
DATA payload {
    stream_id:       uint32
    stream_sequence: uint32
    flags:           uint16
    data:            bytes
}
```

### 4.3 Acknowledgment

BGP-X uses a **cumulative acknowledgment** scheme. The receiver acknowledges the highest contiguous stream sequence number received:

```
ACK payload {
    stream_id:   uint32
    ack_seq:     uint32   // Highest contiguous sequence number received
    recv_window: uint32   // Current receive window size in bytes
}
```

ACK messages are sent:
- Immediately after receiving a FIN
- After receiving N bytes of data (default N = 16,384 bytes = 16 KB)
- After receiving a packet with the PUSH flag set
- After a delay of no more than 100ms since the last sent ACK

### 4.4 Retransmission

Unacknowledged data is retransmitted after a timeout:

```
retransmit_timeout = max(200ms, 2 × moving_average_rtt)
```

On retransmission, the timeout is doubled (exponential backoff) up to a maximum of 60 seconds. After 10 consecutive retransmission failures, the stream is reset.

### 4.5 Receive Window and Flow Control

The receiver advertises a receive window (available buffer space) in each ACK. The sender MUST NOT send more data than the receive window allows.

```
bytes_in_flight = send_next_seq - ack_seq
if bytes_in_flight >= recv_window:
    block_until_window_update()
```

Default receive window: **1 MB** per stream
Minimum receive window: **64 KB**
Maximum receive window: **16 MB** (configurable)

---

## 5. Datagram Stream (DATAGRAM type)

Datagram streams provide unreliable, unordered delivery. Each DATA message on a datagram stream is an independent unit — there is no sequencing, no acknowledgment, and no retransmission. Datagram behavior is **domain-agnostic**.

### 5.1 Use Cases

- DNS queries (already stateless)
- VoIP/video (where retransmission is more harmful than loss)
- Gaming (real-time, old packets are useless)
- Custom UDP protocols

### 5.2 Datagram Framing

```
DATA payload {
    stream_id:  uint32
    flags:      uint16  (SYN or FIN to open/close; no other flags meaningful)
    data:       bytes
}
```

No stream_sequence field. No ACK mechanism. No flow control.

### 5.3 Datagram Size Limit

Each datagram must fit in a single BGP-X packet. Maximum datagram payload: **1160 bytes** (leaving room for headers and encryption overhead within the 1280-byte MTU target).

---

## 6. Head-of-Line Blocking Prevention

Multiplexing over a shared path introduces the risk of head-of-line (HOL) blocking: a large data transfer on one stream can delay smaller, latency-sensitive messages on other streams.

BGP-X prevents HOL blocking through:

### 6.1 Interleaved Transmission

The client's packet scheduler interleaves packets from different streams rather than sending all packets from one stream before moving to the next. The WFQ (Weighted Fair Queuing) scheduler ensures all streams get regular transmission slots.

### 6.2 Stream Priority

Streams can be assigned a priority level (0 = highest, 7 = lowest, default = 3):

| Priority | Intended Use |
|---|---|
| 0 | Control messages, latency-critical |
| 1–2 | Interactive traffic (DNS, small API calls) |
| 3–4 | General web traffic (default) |
| 5–6 | Background transfers |
| 7 | Bulk data (large file transfers) |

The scheduler sends higher-priority stream packets first within each scheduling quantum.

### 6.3 Maximum Segment Size

To prevent large streams from monopolizing the path, each packet from any stream is limited to the session MTU (1160 bytes of application data). Large data transfers are automatically segmented.

---

## 7. ECH and Stream Opening

When STREAM_OPEN includes ECH flags, the exit node evaluates Encrypted Client Hello support.

**ECH Evaluation Logic (at Exit Node):**

1. Check `ech_capable = true` in exit node's own configuration.
2. Check if the destination supports ECH (via DNS HTTPS record).
3. If `ECH_REQUIRED` flag is set:
   - If ECH fails: send `STREAM_CLOSE` with reason `ERR_ECH_REQUIRED_NOT_SUPPORTED`.
4. If `ECH_PREFERRED` flag is set:
   - If ECH fails: retry without ECH (one retry).
   - Set flags in first response DATA: `ECH_NEGOTIATED=0`, `ECH_AVAILABLE=1`.
5. If ECH succeeds:
   - Set flags in first response DATA: `ECH_NEGOTIATED=1`, `ECH_AVAILABLE=1`.

**Cross-Domain Note**: ECH is only evaluated at the final clearnet exit hop. If the path traverses mesh segments before exiting to clearnet, ECH is still evaluated at the exit — unchanged behavior. If the final destination is a mesh island service (no clearnet exit), ECH is not applicable.

---

## 8. Maximum Streams Per Session

Default maximum concurrent open streams per session: **1024**

This limit is configurable by:
- Node operators (global cap)
- Applications (per-session cap via SDK PathConfig)

---

## 9. Multiplexing at the Exit Node

The exit node manages multiplexing between the BGP-X overlay side and the clearnet side.

### 9.1 Stream-to-Connection Mapping

Each BGP-X stream maps to one outbound clearnet connection (TCP connection or UDP socket) at the exit node:

```
bgpx_stream_id → OutboundConnection {
    protocol:   TCP | UDP,
    remote_ip:  IpAddr,
    remote_port: u16,
    local_port:  u16,     // ephemeral, assigned by OS
    state:      CONNECTING | CONNECTED | CLOSING | CLOSED
}
```

### 9.2 Connection Establishment

When a STREAM_OPEN arrives at the exit node, it immediately begins establishing the outbound clearnet connection. By the time the first DATA arrives, the connection is often already established (reducing perceived latency).

### 9.3 Response Demultiplexing

When data arrives from a clearnet destination, the exit node must route it back to the correct BGP-X stream. It uses the local port number (assigned by OS to each outbound connection) as the key:

```
local_port → bgpx_stream_id → encode in return DATA message
```

### 9.4 Connection Pooling

For exit nodes serving many streams to the same destination (e.g., multiple clients accessing the same website), connection pooling MAY be used for HTTP/2 and HTTP/3 connections. Connection pooling is transparent to BGP-X clients — it is an exit-node optimization.

---

## 10. Store-and-Forward vs. PAUSED State

These are distinct mechanisms for handling intermittent connectivity.

| Mechanism | What It Handles | Configured By |
|---|---|---|
| **PAUSED state** | Temporary stream suspension within an active path | Automatic (transport-driven) |
| **Store-and-forward** | Queuing stream OPEN requests when island offline | SDK `StreamConfig.store_and_forward` |

- **PAUSED**: stream is open but momentarily cannot deliver; resumes automatically when transport is available.
- **Store-and-forward**: path construction is deferred; stream is not yet opened.

---

## 11. Cleanup on Session Close

When a session closes (for any reason):
- All open streams within that session MUST be immediately reset.
- Applications using the stream MUST be notified of the reset via the SDK callback mechanism.
- Outbound connections at the exit node MUST be closed.
- Stream state for all affected streams MUST be cleared.

---

## 12. Implementation Notes

### 12.1 Buffer Management

Per-stream buffers:
- **Send buffer**: holds unacknowledged data awaiting ACK (ordered streams only)
- **Receive buffer**: holds out-of-order data awaiting in-order delivery

Buffer allocation is lazy — buffers are allocated when the first data arrives and freed when the stream closes.

### 12.2 Stream State Persistence

Stream state is NOT persisted across process restarts. Sessions and their streams are ephemeral. Applications MUST handle stream resets gracefully and implement their own application-level reconnect logic if needed.

---

## 13. Multiplexing Metrics

| Metric | Description |
|---|---|---|
| `active_streams` | Current count of open streams across all sessions |
| `streams_opened_total` | Total streams opened since daemon start |
| `streams_closed_normal` | Streams closed with FIN (normal close) |
| `streams_closed_reset` | Streams closed with RST (abnormal close) |
| `streams_timed_out` | Streams closed due to inactivity timeout |
| `streams_paused` | Streams currently in PAUSED state |
| `streams_paused_by_domain` | Streams paused per domain type |
| `retransmissions_total` | Total packet retransmissions (ordered streams) |
| `max_concurrent_streams` | High watermark of concurrent open streams |
| `bytes_per_stream_histogram` | Distribution of total bytes per stream |
| `ech_negotiated_total` | Streams where ECH was successfully used |
| `ech_fallback_total` | Streams where ECH fell back to standard TLS |
| `store_and_forward_queued` | Streams currently queued in store-and-forward |
| `cross_domain_streams_active` | Streams currently traversing multiple routing domains |
