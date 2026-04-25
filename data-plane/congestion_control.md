# BGP-X Congestion Control Specification

**Version**: 0.1.0-draft

This document specifies the congestion control mechanisms used in the BGP-X data plane to manage bandwidth, prevent queue buildup, and maintain quality of service across multi-hop paths.

---

## 1. Overview

Congestion control in BGP-X is challenging for several reasons unique to overlay networks:

- **Opaque intermediate hops**: The client cannot directly observe congestion at relay nodes — it can only infer it from end-to-end measurements
- **Multi-hop path**: Congestion at any single hop affects the entire path, but the client cannot pinpoint which hop is bottlenecked
- **Privacy constraints**: Congestion feedback signals must not leak information about the traffic being carried
- **Latency amplification**: In a 4-hop path, a 50ms queuing delay at each hop produces 200ms of additional latency

BGP-X congestion control uses a combination of:

1. **End-to-end rate control** at the client (stream-level pacing)
2. **Per-node backpressure** signaling from relay to client
3. **Path quality monitoring** with automatic path rebuilding when congestion is severe

---

## 2. Congestion Detection

### 2.1 Client-side detection

The client detects congestion through:

#### Round-trip latency increase

The client measures the round-trip time (RTT) of KEEPALIVE exchanges. Sustained RTT increase indicates queuing along the path.

```
if moving_average_rtt > 2.0 × baseline_rtt:
    trigger_congestion_event(MILD)

if moving_average_rtt > 4.0 × baseline_rtt:
    trigger_congestion_event(SEVERE)
```

Baseline RTT is established during the first 10 KEEPALIVE exchanges after path construction.

#### Packet loss

Packet loss is inferred from gaps in the return-path sequence number space:

```
if loss_rate_30s > 0.02:  # 2%
    trigger_congestion_event(MILD)

if loss_rate_30s > 0.10:  # 10%
    trigger_congestion_event(SEVERE)
```

#### Transmission timeout

If a stream does not receive an acknowledgment within the expected window:

```
timeout = max(200ms, 4 × moving_average_rtt)

if stream.time_since_last_ack > timeout:
    trigger_congestion_event(TIMEOUT)
```

### 2.2 Node-side detection

Relay nodes detect local congestion through output queue depth:

```
if output_queue_depth > 0.75 × MAX_QUEUE_DEPTH:
    send_backpressure_signal(level=MILD)

if output_queue_depth > 0.90 × MAX_QUEUE_DEPTH:
    send_backpressure_signal(level=SEVERE)
    begin_tail_drop()
```

### 2.3 Backpressure signaling

Relay nodes signal congestion back toward the client using a lightweight mechanism embedded in the RELAY packet flags field:

| Flag Bit | Name | Meaning |
|---|---|---|
| 8 | CONGESTION_MILD | Node queue above 75% capacity |
| 9 | CONGESTION_SEVERE | Node queue above 90% capacity |
| 10 | CONGESTION_CLEAR | Congestion resolved (queue below 50%) |

These flags are set in the flags field of the layer header (visible only after decryption at the appropriate hop — the client sees these via the return path).

**Privacy note**: Congestion flags are encrypted within the onion layers and are not visible to outside observers. They do not leak information about traffic content or path structure.

---

## 3. Congestion Response

### 3.1 Client congestion response by severity

#### MILD congestion

```
action:
  - Reduce send rate by 20% (multiplicative decrease)
  - Hold reduced rate for minimum 5 seconds before increasing
  - Log congestion event (without session identifiers)
```

#### SEVERE congestion

```
action:
  - Reduce send rate by 50%
  - Pause new stream openings for 10 seconds
  - Begin evaluating path quality for rebuild
  - If SEVERE sustained for > 30 seconds: trigger path rebuild
```

#### TIMEOUT congestion

```
action:
  - Halt transmission immediately
  - Wait for RTT measurement to stabilize
  - If RTT stabilizes: resume at 25% of pre-congestion rate
  - If RTT does not stabilize within 60 seconds: rebuild path
```

### 3.2 Rate recovery

After congestion clears (CONGESTION_CLEAR signal or sustained low RTT), the send rate recovers using additive increase:

```
every RTT without congestion:
    send_rate += ADDITIVE_INCREASE_STEP

ADDITIVE_INCREASE_STEP = min(
    10% of current_send_rate,
    1 Mbps
)

send_rate = min(send_rate, max_configured_rate)
```

This is similar to TCP AIMD (Additive Increase, Multiplicative Decrease) but adapted for the overlay context.

---

## 4. Stream-Level Pacing

Individual streams are paced to avoid bursty transmission that causes queuing at relay nodes.

### 4.1 Token bucket pacing

Each stream uses a token bucket for rate limiting:

```
bucket_capacity    = 2 × RTT × send_rate  # Burst allowance = 2 RTT worth of data
token_refill_rate  = send_rate
token_consumption  = packet_size per packet
```

Packets are held until sufficient tokens are available. This prevents large bursts that would fill relay queues.

### 4.2 Inter-packet gap

For latency-sensitive streams, an explicit inter-packet gap can be configured:

```
gap_ms = (packet_size_bytes × 8) / (send_rate_bps / 1000)
```

---

## 5. Multi-Stream Bandwidth Sharing

When multiple streams share a single path, bandwidth is shared fairly using a weighted fair queue (WFQ) at the client.

### 5.1 Default weights

All streams have equal weight by default. Applications can request higher or lower weight via the SDK.

### 5.2 Scheduling algorithm

```
function select_next_packet_to_send(streams):

    # Find stream with largest deficit (WFQ)
    selected_stream = None
    max_deficit = -infinity

    for stream in active_streams:
        stream.deficit += stream.weight × quantum
        if stream.has_data() and stream.deficit > max_deficit:
            max_deficit = stream.deficit
            selected_stream = stream

    if selected_stream:
        packet = selected_stream.dequeue_packet()
        selected_stream.deficit -= packet.size
        return packet
```

---

## 6. Path Quality Scoring

The congestion control module maintains a real-time path quality score used to decide when to rebuild a path:

```
quality_score = (
    0.50 × latency_score(current_rtt, baseline_rtt) +
    0.30 × loss_score(current_loss_rate) +
    0.20 × throughput_score(current_throughput, target_throughput)
)
```

Where:

```
latency_score(rtt, baseline) = max(0, 1 - (rtt - baseline) / (5 × baseline))
loss_score(loss_rate) = max(0, 1 - (loss_rate / 0.20))
throughput_score(actual, target) = min(1, actual / target)
```

Score range: [0.0, 1.0]

Path rebuild triggers:

| Quality Score | Action |
|---|---|
| >= 0.80 | No action (healthy) |
| 0.50–0.80 | Log warning, monitor closely |
| 0.20–0.50 | Begin pre-building replacement path |
| < 0.20 | Immediate path rebuild |

---

## 7. Cover Traffic and Congestion

When cover traffic is enabled, it competes for bandwidth with real traffic. The congestion control module adjusts cover traffic generation based on available capacity:

```
cover_traffic_rate = max(
    MIN_COVER_RATE,
    min(
        MAX_COVER_RATE,
        target_cover_rate × (1 - congestion_level)
    )
)

MIN_COVER_RATE = 10 Kbps    # Always send some cover traffic
MAX_COVER_RATE = 1 Mbps     # Cap cover traffic impact
target_cover_rate = 100 Kbps  # Default target
```

During SEVERE congestion, cover traffic rate is reduced to MIN_COVER_RATE to prioritize real traffic.

---

## 8. Bandwidth Limits and Fairness

### 8.1 Node-level bandwidth limiting

Relay nodes enforce per-session bandwidth limits to prevent a single client from monopolizing the node's capacity:

```
max_bandwidth_per_session = total_node_bandwidth / max(active_sessions, 10)
max_bandwidth_per_session = min(max_bandwidth_per_session, 100 Mbps)
```

This ensures fairness among clients sharing a relay node.

### 8.2 Total node bandwidth cap

Node operators set a total bandwidth cap in configuration. The node monitors total throughput and sends CONGESTION_SEVERE signals when approaching 90% of the cap.

---

## 9. Metrics

| Metric | Description |
|---|---|
| `current_rtt_ms` | Current measured round-trip time |
| `baseline_rtt_ms` | Baseline RTT established at path creation |
| `packet_loss_rate` | 30-second rolling packet loss rate |
| `send_rate_mbps` | Current paced send rate |
| `path_quality_score` | Current path quality score (0.0–1.0) |
| `congestion_events_total` | Total congestion events since path creation |
| `path_rebuilds_total` | Total path rebuilds since client start |
| `cover_traffic_rate_kbps` | Current cover traffic send rate |

All metrics are scoped to paths and streams — no user-identifying information is included.
