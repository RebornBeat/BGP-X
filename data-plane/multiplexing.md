# BGP-X Stream Multiplexing Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X multiplexes multiple streams over a single path. Cross-domain paths maintain the same stream multiplexing model — streams are continuous across domain boundaries.

---

## 2. Stream Concepts

### Stream Definition

A stream is a logical bidirectional channel within a BGP-X session. Identified by a 32-bit Stream ID within the session.

### Stream ID Assignment (MANDATORY)

- **Client-initiated streams**: ODD IDs (1, 3, 5, ...)
- **Service-initiated streams**: EVEN IDs (2, 4, 6, ...)

Violation: sender receives RST, error logged internally. Applies in all routing domains.

### Cross-Domain Stream Continuity

A BGP-X stream maintains its stream_id across all domain boundaries in a cross-domain path. The stream is NOT re-identified at domain transitions.

```
Stream ID 7 (client-initiated, odd)
  → in clearnet segment: stream_id = 7
  → at domain bridge: stream_id = 7
  → in mesh segment: stream_id = 7
  → at service: stream_id = 7
```

One stream, one stream_id, one socket — regardless of how many domains the path traverses.

### Stream Types

| Type | Delivery | Use Cases |
|---|---|---|
| ORDERED | Reliable, ordered (like TCP) | HTTP, API calls, file transfers |
| DATAGRAM | Unreliable, unordered (like UDP) | DNS, VoIP, gaming |

### path_id Association

Each stream is associated with a specific path (via path_id). When path is rebuilt (including cross-domain rebuilds), path_id changes. In-flight streams may be migrated to new path.

---

## 3. Stream Lifecycle

```
IDLE → OPEN → HALF_CLOSED_LOCAL → CLOSED
           └→ HALF_CLOSED_REMOTE → CLOSED
           └→ RESET (immediate close)
           └→ PAUSED (store-and-forward for mesh)
```

### PAUSED State (Cross-Domain Specific)

For mesh transports with duty cycle or intermittent connectivity:

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

- For LoRa duty cycle: stream pauses during duty cycle wait; resumes when duty cycle resets
- For offline mesh island: stream pauses; resumes when island bridge node comes online
- Application receives `STREAM_PAUSED` event (not an error) indicating temporary suspension

---

## 4. Ordered Stream (TCP-like)

Per-stream sequence numbers (uint32, random start). Cumulative ACK. Retransmission timeout: `max(200ms, 2 × RTT)`. Flow control via receive window in each ACK. All domain-agnostic.

---

## 5. Datagram Stream (UDP-like)

No ordering, no ACK, no retransmission. Maximum payload: **1160 bytes** (fits in single BGP-X packet). Domain-agnostic.

---

## 6. Head-of-Line Blocking Prevention

WFQ scheduler interleaves packets from different streams. Stream priorities 0-7 (0 = highest, 7 = lowest, default 3). Maximum segment size: session MTU (1160 bytes application data). Domain-agnostic.

---

## 7. ECH and Cross-Domain Streams

ECH is evaluated at the clearnet exit hop. If the path traverses mesh segments before exiting to clearnet, ECH is still evaluated at the final clearnet exit — unchanged behavior.

If the final destination is a mesh island service (no clearnet exit), ECH is not applicable.

---

## 8. Maximum Streams Per Session (Unchanged)

Default maximum: **1024** concurrent open streams per session.

---

## 9. Store-and-Forward vs. PAUSED State

| Mechanism | What It Handles | Configured By |
|---|---|---|
| PAUSED state | Temporary stream suspension within active path | Automatic (transport-driven) |
| Store-and-forward (path-level) | Queuing stream OPEN requests when island offline | SDK `StreamConfig.store_and_forward` |

These are distinct:
- PAUSED: stream is open but momentarily cannot deliver; resumes automatically
- Store-and-forward: path construction deferred; stream not yet opened

---

## 10. Multiplexing at the Exit Node (Unchanged)

Each BGP-X stream maps to one outbound clearnet connection. Domain-agnostic.

---

## 11. Cleanup on Session Close (Unchanged)

All open streams immediately reset when session closes. Applications notified. Outbound connections at exit closed. Domain-agnostic.

---

## 12. Metrics (Updated)

| Metric | Description |
|---|---|
| active_streams | Current open streams across all sessions |
| streams_opened_total | Total streams since daemon start |
| streams_closed_normal | Streams closed with FIN |
| streams_closed_reset | Streams closed with RST |
| streams_timed_out | Streams closed due to inactivity |
| streams_paused | Streams currently in PAUSED state |
| streams_paused_by_domain | Streams paused per domain type |
| ech_negotiated_total | Streams where ECH was used |
| ech_fallback_total | Streams where ECH fell back to standard TLS |
| cross_domain_streams_active | Streams traversing multiple routing domains |
