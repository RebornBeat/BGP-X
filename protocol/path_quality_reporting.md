# BGP-X Path Quality Reporting Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X Path Quality Reporting (PQR) extension — the mechanism by which relay nodes report aggregate path quality metrics back to the client without leaking path composition. In v1, PQR is extended to include domain identification, enabling clients to determine which routing domain segment is experiencing issues in cross-domain paths.

---

## 1. Overview

Path quality reporting allows relay nodes to signal aggregate path quality metrics back to the client:

- Approximate hop latency (coarse bucket, not precise timing)
- Congestion status (none, mild, severe)
- Which routing domain the hop is in (domain_id — added in v1)

This information helps the client make path rebuild decisions without requiring the client to infer quality solely from end-to-end measurements. The domain identifier enables **domain-level rebuilds** — replacing only the degraded segment of a cross-domain path rather than the entire path.

PQR is designed with strict privacy constraints:

- Reports are encrypted for the client only (using client's session key)
- Intermediate relays forward reports opaquely (no decryption)
- Reports contain NO node identifiers
- Reports contain NO path composition information
- Reports are coarse (buckets, not precise measurements)
- Domain ID reveals only the routing domain type and instance (e.g., "mesh:lima-district-1"), which the client already knows from path construction

---

## 2. Report Structure

A PATH_QUALITY_REPORT message payload is always exactly **20 bytes** (after encryption, plus 16-byte Poly1305 tag = 36 bytes ciphertext). The fixed size ensures no timing or content information leakage.

### 2.1 Unencrypted Content (Before Encryption)

```
Byte offset:
  0-7       Domain ID of reporting node (8 bytes)
            Format: type (4 bytes BE) + instance hash (4 bytes BE)

            Examples:
              Clearnet relay:              0x00000001-00000000
              Clearnet-satellite-LEO:       0x00000001-00000000 (same domain; satellite latency calibration)
              BGP-X overlay relay:          0x00000002-00000000
              Mesh relay (lima-1):          0x00000003-a1b2c3d4
              LoRa regional (eu-868):       0x00000004-bb220033
              Reserved (not active):        0x00000005-* (future BGP-X satellite; ignore if received)

  8         hop_latency_bucket (domain-calibrated, see Section 2.2)

  9         congestion_flag:
            0x00 = no congestion
            0x01 = mild congestion (output queue 75-90% full)
            0x02 = severe congestion (output queue >90% full)

  10-19     padding (random, for size normalization)
```

Total unencrypted payload: **20 bytes**.

### 2.2 Latency Bucket Calibration by Domain

Latency buckets are calibrated per routing domain type. This acknowledges that acceptable latency varies dramatically between clearnet fiber, WiFi mesh, LoRa radio, and satellite links.

| Domain Type | Bucket 0x00 | Bucket 0x01 | Bucket 0x02 | Bucket 0x03 |
|---|---|---|---|---|
| **Clearnet (fiber/cellular)** | <50ms | 50-150ms | 150-300ms | >300ms |
| **Clearnet-satellite-LEO** (Starlink, Kuiper) | <30ms | 30-50ms | 50-100ms | >100ms |
| **Clearnet-satellite-GEO** (Inmarsat, HughesNet) | <300ms | 300-600ms | 600-1000ms | >1000ms |
| **WiFi Mesh** | <10ms | 10-50ms | 50-100ms | >100ms |
| **LoRa** (any SF) | <200ms | 200ms-1s | 1-3s | >3s |
| **Bluetooth BLE** | <20ms | 20-50ms | 50-100ms | >100ms |

**Note**: LoRa latency varies significantly with spreading factor (SF7 ~50ms per hop, SF12 ~2-5 seconds per hop). The bucket ranges accommodate all SF values. A LoRa relay at SF12 reporting bucket 0x02 (1-3s) may be normal for that configuration.

### 2.3 Encryption

The report is encrypted with the client's session key for this specific hop:

```
Key:    session_key[hop_i]   (the key shared between client and this relay)
Nonce:  report_sequence_number (separate sequence space from data packets)
AAD:    path_id || hop_position_byte

Plaintext:  20 bytes
Ciphertext: 20 bytes + 16 bytes Poly1305 tag = 36 bytes payload
```

Total PATH_QUALITY_REPORT packet: 32 bytes common header + 36 bytes payload = **68 bytes**.

Only the client can decrypt this — intermediate relays see only encrypted bytes. Intermediate relays and domain bridge nodes do NOT decrypt PATH_QUALITY_REPORT at any point, including across domain boundaries.

---

## 3. Who Generates Reports

### Exit Nodes

Exit nodes always generate PATH_QUALITY_REPORT when the extension is enabled (extension flag bit 3 accepted). They have the best visibility into congestion — they observe the connection to the clearnet destination.

### Relay Nodes

Relay nodes generate PATH_QUALITY_REPORT when:

- Extension flag bit 3 is set for this session
- They detect congestion (output queue depth)
- They are computing latency statistics

### Domain Bridge Nodes

Domain bridge nodes generate **one report per domain they serve**:

- A bridge serving clearnet + mesh generates two reports:
  - One with domain_id = clearnet (for the clearnet-side hop)
  - One with domain_id = mesh:\<island_id\> (for the mesh-side hop)

Each report is encrypted independently with the appropriate session key for that hop.

---

## 4. Report Generation by Relay Nodes

### 4.1 Latency Bucket Measurement

Relay nodes measure their local processing latency (time from packet receipt to forwarding):

```
hop_latency = time_forwarded - time_received

hop_latency_bucket (using domain-calibrated thresholds):
  0x00 if hop_latency < threshold_0
  0x01 if threshold_0 <= hop_latency < threshold_1
  0x02 if threshold_1 <= hop_latency < threshold_2
  0x03 if hop_latency >= threshold_2
```

Thresholds are selected from Section 2.2 based on the node's domain_id.

**Note**: This measures processing + queuing latency at this hop, not network RTT. Network latency between hops is not measurable by the relay node itself.

### 4.2 Congestion Flag

```
output_queue_depth = current_pending_packets_for_next_hop

congestion_flag:
  0x00 if output_queue_depth < 0.75 × MAX_QUEUE_DEPTH
  0x01 if 0.75 × MAX_QUEUE_DEPTH <= output_queue_depth < 0.90 × MAX_QUEUE_DEPTH
  0x02 if output_queue_depth >= 0.90 × MAX_QUEUE_DEPTH
```

MAX_QUEUE_DEPTH is implementation-defined (default: 1000 packets). The congestion flag reflects instantaneous state — it may change rapidly.

---

## 5. Report Propagation on Return Path

PATH_QUALITY_REPORT messages travel on the return path using path_id routing — identical to single-domain paths, including crossing domain boundaries at bridge nodes.

### 5.1 Single-Domain Path

```
Relay node generates report → encrypts for client
         │
         ▼  (path_id lookup → forward toward client)
Previous relay node → forward opaquely (no decryption)
         │
         ▼
Entry node → forward opaquely (no decryption)
         │
         ▼
Client receives encrypted report → decrypts with session_key[hop_i]
```

### 5.2 Cross-Domain Path

```
Mesh relay generates report with domain_id = mesh:lima-district-1
    → encrypted for client (only client can decrypt)
    │
    ▼  (path_id lookup → forward toward client via mesh transport)
Mesh relay → domain bridge node (via mesh radio, opaque)
    │
    ▼  (cross-domain path_id lookup → clearnet predecessor)
Domain bridge node → clearnet relay → ... → entry node
    │
    ▼  (all forwarding is opaque, no decryption at any intermediate)
Client receives encrypted report → decrypts
    → reads domain_id = mesh:lima-district-1
    → identifies which segment reported
```

**Key property**: Intermediate relays and bridge nodes do NOT decrypt PATH_QUALITY_REPORT at any point. This maintains path composition privacy across all domain boundaries.

---

## 6. Client Report Aggregation

The client maintains per-hop quality records with domain tagging:

```rust
struct HopQualityRecord {
    hop_position: u8,
    domain_id: DomainId,              // Which routing domain this hop is in
    latency_bucket: u8,
    congestion_flag: u8,
    sample_count: u32,
    rolling_average_bucket: f32,
    last_report_timestamp: SystemTime,
}
```

### 6.1 Per-Domain Calibrated Quality Score

```
quality_score[hop_i] = 0.60 × latency_score(bucket, domain_id) + 0.40 × congestion_score(flag)
```

Latency score varies by domain (accounting for expected latency differences):

```
latency_score by domain:

clearnet:        [1.0, 0.7, 0.4, 0.1]  per bucket [0, 1, 2, 3]
satellite-LEO:   [1.0, 0.8, 0.5, 0.2]  (tighter tolerance for LEO)
satellite-GEO:   [1.0, 0.9, 0.7, 0.5]  (wide tolerance; GEO expected high latency)
wifi_mesh:       [1.0, 0.8, 0.5, 0.2]  (tighter tolerance; WiFi should be fast)
lora:            [1.0, 0.9, 0.7, 0.4]  (wide tolerance; LoRa expected high latency)
ble:             [1.0, 0.8, 0.5, 0.2]

congestion_score (all domains):
  1.0 if flag = 0x00 (none)
  0.5 if flag = 0x01 (mild)
  0.0 if flag = 0x02 (severe)
```

### 6.2 Overall Path Quality Score

```
path_quality = min(quality_score[hop_i] for all hops)
# Overall quality limited by worst hop
```

Rolling average over last 10 reports per hop. Exponential decay with half-life of 60 seconds.

### 6.3 Domain-Level Rebuild Decision

Cross-domain paths enable domain-segment-level rebuilds:

```
if quality_score[hop] < 0.20 AND domain_id.type == mesh:
    → attempt segment-level rebuild for that mesh segment only
    → keep clearnet segments intact if possible
    → do NOT rebuild the entire path

if quality_score[hop] < 0.20 AND domain_id.type == clearnet:
    → attempt segment-level rebuild for that clearnet segment
    → keep mesh segments intact if possible
    → do NOT rebuild the entire path

if quality_score < 0.20 at bridge node position:
    → attempt to find alternative bridge node for same domain transition
    → if alternative available: replace bridge node, keep rest of path
    → if no alternative: full path rebuild

if overall path quality < 0.20:
    → immediate full path rebuild regardless of domain
```

**Benefit**: A degraded LoRa segment does not force rebuilding the clearnet segments. A congested clearnet relay does not force rebuilding the mesh segments.

---

## 7. Path Rebuild Triggers from PQR

| Condition | Action |
|---|---|
| Any hop shows bucket=0x03 for 3+ consecutive reports | Begin pre-building replacement path |
| Any hop shows congestion_flag=0x02 (severe) sustained for 30+ seconds | Reduce send rate by 50% |
| Overall path quality < 0.20 | Immediate path rebuild |
| Any hop shows congestion_flag=0x02 for 60+ seconds | Immediate path rebuild |
| Specific domain segment degraded (domain_id identifies it) | Domain-level segment rebuild attempt |
| Bridge node hop quality < 0.20 | Find alternative bridge node for same domain pair |

---

## 8. PQR Privacy Properties

### What PQR Does NOT Reveal

- **Individual node identities**: domain_id type+instance only — same for all nodes in that island/domain
- **Path composition**: intermediate relays see only encrypted bytes, including across domain boundaries
- **Per-connection timing at fine granularity**: buckets only (4 values per domain)
- **Cross-domain path structure beyond what client already knows**: domain_id matches the path construction

### What PQR DOES Reveal to Client Only

- Coarse latency bucket at each hop position (4 values, domain-calibrated)
- Congestion status at each hop position (3 values)
- Which routing domain each hop is in (domain_id — client already knows this from path construction)

This information is only available to the client (encrypted with per-hop session keys). It is not available to any intermediate party, including domain bridge nodes.

---

## 9. Geographic Plausibility and PQR

Geographic plausibility scoring is a separate mechanism from PQR. However, PQR's domain-calibrated latency buckets inform the client about hop latency, which may correlate with geographic distance.

**Important distinction**:

- **Geographic plausibility**: RTT-based reputation signal; OPTIONAL; applies only if node declares jurisdiction
- **PQR latency buckets**: congestion and processing latency measurement; always enabled when extension accepted

PQR measures hop processing latency, not network RTT between hops. PQR does not replace or substitute for geographic plausibility scoring.

### Satellite Nodes and Geo Plausibility

Satellite-connected clearnet nodes (latency_class = satellite-*) are **exempt from geographic plausibility RTT scoring**. This exemption does not affect PQR — these nodes still generate PATH_QUALITY_REPORT like any other clearnet relay. The exemption applies to the separate geo plausibility reputation signal, not to congestion or latency reporting.

---

## 10. Satellite Internet Clarification

Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat, OneWeb, Kuiper) are **clearnet domain (0x00000001)**.

A satellite-connected node reports:

```
domain_id: 0x00000001-00000000 (clearnet)
latency_bucket: calibrated to satellite class (see Section 2.2)
```

Domain type 0x00000005 (bgpx-satellite) is RESERVED for future BGP-X-native satellite infrastructure and is NOT used in current deployments. Any PATH_QUALITY_REPORT claiming domain_id type 0x00000005 MUST be ignored by the client.

---

## 11. Extension Negotiation

PQR is enabled when:

- Extension flag bit 3 is set in HANDSHAKE_INIT
- Node accepts bit 3 in HANDSHAKE_RESP (set in accepted extensions)
- Session is established with PQR flag set on both sides

PQR is **domain-agnostic** — the same negotiation occurs in all routing domains (clearnet, mesh, satellite WAN).

If a node does not support PQR (extension bit 3 not accepted):

- Client does not expect PATH_QUALITY_REPORT from that node
- Client falls back to end-to-end RTT estimation via KEEPALIVE timing
- Path quality decisions use KEEPALIVE round-trip times instead

---

## 12. Implementation Requirements

Implementations MUST:

- Generate PATH_QUALITY_REPORT only when extension bit 3 is accepted
- Use domain-calibrated latency bucket thresholds (Section 2.2)
- Encrypt reports with the correct per-hop session key
- Include the correct domain_id for the hop's routing domain
- Forward received PATH_QUALITY_REPORT opaquely via path_id routing
- NOT decrypt PATH_QUALITY_REPORT at intermediate nodes (including bridge nodes)
- Maintain per-hop rolling averages for quality scoring
- Apply domain-level rebuild logic when domain_id identifies a degraded segment

Implementations MUST NOT:

- Include node identifiers in PATH_QUALITY_REPORT
- Use latency thresholds from a different domain type
- Decrypt or modify PATH_QUALITY_REPORT at any intermediate hop
- Log the contents of PATH_QUALITY_REPORT

Implementations SHOULD:

- Use the domain-level rebuild logic to minimize path disruption
- Pre-build replacement segments when quality degrades
- Account for satellite-class latency in quality scoring
