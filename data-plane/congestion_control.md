# BGP-X Congestion Control Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X congestion control addresses challenges unique to overlay networks:

- **Opaque intermediate hops**: client cannot directly observe congestion at relay nodes
- **Multi-hop paths**: congestion at one hop affects the entire path
- **Privacy constraints**: congestion feedback must not leak path information
- **Cross-domain paths**: different domains have fundamentally different latency baselines
- **Latency amplification**: 20ms queuing per hop = 80ms for 4-hop path

BGP-X combines:
1. **End-to-end rate control** at the client (stream-level pacing)
2. **Per-node backpressure** signaling via PATH_QUALITY_REPORT (encrypted for client only, with domain_id)
3. **Path quality monitoring** with automatic domain-level and full-path rebuild
4. **Domain-calibrated baselines**: different thresholds for clearnet, WiFi mesh, LoRa, and satellite

---

## 2. Congestion Detection

### 2.1 Client-Side Detection

**RTT increase** via KEEPALIVE timing (domain-calibrated baseline):

```python
def get_congestion_baseline_ms(path):
    # Use the maximum expected latency domain as the baseline
    # to avoid false positives on high-latency segments
    max_latency_domain = max(
        path.domain_segments,
        key=lambda seg: DOMAIN_BASELINE_LATENCY_MS[seg.domain.type]
    )
    return DOMAIN_BASELINE_LATENCY_MS[max_latency_domain.domain.type]

DOMAIN_BASELINE_LATENCY_MS = {
    "clearnet":   200,    # ms — standard internet baseline
    "wifi_mesh":  50,     # ms — WiFi mesh should be fast
    "lora":       5000,   # ms — LoRa is inherently high latency
    "satellite":  600,    # ms — GEO satellite; or 50ms for LEO
}

# MILD trigger: measured_rtt > 2.0 × domain_calibrated_baseline
# SEVERE trigger: measured_rtt > 4.0 × domain_calibrated_baseline
```

Using domain-specific baselines prevents false SEVERE triggers on paths with LoRa segments.

**Packet loss** via sequence number gaps:

```
MILD if loss_rate_30s > 2%
SEVERE if loss_rate_30s > 10%
```

**Transmission timeout**:

```
timeout = max(200ms, 4 × moving_average_rtt)

if stream.time_since_last_ack > timeout:
    trigger_congestion(TIMEOUT)
```

### 2.2 Node-Side Detection and Signaling

Relay nodes detect local congestion via output queue depth:

```
if output_queue_depth > 0.75 × MAX_QUEUE_DEPTH:
    congestion_flag = MILD

if output_queue_depth > 0.90 × MAX_QUEUE_DEPTH:
    congestion_flag = SEVERE
    begin_tail_drop()

if output_queue_depth < 0.50 × MAX_QUEUE_DEPTH:
    congestion_flag = CLEAR
```

### 2.3 Congestion Signal Propagation

Congestion signals travel to the client via PATH_QUALITY_REPORT (now including domain_id) on the return path. Intermediate relays and bridge nodes forward PATH_QUALITY_REPORT opaquely via path_id routing — no decryption at any intermediate node, including across domain boundaries.

Only the client can decrypt PATH_QUALITY_REPORT (uses per-hop session key). The congestion signal does not leak path composition to any observer.

### 2.4 Domain-Level Congestion Identification

```python
def identify_congested_domain(pqr_reports):
    # Reports from all hops on the path, each tagged with domain_id
    for report in pqr_reports:
        if report.congestion_flag == SEVERE:
            return report.domain_id  # Pinpoints which routing domain

    for report in pqr_reports:
        if report.congestion_flag == MILD:
            return report.domain_id

    return None  # No congestion
```

---

## 3. Congestion Response

### 3.1 MILD Congestion

```
- Reduce send rate by 20% (multiplicative decrease)
- Hold reduced rate for minimum 5 seconds before increasing
- Log congestion event (no session identifiers)
```

### 3.2 SEVERE Congestion

```
- Reduce send rate by 50%
- Pause new stream openings for 10 seconds
- Evaluate path quality for domain-level or full rebuild
- If SEVERE sustained for >30 seconds: trigger rebuild
```

### 3.3 TIMEOUT Congestion

```
- Halt transmission immediately
- Wait for RTT measurement to stabilize
- If RTT stabilizes: resume at 25% of pre-congestion rate
- If RTT does not stabilize within 60 seconds: rebuild path
```

### 3.4 Rate Recovery

After congestion clears (CONGESTION_CLEAR or sustained low RTT):

```
every RTT without congestion:
    send_rate += min(10% of current_rate, 1 Mbps)
    send_rate = min(send_rate, max_configured_rate)
```

Additive increase, multiplicative decrease (AIMD) adapted for overlay context.

---

## 4. Domain-Level Rebuild Decision

When PATH_QUALITY_REPORT identifies which domain_id is congested:

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

---

## 5. Path Quality Scoring

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

### Path Quality Rebuild Thresholds

| Quality Score | Action |
|---|---|
| ≥ 0.80 | Healthy — no action |
| 0.50 – 0.80 | Warning — monitor closely, increase sampling |
| 0.20 – 0.50 | Pre-build a replacement path in background |
| < 0.20 | Immediate path rebuild |

### Path Rebuild Triggers from PQR Reports

| Condition | Action |
|---|---|
| Any hop shows bucket=0x03 (degraded) for 3+ consecutive reports | Begin pre-building replacement path |
| Any hop shows congestion_flag=0x02 (severe) sustained for 30+ seconds | Reduce send rate 50% |
| Overall path quality < 0.20 | Immediate path rebuild |
| Any hop shows congestion_flag=0x02 sustained for 60+ seconds | Immediate path rebuild |
| Specific domain segment degraded (domain_id identifies it) | Domain-level segment rebuild attempt |

---

## 6. Stream-Level Pacing

### 6.1 Token Bucket

Per-stream token bucket rate limiting:

```
bucket_capacity = 2 × RTT × send_rate   // 2 RTT of burst allowed
token_refill_rate = send_rate
token_consumption = packet_size per packet
```

### 6.2 Inter-Packet Gap

For latency-sensitive streams:

```
gap_ms = (packet_size_bytes × 8) / (send_rate_bps / 1000)
```

---

## 7. Multi-Stream Bandwidth Sharing

Weighted Fair Queue (WFQ) at the client ensures no single stream monopolizes the path.

### 7.1 Stream Priorities

| Priority | Intended Use |
|---|---|
| 0 | Control messages, latency-critical (BGP-X protocol traffic) |
| 1-2 | Interactive traffic (DNS, small API calls) |
| 3-4 | General web traffic (default) |
| 5-6 | Background transfers |
| 7 | Bulk data (large file transfers) |

Default priority: 3. Applications may specify via SDK `StreamConfig.priority`.

### 7.2 Equal Weight Default

All streams equal weight by default. Higher-priority streams get proportionally more bandwidth when the path is saturated. Lower-priority streams are rate-limited but never completely starved.

### 7.3 Cross-Domain Effective Bandwidth

```python
def compute_effective_bandwidth(path):
    return min(
        estimate_segment_bandwidth(seg)
        for seg in path.domain_segments
    )
```

The effective throughput of a cross-domain path is limited by its bottleneck segment. For a path through clearnet (1 Gbps) + LoRa (50 Kbps), effective bandwidth is 50 Kbps. WFQ respects this limit — streams do not attempt to exceed the bottleneck domain's capacity.

---

## 8. Cover Traffic and Congestion

When cover traffic is enabled:

```python
cover_traffic_rate_kbps = max(
    MIN_COVER_RATE_BY_DOMAIN[domain_type],
    target_cover_rate_kbps × (1 - congestion_level)
)

MIN_COVER_RATE_BY_DOMAIN = {
    "clearnet":   10,    # Kbps — always send some
    "wifi_mesh":  5,     # Kbps
    "lora":       0,     # Kbps — LoRa duty cycle is already the binding constraint
    "satellite":  5,     # Kbps
}
```

During SEVERE congestion: cover traffic reduced to minimum (MIN_COVER_RATE) to prioritize real traffic.

**Note**: cover traffic uses session_key (not a separate key). Congestion reduction affects cover rate but not the key used — COVER packets remain externally indistinguishable from RELAY regardless of rate adjustments.

---

## 9. LoRa Duty Cycle as Hard Bandwidth Cap

LoRa duty cycle limits (1% in EU) act as a hard bandwidth cap independent of congestion control.

BGP-X LoRa implementation respects duty cycle via token bucket:

```
EU duty cycle: 1% → ~36 seconds transmit per hour at SF7
At SF7 (~200 bytes, ~250ms airtime per packet):
  → ~144 packets per hour maximum

Token bucket:
  capacity = duty_cycle_budget_per_interval  // e.g., 36 seconds per hour
  refill_rate = duty_cycle_fraction          // 1% per second budget
  cost_per_packet = airtime_seconds          // ~0.25s for SF7 200-byte frame
```

When duty cycle is reached: transmission pauses; streams in LoRa domain enter PAUSED state; resume when duty cycle resets.

---

## 10. Satellite Path Congestion

Satellite paths (especially GEO with 600ms RTT) require special handling:

- KEEPALIVE timeout extended to 5 minutes (configurable)
- Congestion baseline: satellite-class RTT (not internet baseline)
- RTT recovery slower — use longer smoothing window
- Path quality scoring uses satellite-calibrated latency buckets

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
