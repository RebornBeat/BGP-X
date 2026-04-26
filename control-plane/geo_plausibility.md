# BGP-X Geographic Plausibility Scoring

**Version**: 0.1.0-draft

---

## 1. Overview

Geographic plausibility scoring uses RTT measurements to verify node region claims. No external database required — physics provides the constraint: the speed of light sets a minimum propagation time between geographic locations.

Domain-specific thresholds apply for different routing domains. A LoRa relay with 2-second RTT is not suspicious; a clearnet relay with 2-second RTT is highly suspicious.

---

## 2. RTT Measurement Procedure

RTT measurements are collected from KEEPALIVE exchanges:

```
1. Record time_sent when KEEPALIVE is transmitted to a node
2. Record time_received when KEEPALIVE response arrives
3. RTT_sample = time_received - time_sent
4. Apply to rolling average: RTT[node_id] = exponential_moving_average(RTT_sample, alpha=0.1)
```

Minimum samples before scoring: **5 RTT measurements**. Before 5 measurements: return neutral score (0.5).

RTT measurements MUST be constant-time (not timing-channel-leaking).

Outlier rejection: discard samples >5× current average (likely network anomaly).

---

## 3. Score Computation

```python
def compute_geo_plausibility_score(node, client_region, domain):

    # Domain-specific exemptions
    if domain.type == "satellite":
        return 0.5  # Neutral — satellite location is orbital, not geographic

    if is_lora_mesh_node(node, domain):
        return compute_mesh_lora_plausibility(node)

    if is_wifi_mesh_node(node, domain):
        return compute_mesh_wifi_plausibility(node)

    if node.rtt_measurement_count < 5:
        return 0.5  # Neutral — insufficient measurements

    measured_rtt = RTT[node.node_id]
    (expected_min, expected_max) = get_expected_rtt(client_region, node.region)

    ratio = measured_rtt / expected_max

    if ratio <= 1.0:   return 1.0   # Within expected range
    elif ratio <= 1.5: return 0.7   # Slightly above (acceptable variance)
    elif ratio <= 2.0: return 0.4   # Notably above (suspicious)
    elif ratio <= 3.0: return 0.2   # Significantly above (highly suspicious)
    else:              return 0.0   # Far above (implausible)
```

---

## 4. Per-Region RTT Threshold Table

Expected one-way RTT ranges (milliseconds). Clients measure round-trip; compare against 2× one-way:

| From ↓ / To → | NA | EU | AP | SA | AF | ME |
|---|---|---|---|---|---|---|
| NA | 5-50 | 50-120 | 100-200 | 40-100 | 150-250 | 100-200 |
| EU | 50-120 | 5-50 | 100-200 | 80-150 | 50-150 | 40-100 |
| AP | 100-200 | 100-200 | 5-50 | 150-250 | 180-280 | 100-180 |
| SA | 40-100 | 80-150 | 150-250 | 5-50 | 180-280 | 150-250 |
| AF | 150-250 | 50-150 | 180-280 | 180-280 | 5-50 | 50-100 |
| ME | 100-200 | 40-100 | 100-180 | 150-250 | 50-100 | 5-50 |

"Expected max" for score computation = column maximum × 2 (round-trip).

Example: NA client → node claiming EU: Expected RTT 100-240ms. Score threshold applied to `measured_rtt / 240ms`.

---

## 5. Domain-Specific Plausibility

### Clearnet Nodes

Use standard internet RTT thresholds from table above. Unchanged.

### WiFi Mesh Island Nodes

```python
def compute_mesh_wifi_plausibility(node):
    # Expected RTT within mesh island: 1-20ms per hop
    if node.latency_ms < 20:    return 1.0
    elif node.latency_ms < 50:  return 0.7
    elif node.latency_ms < 100: return 0.4
    else:                       return 0.2
```

### LoRa Mesh Island Nodes

```python
def compute_mesh_lora_plausibility(node):
    # LoRa RTT: 100ms-5000ms per hop — high latency is expected
    if node.latency_ms < 5000:    return 1.0  # Within LoRa range: fully plausible
    elif node.latency_ms < 15000: return 0.5  # Slow but possible (multi-hop LoRa)
    else:                         return 0.1  # Suspiciously slow even for LoRa
```

### Domain Bridge Nodes

Domain bridge nodes are checked independently for each domain they serve:

```python
def score_bridge_geo_plausibility(node, from_domain, to_domain, client_region):
    clearnet_score = compute_geo_plausibility_score(node, client_region, from_domain)
    bridge_latency = node.get_bridge(from_domain, to_domain).bridge_latency_ms
    bridge_score = compute_bridge_latency_plausibility(bridge_latency, from_domain, to_domain)
    return (clearnet_score + bridge_score) / 2.0

def compute_bridge_latency_plausibility(latency_ms, from_domain, to_domain):
    # WiFi mesh bridge: fast
    if to_domain.type == "mesh_wifi":
        return 1.0 if latency_ms < 50 else 0.5 if latency_ms < 200 else 0.2
    # LoRa bridge: inherently slower
    if to_domain.type == "mesh_lora":
        return 1.0 if latency_ms < 500 else 0.7 if latency_ms < 2000 else 0.4
    # Clearnet bridge: internet latency
    return compute_geo_plausibility_score_from_rtt(latency_ms, from_domain, to_domain)
```

### Satellite Nodes

Exempt from geographic plausibility scoring. Return neutral score 0.5.

---

## 6. Reputation Events Generated

| Condition | Event | Score Change |
|---|---|---|
| RTT within expected range | GEO_PLAUSIBLE | +0.3 |
| RTT 1.5-2× expected max | GEO_SUSPICIOUS | -1.5 |
| RTT 2-3× expected max | GEO_IMPLAUSIBLE | -3.0 |
| RTT >3× expected max | GEO_IMPLAUSIBLE (severe) | -3.0 per occurrence |
| ≥3 GEO_IMPLAUSIBLE within 7 days | PROTOCOL_VIOLATION_GEO | -10.0 |

For mesh and bridge nodes: domain-tagged events (e.g., GEO_SUSPICIOUS with domain = mesh:lima-district-1).

---

## 7. Attack Resistance

**Adversary adds RTT delay**: makes node appear LESS trustworthy (higher RTT than expected). Score decreases; node selected less. Self-defeating attack.

**Adversary reduces RTT delay**: impossible below speed-of-light limits. A node in Asia claiming EU region cannot respond in <100ms from EU — physically prevented.

**Fake RTT records**: reputation database is local-first. Adversary cannot inject fake RTT measurements without compromising the client device.

---

## 8. Score Integration with Path Selection

```
path_score_component = w_geo × S_geo_plausibility(node, domain) = 0.10 × score
```

At default weights: a fully plausible node contributes 0.10; an implausible node contributes 0.00. This is meaningful but not dominating. A node with excellent uptime and reputation but slightly suspicious RTT is still selected — just with lower probability.

---

## 9. Implementation Notes

- RTT measurements MUST be constant-time
- Exponential moving average with α = 0.1 (slow-moving; resistant to single outlier)
- Outlier rejection: >5× current average discarded
- Storage: per-NodeID in memory; persisted to reputation database
- Thread safety: atomic updates or per-node locks
- Minimum 5 measurements before scoring (return 0.5 neutral for new nodes)

---

## 10. Test Vectors

1. NA client → EU node, RTT 140ms: ratio = 140/240 = 0.58 → score = 1.0
2. NA client → EU node, RTT 400ms: ratio = 400/240 = 1.67 → score = 0.7
3. NA client → EU node, RTT 600ms: ratio = 600/240 = 2.5 → score = 0.4
4. NA client → EU node, RTT 900ms: ratio = 900/240 = 3.75 → score = 0.0
5. LoRa mesh node, RTT 1500ms: compute_mesh_lora_plausibility → 1.0
6. Satellite node: exempt → 0.5
7. < 5 measurements: neutral → 0.5
8. Same-region node, RTT 5ms: ratio = 5/100 = 0.05 → score = 1.0
9. WiFi mesh node, RTT 8ms: compute_mesh_wifi_plausibility → 1.0
10. WiFi mesh node, RTT 75ms: compute_mesh_wifi_plausibility → 0.4
