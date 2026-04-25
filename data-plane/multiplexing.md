# BGP-X Stream Multiplexing Specification

**Version**: 0.1.0-draft

This document specifies how BGP-X multiplexes multiple logical streams over a single overlay path, including stream lifecycle, ordering, flow control, and reassembly.

---

## 1. Overview

BGP-X supports **stream multiplexing** — the ability to carry multiple independent logical connections over a single BGP-X path simultaneously.

### 1.1 Why multiplexing matters

Without multiplexing:
- Each connection (HTTP request, DNS query, API call) requires its own path
- Path setup (handshake × N hops) cost is paid for every connection
- Many simultaneous connections mean many independent circuit constructions, each potentially observable

With multiplexing:
- One path handles all connections in a session window
- Path setup cost is paid once
- Connection patterns across streams are harder to observe individually
- A browser loading a web page (dozens of connections) uses one BGP-X path

### 1.2 Comparison to Tor

Tor creates one circuit per TCP stream. A web page load requiring 30 TCP connections creates 30 Tor circuits (or shares circuits in ways that reduce anonymity). BGP-X multiplexes all 30 connections over a single path, improving both efficiency and anonymity.

---

## 2. Stream Concepts

### 2.1 Stream definition

A **stream** is a logical bidirectional channel within a BGP-X session. Streams are identified by a 32-bit **Stream ID** within their session.

Stream IDs are assigned by the initiating party (client for client-initiated streams, service for service-initiated streams). Client stream IDs use odd numbers; service stream IDs use even numbers (preventing collision without coordination).

### 2.2 Stream types

| Type | Description | Use Cases |
|---|---|---|
| ORDERED | Reliable, ordered delivery (like TCP) | HTTP, API calls, file transfers |
| DATAGRAM | Unreliable, unordered (like UDP) | DNS queries, VoIP, gaming |

Stream type is specified at stream open time and cannot be changed.

### 2.3 Maximum streams per session

Default maximum concurrent open streams per session: **1024**

This limit is configurable by both node operators (global cap) and by application (per-session cap).

---

## 3. Stream Lifecycle

```
         Client                                    Server / Exit
            │                                           │
            │──── STREAM_OPEN (stream_id=1) ───────────►│
            │                                           │
            │     [Server accepts stream]               │
            │                                           │
            │◄─── DATA (stream_id=1, SYN flag) ─────────│
            │                                           │
            │     [Data exchange]                       │
            │                                           │
            │──── DATA (stream_id=1, FIN flag) ─────────►│
            │                                           │
            │◄─── DATA (stream_id=1, FIN flag) ──────────│
            │                                           │
            │     [Stream CLOSED]                       │
```

### 3.1 Stream states

```
IDLE → OPEN → HALF_CLOSED_LOCAL → CLOSED
           └→ HALF_CLOSED_REMOTE → CLOSED
           └→ RESET (immediate close, no drain)
```

| State | Description |
|---|---|
| IDLE | Stream ID not yet used |
| OPEN | Both parties can send and receive |
| HALF_CLOSED_LOCAL | Local party sent FIN; can still receive |
| HALF_CLOSED_REMOTE | Remote party sent FIN; can still send |
| CLOSED | Both parties have sent FIN; no more data |
| RESET | Stream abruptly closed (RST flag) |

### 3.2 Stream Open

A stream is opened by sending a STREAM_OPEN message (Msg Type 0x06) or a DATA message with the SYN flag set.

STREAM_OPEN payload contains the destination address, allowing the exit node to establish the outbound connection before any data arrives. This reduces latency for the first data exchange.

```
STREAM_OPEN {
    stream_id:       uint32  (client-initiated: odd; service-initiated: even)
    stream_type:     uint8   (0x01 = ORDERED, 0x02 = DATAGRAM)
    destination_type: uint8  (0x04 = IPv4, 0x06 = IPv6, 0x0A = domain, 0xBB = ServiceID)
    destination:     bytes   (address)
    port:            uint16
    flags:           uint16
}
```

### 3.3 Stream Close

A stream is closed by sending a DATA message with the FIN flag set. After sending FIN, the sender MUST NOT send any more data on this stream. It MAY continue to receive data until it receives FIN from the remote party.

A stream is reset by sending a DATA message with the RST flag set. Both parties MUST immediately stop sending on this stream and discard any buffered data.

### 3.4 Stream timeout

A stream with no activity for 5 minutes (no data, no keepalives specific to the stream) SHOULD be closed with a normal FIN. The session itself remains open.

---

## 4. Ordered Stream (ORDERED type)

Ordered streams provide reliable, sequenced delivery semantics equivalent to TCP.

### 4.1 Per-stream sequence numbers

Ordered streams maintain independent per-stream sequence numbers (separate from the session-level sequence numbers used for replay protection):

```
stream_sequence_number: uint32  (wraps at 2^32)
```

Stream sequence numbers start at a random value chosen by the sender during STREAM_OPEN.

### 4.2 Data framing

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

### 4.5 Receive window and flow control

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

Datagram streams provide unreliable, unordered delivery. Each DATA message on a datagram stream is an independent unit — there is no sequencing, no acknowledgment, and no retransmission.

### 5.1 Use cases

- DNS queries (already stateless)
- VoIP/video (where retransmission is more harmful than loss)
- Gaming (real-time, old packets are useless)
- Custom UDP protocols

### 5.2 Datagram framing

```
DATA payload {
    stream_id:  uint32
    flags:      uint16  (SYN or FIN to open/close; no other flags meaningful)
    data:       bytes
}
```

No stream_sequence field. No ACK mechanism. No flow control.

### 5.3 Datagram size limit

Each datagram must fit in a single BGP-X packet. Maximum datagram payload: **1160 bytes** (leaving room for headers and encryption overhead within the 1280-byte MTU target).

---

## 6. Head-of-Line Blocking Prevention

Multiplexing over a shared path introduces the risk of head-of-line (HOL) blocking: a large data transfer on one stream can delay smaller, latency-sensitive messages on other streams.

BGP-X prevents HOL blocking through:

### 6.1 Interleaved transmission

The client's packet scheduler interleaves packets from different streams rather than sending all packets from one stream before moving to the next. The WFQ scheduler (described in `/data-plane/congestion_control.md`) ensures all streams get regular transmission slots.

### 6.2 Stream priority

Streams can be assigned a priority level (0 = highest, 7 = lowest, default = 3):

| Priority | Intended Use |
|---|---|
| 0 | Control messages, latency-critical |
| 1–2 | Interactive traffic (DNS, small API calls) |
| 3–4 | General web traffic (default) |
| 5–6 | Background transfers |
| 7 | Bulk data (large file transfers) |

The scheduler sends higher-priority stream packets first within each scheduling quantum.

### 6.3 Maximum segment size

To prevent large streams from monopolizing the path, each packet from any stream is limited to the session MTU (1160 bytes of application data). Large data transfers are automatically segmented.

---

## 7. Stream ID Management

### 7.1 ID assignment

Stream IDs are 32-bit unsigned integers.

- Client-initiated streams: odd IDs (1, 3, 5, ...)
- Server/service-initiated streams: even IDs (2, 4, 6, ...)
- Stream ID 0: reserved (not used)

The client starts at stream ID 1 and increments by 2 for each new stream. The server starts at stream ID 2 and increments by 2.

### 7.2 ID exhaustion

After 2^31 streams have been opened (all 32-bit odd or even IDs used), the session MUST be torn down and a new session established. In practice, 2^31 streams per session will never be reached in normal use.

### 7.3 Reuse policy

Stream IDs MUST NOT be reused within a session. Once a stream is CLOSED or RESET, its ID is retired and never reused on the same session.

---

## 8. Multiplexing at the Exit Node

The exit node manages multiplexing between the BGP-X overlay side and the clearnet side.

### 8.1 Stream-to-connection mapping

Each BGP-X stream maps to one outbound clearnet connection (TCP connection or UDP socket) at the exit node:

```
bgpx_stream_id → clearnet_connection {
    protocol:   TCP | UDP
    remote_ip:  IPAddress
    remote_port: uint16
    local_port:  uint16  (ephemeral, assigned by OS)
    state:       CONNECTING | CONNECTED | CLOSING | CLOSED
}
```

### 8.2 Connection establishment

When a STREAM_OPEN arrives at the exit node, it immediately begins establishing the outbound clearnet connection. By the time the first DATA arrives, the connection is often already established (reducing perceived latency).

### 8.3 Response demultiplexing

When data arrives from a clearnet destination, the exit node must route it back to the correct BGP-X stream. It uses the local port number (assigned by OS to each outbound connection) as the key:

```
local_port → bgpx_stream_id → encode in return DATA message
```

### 8.4 Connection pooling

For exit nodes serving many streams to the same destination (e.g., multiple clients accessing the same website), connection pooling MAY be used for HTTP/2 and HTTP/3 connections. Connection pooling is transparent to BGP-X clients — it is an exit-node optimization.

---

## 9. Implementation Notes

### 9.1 Buffer management

Per-stream buffers:
- Send buffer: holds unacknowledged data awaiting ACK (ordered streams only)
- Receive buffer: holds out-of-order data awaiting in-order delivery

Buffer allocation is lazy — buffers are allocated when the first data arrives and freed when the stream closes.

### 9.2 Cleanup on session close

When a session closes (for any reason), all open streams within that session MUST be immediately reset. Applications using the stream MUST be notified of the reset via the SDK callback mechanism.

### 9.3 Stream state persistence

Stream state is NOT persisted across process restarts. Sessions and their streams are ephemeral. Applications MUST handle stream resets gracefully and implement their own application-level reconnect logic if needed.

---

## 10. Multiplexing Metrics

| Metric | Description |
|---|---|
| `active_streams` | Current count of open streams across all sessions |
| `streams_opened_total` | Total streams opened since daemon start |
| `streams_closed_normal` | Streams closed with FIN (normal close) |
| `streams_closed_reset` | Streams closed with RST (abnormal close) |
| `streams_timed_out` | Streams closed due to inactivity timeout |
| `retransmissions_total` | Total packet retransmissions (ordered streams) |
| `max_concurrent_streams` | High watermark of concurrent open streams |
| `bytes_per_stream_histogram` | Distribution of total bytes per stream |
