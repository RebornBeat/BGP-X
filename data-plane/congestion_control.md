# BGP-X Congestion Control Specification

**Version**: 0.1.0-draft

This document specifies the congestion control mechanisms used in the BGP-X data plane to manage bandwidth, prevent queue buildup, and maintain quality of service across multi-hop paths spanning one or more routing domains.

---

## 1. Overview

Congestion control in BGP-X addresses challenges unique to overlay networks:

- **Opaque intermediate hops**: client cannot directly observe congestion at relay nodes — it can only infer from end-to-end measurements
- **Multi-hop paths**: congestion at any single hop affects the entire path, but the client cannot pinpoint which hop is bottlenecked
- **Privacy constraints**: congestion feedback signals must not leak information about traffic content or path composition
- **Cross-domain paths**: different routing domains have fundamentally different latency baselines (clearnet vs. LoRa vs. satellite)
- **Latency amplification**: in a 4-hop path, a 50ms queuing delay at each hop produces 200ms of additional latency

BGP-X combines:

1. **End-to-end rate control** at the client (stream-level pacing)
2. **Per-node backpressure** signaling via PATH_QUALITY_REPORT (encrypted for client only, with domain_id)
3. **Path quality monitoring** with automatic domain-level and full-path rebuild
4. **Domain-calibrated baselines**: different RTT thresholds for clearnet, WiFi mesh, LoRa, and satellite

---

## 2. Congestion Detection

### 2.1 Client-Side Detection

The client detects congestion through multiple signals, using domain-calibrated baselines to avoid false positives on high-latency segments.

#### Round-Trip Latency Increase

The client measures the round-trip time (RTT) of KEEPALIVE exchanges. Sustained RTT increase indicates queuing along the path.

**Domain-Calibrated Baselines**

Different routing domains have fundamentally different latency characteristics. Using a single baseline would cause false SEVERE triggers on paths with LoRa segments.

```python
DOMAIN_BASELINE_LATENCY_MS = {
    "clearnet":   200,    # ms — standard internet baseline
    "wifi_mesh":  50,     # ms — WiFi mesh should be fast
    "lora":       5000,   # ms — LoRa is inherently high latency
    "satellite":  600,    # ms — GEO satellite; or 50ms for LEO
}

def get_congestion_baseline_ms(path):
    """
    Use the maximum expected latency domain as the baseline
    to avoid false positives on high-latency segments.
    """
    max_latency_domain = max(
        path.domain_segments,
        key=lambda seg: DOMAIN_BASELINE_LATENCY_MS[seg.domain.type]
    )
    return DOMAIN_BASELINE_LATENCY_MS[max_latency_domain.domain.type]
```

**Congestion Thresholds**

```python
# MILD congestion trigger
if moving_average_rtt > 2.0 × domain_calibrated_baseline:
    trigger_congestion_event(MILD)

# SEVERE congestion trigger
if moving_average_rtt > 4.0 × domain_calibrated_baseline:
    trigger_congestion_event(SEVERE)
```

Baseline RTT is established during the first 10 KEEPALIVE exchanges after path construction, using the domain-calibrated value as the initial baseline and refining based on actual measurements.

#### Packet Loss

Packet loss is inferred from gaps in the return-path sequence number space:

```python
# MILD congestion trigger
if loss_rate_30s > 0.02:  # 2%
    trigger_congestion_event(MILD)

# SEVERE congestion trigger
if loss_rate_30s > 0.10:  # 10%
    trigger_congestion_event(SEVERE)
```

#### Transmission Timeout

If a stream does not receive an acknowledgment within the expected window:

```python
timeout = max(200ms, 4 × moving_average_rtt)

if stream.time_since_last_ack > timeout:
    trigger_congestion_event(TIMEOUT)
```

### 2.2 Node-Side Detection and Signaling

Relay nodes detect local congestion through output queue depth:

```python
# MILD congestion
if output_queue_depth > 0.75 × MAX_QUEUE_DEPTH:
    congestion_flag = MILD

# SEVERE congestion
if output_queue_depth > 0.90 × MAX_QUEUE_DEPTH:
    congestion_flag = SEVERE
    begin_tail_drop()

# Congestion cleared
if output_queue_depth < 0.50 × MAX_QUEUE_DEPTH:
    congestion_flag = CLEAR
```

### 2.3 Congestion Signal Propagation

Congestion signals travel to the client via PATH_QUALITY_REPORT messages on the return path.

**PATH_QUALITY_REPORT Structure** (20 bytes after decryption):

| Offset | Field | Size | Description |
|---|---|---|---|
| 0-1 | domain_id_type | 2 bytes | Routing domain type (0x0001=clearnet, 0x0003=mesh, 0x0004=lora-regional) |
| 2 | hop_latency_bucket | 1 byte | 0x00=<50ms, 0x01=50-150ms, 0x02=150-300ms, 0x03=>300ms |
| 3 | congestion_flag | 1 byte | 0x00=none, 0x01=mild, 0x02=severe |
| 4-15 | padding | 12 bytes | Random padding for size normalization |

**Propagation Path**:

1. Relay node generates PATH_QUALITY_REPORT with congestion_flag and domain_id
2. Report is encrypted using the client's session key for that hop
3. Report travels on the return path via path_id routing
4. Intermediate relays forward the report opaquely (no decryption)
5. Domain bridge nodes forward the report across domain boundaries opaquely
6. Only the client can decrypt (uses client's session key)

**Privacy Properties**:

- Congestion signals are encrypted within onion layers
- Intermediate relays (including bridge nodes) cannot read the report
- Congestion signals do not leak path composition to any observer
- The domain_id identifies which routing domain segment is affected, enabling targeted rebuilds

### 2.4 Domain-Level Congestion Identification

When the client receives PATH_QUALITY_REPORTs from multiple hops, it can identify which routing domain segment is congested:

```python
def identify_congested_domain(pqr_reports):
    """
    Reports from all hops on the path, each tagged with domain_id.
    Returns the domain_id of the congested segment, or None.
    """
    # Check SEVERE first
    for report in pqr_reports:
        if report.congestion_flag == SEVERE:
            return report.domain_id

    # Then check MILD
    for report in pqr_reports:
        if report.congestion_flag == MILD:
            return report.domain_id

    return None  # No congestion
```

This enables **domain-level rebuilds** instead of full path rebuilds when only one segment is congested.

---

## 3. Congestion Response

### 3.1 MILD Congestion

```
Actions:
  - Reduce send rate by 20% (multiplicative decrease)
  - Hold reduced rate for minimum 5 seconds before increasing
  - Log congestion event (no session identifiers)
  - Monitor for escalation
```

### 3.2 SEVERE Congestion

```
Actions:
  - Reduce send rate by 50%
  - Pause new stream openings for 10 seconds
  - Evaluate path quality for domain-level or full rebuild
  - If SEVERE sustained for >30 seconds: trigger rebuild
```

### 3.3 TIMEOUT Congestion

```
Actions:
  - Halt transmission immediately
  - Wait for RTT measurement to stabilize
  - If RTT stabilizes: resume at 25% of pre-congestion rate
  - If RTT does not stabilize within 60 seconds: rebuild path
```

### 3.4 Domain-Level Rebuild Decision

When PATH_QUALITY_REPORT identifies which domain_id is congested, the client can perform targeted rebuilds:

```python
if congested_domain.type == "mesh":
    # Attempt segment rebuild for that mesh segment only
    # Keep clearnet segments intact if possible
    attempt_domain_segment_rebuild(congested_domain)

elif congested_domain.type == "clearnet" (intermediate segment):
    # Attempt clearnet segment rebuild
    # Keep mesh segments intact if possible
    attempt_domain_segment_rebuild(congested_domain)

elif congested_at_bridge_position:
    # Select alternative bridge node for same domain transition
    attempt_bridge_node_rebuild(from_domain, to_domain)
    # If no alternative: rebuild segments on both sides

if overall_quality < 0.20 regardless_of_domain:
    # Immediate full path rebuild
    rebuild_entire_path()
```

**Benefits of Domain-Level Rebuilds**:

- Preserves established sessions in uncongested domains
- Reduces latency of rebuild (only rebuild affected segment)
- Minimizes disruption to cross-domain traffic
- Allows targeted selection of alternative nodes in the congested domain

### 3.5 Rate Recovery

After congestion clears (CONGESTION_CLEAR signal or sustained low RTT):

```python
every RTT_without_congestion:
    send_rate += min(10% of current_rate, 1 Mbps)
    send_rate = min(send_rate, max_configured_rate)
```

Additive increase, multiplicative decrease (AIMD) adapted for overlay context.

---

## 4. Path Quality Scoring

The congestion control module maintains a real-time path quality score used to decide when to rebuild a path.

### 4.1 Scoring Formula

```python
def compute_path_quality_score(path, pqr_reports):
    quality_score = (
        0.50 × latency_score(current_rtt, baseline_rtt) +
        0.30 × loss_score(current_loss_rate) +
        0.20 × throughput_score(actual_throughput, target_throughput)
    )
    return quality_score

def latency_score(current_rtt, baseline_rtt):
    return max(0, 1 - (current_rtt - baseline_rtt) / (5 * baseline_rtt))

def loss_score(loss_rate):
    return max(0, 1 - loss_rate / 0.20)

def throughput_score(actual, target):
    return min(1, actual / target) if target > 0 else 1.0
```

Score range: [0.0, 1.0]

### 4.2 Path Quality Rebuild Thresholds

| Quality Score | Action |
|---|---|
| ≥ 0.80 | Healthy — no action |
| 0.50 – 0.80 | Warning — monitor closely, increase sampling |
| 0.20 – 0.50 | Pre-build a replacement path in background |
| < 0.20 | Immediate path rebuild |

### 4.3 Path Rebuild Triggers from PATH_QUALITY_REPORT

| Condition | Action |
|---|---|
| Any hop shows bucket=0x03 (degraded) for 3+ consecutive reports | Begin pre-building replacement path |
| Any hop shows congestion_flag=0x02 (severe) sustained for 30+ seconds | Reduce send rate 50% |
| Overall path quality < 0.20 | Immediate path rebuild |
| Any hop shows congestion_flag=0x02 sustained for 60+ seconds | Immediate path rebuild |
| Specific domain segment degraded (domain_id identifies it) | Domain-level segment rebuild attempt |

---

## 5. Stream-Level Pacing

Individual streams are paced to avoid bursty transmission that causes queuing at relay nodes.

### 5.1 Token Bucket Pacing

Each stream uses a token bucket for rate limiting:

```python
bucket_capacity = 2 × RTT × send_rate  # Burst allowance = 2 RTT worth of data
token_refill_rate = send_rate
token_consumption = packet_size per packet
```

Packets are held until sufficient tokens are available. This prevents large bursts that would fill relay queues.

### 5.2 Inter-Packet Gap

For latency-sensitive streams, an explicit inter-packet gap can be configured:

```python
gap_ms = (packet_size_bytes × 8) / (send_rate_bps / 1000)
```

---

## 6. Multi-Stream Bandwidth Sharing

When multiple streams share a single path, bandwidth is shared fairly using a weighted fair queue (WFQ) at the client.

### 6.1 Stream Priorities

| Priority | Intended Use |
|---|---|
| 0 | Control messages, latency-critical (BGP-X protocol traffic) |
| 1-2 | Interactive traffic (DNS, small API calls) |
| 3-4 | General web traffic (default) |
| 5-6 | Background transfers |
| 7 | Bulk data (large file transfers) |

Default priority: 3. Applications may specify via SDK `StreamConfig.priority`.

### 6.2 Equal Weight Default

All streams have equal weight by default. Higher-priority streams get proportionally more bandwidth when the path is saturated. Lower-priority streams are rate-limited but never completely starved.

### 6.3 WFQ Scheduling Algorithm

```python
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

### 6.4 Cross-Domain Effective Bandwidth

```python
def compute_effective_bandwidth(path):
    """
    The effective throughput of a cross-domain path is limited by
    its bottleneck segment.
    """
    return min(
        estimate_segment_bandwidth(seg)
        for seg in path.domain_segments
    )
```

For a path through clearnet (1 Gbps) + LoRa (50 Kbps), effective bandwidth is 50 Kbps. WFQ respects this limit — streams do not attempt to exceed the bottleneck domain's capacity.

---

## 7. Cover Traffic and Congestion

When cover traffic is enabled, it competes for bandwidth with real traffic. The congestion control module adjusts cover traffic generation based on available capacity and domain type.

### 7.1 Cover Traffic Rate Calculation

```python
def compute_cover_traffic_rate(domain_type, congestion_level):
    MIN_COVER_RATE = {
        "clearnet":   10,    # Kbps — always send some
        "wifi_mesh":  5,     # Kbps
        "lora":       0,     # Kbps — LoRa duty cycle is already the binding constraint
        "satellite":  5,     # Kbps
    }

    MAX_COVER_RATE = 1000  # Kbps (1 Mbps cap)
    target_cover_rate = 100  # Kbps default

    return max(
        MIN_COVER_RATE[domain_type],
        min(
            MAX_COVER_RATE,
            target_cover_rate × (1 - congestion_level)
        )
    )
```

### 7.2 SEVERE Congestion Behavior

During SEVERE congestion: cover traffic reduced to MIN_COVER_RATE to prioritize real traffic.

### 7.3 Cover Traffic Key

**Important**: Cover traffic uses the **same session_key as RELAY packets**. There is no separate cover_key. Both use ChaCha20-Poly1305 encryption and are externally indistinguishable. Congestion reduction affects cover rate but not the key used.

---

## 8. LoRa Duty Cycle as Hard Bandwidth Cap

LoRa duty cycle limits (1% in EU) act as a hard bandwidth cap independent of congestion control. BGP-X automatically respects duty cycle limits.

### 8.1 Duty Cycle Token Bucket

```python
# EU duty cycle: 1% → ~36 seconds transmit per hour at SF7
# At SF7 (~200 bytes, ~250ms airtime per packet):
#   → ~144 packets per hour maximum

class LoRaDutyCycleEnforcer:
    def __init__(self, duty_cycle_fraction=0.01):
        self.bucket_capacity = duty_cycle_fraction * 3600  # seconds per hour
        self.refill_rate = duty_cycle_fraction             # per second
        self.current_budget = self.bucket_capacity
        self.last_refill = time.now()

    def can_transmit(self, airtime_seconds):
        self._refill()
        return self.current_budget >= airtime_seconds

    def consume(self, airtime_seconds):
        self.current_budget -= airtime_seconds

    def _refill(self):
        now = time.now()
        elapsed = now - self.last_refill
        self.current_budget = min(
            self.bucket_capacity,
            self.current_budget + elapsed * self.refill_rate
        )
        self.last_refill = now
```

### 8.2 Stream Behavior on Duty Cycle Exhaustion

When duty cycle is reached:

1. Transmission pauses
2. Streams in LoRa domain enter PAUSED state
3. Resume when duty cycle resets
4. No packets dropped — buffered until capacity available

---

## 9. Satellite Path Congestion

Satellite paths (especially GEO with 600ms RTT) require special handling:

### 9.1 Extended Timeouts

```python
# Standard KEEPALIVE interval: 25 seconds
# Satellite KEEPALIVE interval: up to 5 minutes (configurable)

# Standard session dead timeout: 90 seconds
# Satellite session dead timeout: up to 15 minutes (configurable)
```

### 9.2 Congestion Baseline

Satellite paths use satellite-class RTT baseline, not internet baseline. A 600ms RTT on a GEO link is normal; on WiFi mesh it would be severe congestion.

### 9.3 RTT Smoothing

RTT recovery is slower on satellite links. Use longer smoothing window:

```python
# Standard RTT smoothing: 10 samples
# Satellite RTT smoothing: 30 samples

satellite_rtt_ewma_alpha = 0.1  # Slower adaptation
```

### 9.4 Path Quality Scoring

Path quality scoring uses satellite-calibrated latency buckets:

```python
SATELLITE_LATENCY_BUCKETS = {
    "LEO": {  # Starlink, Kuiper, OneWeb
        0x00: "<30ms",
        0x01: "30-50ms",
        0x02: "50-100ms",
        0x03: ">100ms"
    },
    "GEO": {  # Inmarsat, HughesNet, Viasat
        0x00: "<300ms",
        0x01: "300-600ms",
        0x02: "600-1000ms",
        0x03: ">1000ms"
    }
}
```

---

## 10. Bandwidth Limits and Fairness

### 10.1 Per-Session Limits

Relay nodes enforce per-session bandwidth to prevent a single client from monopolizing capacity:

```python
max_bandwidth_per_session = min(
    total_node_bandwidth / max(active_sessions, 10),
    100 Mbps  # Cap
)
```

### 10.2 Total Node Cap

Node operators set `total_bandwidth_mbps` in configuration. Nodes approaching 90% of cap send CONGESTION_MILD via PATH_QUALITY_REPORT.

---

## 11. Metrics

| Metric | Description |
|---|---|
| current_rtt_ms | Current measured round-trip time |
| baseline_rtt_ms | Domain-calibrated baseline at path creation |
| packet_loss_rate | 30-second rolling loss rate |
| send_rate_mbps | Current paced send rate |
| path_quality_score | Current overall score (0.0-1.0) |
| congestion_events_total | Total congestion events since path creation |
| domain_congestion_by_segment | Which routing domain segment shows congestion |
| path_rebuilds_total | Total path rebuilds since daemon start |
| cover_traffic_rate_kbps_by_domain | Current cover rate per domain type |
| bottleneck_domain | Current bandwidth-limiting domain segment type |
| lora_duty_cycle_remaining_pct | LoRa duty cycle remaining (mesh paths) |
| effective_bandwidth_kbps | Actual effective bandwidth (bottleneck-limited) |
| domain_segment_rebuilds_total | Domain-level segment rebuilds |
| cross_domain_rebuilds_total | Full cross-domain path rebuilds |

All metrics are scoped to paths and streams — no user-identifying information is included.
