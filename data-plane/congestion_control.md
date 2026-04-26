# BGP-X Congestion Control Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X congestion control extends to cross-domain paths. The domain identifier in PATH_QUALITY_REPORT enables clients to identify congestion by routing domain segment, enabling domain-level path rebuilds rather than full path rebuilds.

---

## 2. Congestion Detection (Domain-Aware)

### Client-Side Detection

**RTT increase** via KEEPALIVE timing:

```python
def get_congestion_baseline(path):
    max_latency_domain = max(
        path.domain_segments,
        key=lambda seg: get_domain_baseline_latency(seg.domain)
    )
    return get_domain_baseline_latency(max_latency_domain.domain)

def get_domain_baseline_latency(domain):
    if domain.type == "clearnet":   return 200   # ms
    if domain.type == "wifi_mesh":  return 50    # ms
    if domain.type == "lora":       return 5000  # ms (LoRa is inherently high latency)
    if domain.type == "satellite":  return 600   # ms (GEO) or 50ms (LEO)
    return 200  # Default

# MILD congestion trigger: measured_rtt > 2.0 × baseline
# SEVERE congestion trigger: measured_rtt > 4.0 × baseline
```

Using domain-specific baselines prevents false SEVERE triggers on paths with LoRa segments.

**Packet loss** via sequence number gaps:
```
MILD if loss_rate_30s > 2%
SEVERE if loss_rate_30s > 10%
```

### Node-Side Detection and Signaling

Relay nodes detect congestion via output queue depth:
```
output_queue_depth > 75% of MAX_QUEUE_DEPTH → congestion_flag = MILD
output_queue_depth > 90% → SEVERE; begin tail drop
output_queue_depth < 50% → CLEAR
```

PATH_QUALITY_REPORT (now including domain_id) carries the signal to the client. Intermediate relays and bridge nodes forward opaquely — congestion signals do not leak path composition to any observer.

---

## 3. Domain-Level Congestion Identification

```python
def identify_congested_domain(pqr_reports):
    for report in pqr_reports:
        if report.congestion_flag == SEVERE:
            return report.domain_id  # Pinpoints which domain

    for report in pqr_reports:
        if report.congestion_flag == MILD:
            return report.domain_id

    return None  # No congestion
```

---

## 4. Congestion Response

### MILD Congestion
- Reduce send rate by 20% (multiplicative decrease)
- Hold reduced rate for minimum 5 seconds before increasing
- Log congestion event (no session identifiers)

### SEVERE Congestion
- Reduce send rate by 50%
- Pause new stream openings for 10 seconds
- Evaluate path quality for rebuild
- If SEVERE sustained for >30 seconds: trigger path rebuild

### Domain-Level Rebuild Decision

```
if congested_domain.type == "mesh":
    → attempt mesh segment rebuild only
    → keep clearnet segments intact if possible

if congested_domain.type == "clearnet" (intermediate segment):
    → attempt clearnet segment rebuild
    → keep mesh segments intact if possible

if congested at bridge position:
    → select alternative bridge node for same domain transition
    → rebuild segments on both sides if needed

if overall quality < 0.20 regardless of domain:
    → immediate full path rebuild
```

### Rate Recovery

After congestion clears:
```
every RTT without congestion:
    send_rate += min(10% of current_rate, 1 Mbps)
    send_rate = min(send_rate, max_configured_rate)
```

---

## 5. Cover Traffic and Congestion (Domain-Aware)

```
cover_traffic_rate[domain] = max(
    MIN_COVER_RATE_FOR_DOMAIN[domain],
    target_cover_rate × (1 - congestion_level_in_domain)
)

MIN_COVER_RATE:
  clearnet:  10 Kbps
  wifi_mesh: 5 Kbps
  lora:      0 Kbps  (duty cycle is already the binding constraint)
```

Cover traffic uses session_key (not a separate key). Applies identically in all domains.

---

## 6. Effective Bandwidth for Cross-Domain Paths

```python
def compute_effective_bandwidth(path):
    return min(
        estimate_segment_bandwidth(seg)
        for seg in path.domain_segments
    )
```

The effective throughput of a cross-domain path is limited by the bottleneck segment. For a path through clearnet (1 Gbps) + LoRa mesh (50 Kbps), effective bandwidth is 50 Kbps. Congestion control rate must not exceed the bottleneck.

---

## 7. Stream-Level Pacing (Unchanged)

Per-stream token bucket rate limiting, inter-packet gap calculation, WFQ scheduling — all unchanged. Domain-agnostic.

---

## 8. LoRa Duty Cycle as Hard Bandwidth Cap

LoRa duty cycle limits (1% in EU) act as a hard bandwidth cap independent of congestion control. BGP-X automatically respects duty cycle limits via token bucket. When duty cycle is reached: transmission pauses; streams enter PAUSED state; resume when duty cycle resets.

---

## 9. Metrics (Updated)

| Metric | Description |
|---|---|
| current_rtt_ms | Current measured round-trip time |
| baseline_rtt_ms | Domain-calibrated baseline at path creation |
| path_quality_score | Current overall score (0.0-1.0) |
| congestion_events_total | Total congestion events |
| domain_congestion_by_segment | Which domain segment shows congestion |
| cover_traffic_rate_kbps_by_domain | Current cover rate per domain |
| bottleneck_domain | Current bandwidth-limiting domain segment |
| lora_duty_cycle_remaining_pct | LoRa duty cycle remaining (mesh paths) |
