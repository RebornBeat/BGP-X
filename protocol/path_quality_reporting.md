# BGP-X Path Quality Reporting Specification

**Version**: 0.1.0-draft

---

## 1. Overview

Path quality reporting allows relay nodes to signal aggregate path quality metrics back to the client without leaking path composition. The domain identifier added in v1 enables clients to determine which routing domain segment is experiencing issues in cross-domain paths, enabling domain-level path segment rebuilds.

---

## 2. Report Structure

A PATH_QUALITY_REPORT payload is always exactly **20 bytes** (after encryption, plus 16-byte tag = 36 bytes ciphertext).

### Unencrypted Content (Before Encryption)

```
Bytes 0-7:   Domain ID of reporting node (8 bytes)
             type (4 bytes BE) + instance (4 bytes BE)
             Examples:
               Clearnet relay:        0x00000001-00000000
               Mesh relay (lima-1):   0x00000003-a1b2c3d4
               LoRa relay (eu-868):   0x00000004-bb220033

Byte 8:      hop_latency_bucket (domain-calibrated):
             Clearnet: 0x00=<50ms, 0x01=50-150ms, 0x02=150-300ms, 0x03=>300ms
             WiFi mesh: 0x00=<10ms, 0x01=10-50ms, 0x02=50-100ms, 0x03=>100ms
             LoRa: 0x00=<200ms, 0x01=200ms-1s, 0x02=1-3s, 0x03=>3s
             Satellite LEO: 0x00=<30ms, 0x01=30-50ms, 0x02=50-100ms, 0x03=>100ms

Byte 9:      congestion_flag:
             0x00 = no congestion
             0x01 = mild (output queue 75-90% full)
             0x02 = severe (output queue >90% full)

Bytes 10-19: padding (random, for size normalization)
```

### Encryption

```
Key:    session_key[hop_i]
Nonce:  report_sequence_number (separate sequence space from data packets)
AAD:    path_id || hop_position_byte

Plaintext:  20 bytes
Ciphertext: 20 bytes + 16 bytes Poly1305 tag = 36 bytes payload
```

Total PATH_QUALITY_REPORT packet: 32 bytes header + 36 bytes payload = **68 bytes**.

Only the client can decrypt this — encrypted with the per-hop session key shared between client and that specific relay. Intermediate relays see only encrypted bytes.

---

## 3. Who Generates Reports

Exit nodes always generate PATH_QUALITY_REPORT when extension bit 3 is accepted.

Relay nodes generate when:
- Extension flag bit 3 is set for this session
- They detect congestion (output queue depth)
- They are computing latency statistics

Domain bridge nodes generate one report per domain they serve (one for the clearnet side, one for the mesh side), each with the appropriate domain_id.

---

## 4. Report Propagation on Return Path

PATH_QUALITY_REPORT messages travel on the return path using path_id routing — identical to single-domain paths, including crossing domain boundaries at bridge nodes.

```
Mesh relay generates report with domain_id = mesh:lima-district-1
    → encrypted for client (only client can decrypt)
    │
    ▼  (path_id lookup → forward toward client)
Mesh relay → bridge node (via mesh radio, opaque)
    │
    ▼  (cross-domain path_id lookup → clearnet predecessor)
Bridge node → clearnet relay → ... → client (opaque, no decryption at any intermediate)
    │
    ▼
Client decrypts → reads domain_id = mesh:lima-district-1 → identifies which segment reported
```

Intermediate relays and bridge nodes do NOT decrypt PATH_QUALITY_REPORT. This maintains path composition privacy.

---

## 5. Client Report Aggregation

The client maintains per-hop quality records with domain tagging:

```rust
struct HopQualityRecord {
    hop_position: u8,
    domain_id: DomainId,         // Which routing domain this hop is in
    latency_bucket: u8,
    congestion_flag: u8,
    sample_count: u32,
    rolling_average_bucket: f32,
}
```

### Per-Domain Calibrated Quality Score

```
quality_score[hop_i] = 0.60 × latency_score(bucket, domain_id) + 0.40 × congestion_score(flag)

latency_score by domain:
  clearnet: [1.0, 0.7, 0.4, 0.1] per bucket [0, 1, 2, 3]
  wifi_mesh: [1.0, 0.8, 0.5, 0.2]  (tighter tolerance; WiFi should be fast)
  lora: [1.0, 0.9, 0.7, 0.4]       (wide tolerance; LoRa expected high latency)
  satellite: [1.0, 0.8, 0.5, 0.2]
```

### Domain-Level Rebuild Decision

```
if quality_score[hop] < 0.20 AND domain_id.type == mesh:
    → attempt segment-level rebuild for that mesh segment only
    → keep clearnet segments intact if possible

if quality_score[hop] < 0.20 AND domain_id.type == clearnet:
    → attempt segment-level rebuild for that clearnet segment
    → keep mesh segments intact if possible

if quality_score < 0.20 at bridge node position:
    → attempt to find alternative bridge node for same domain transition
    → if no alternative: full path rebuild

if overall path quality < 0.20:
    → immediate full path rebuild regardless of domain
```

---

## 6. Path Rebuild Triggers from PQR

| Condition | Action |
|---|---|
| Any hop shows bucket=0x03 for 3+ consecutive reports | Begin pre-building replacement path |
| Any hop shows congestion_flag=0x02 (severe) for 30+ seconds | Reduce send rate by 50% |
| Overall path quality < 0.20 | Immediate path rebuild |
| Any hop shows congestion_flag=0x02 for 60+ seconds | Immediate path rebuild |
| Specific domain segment degraded (domain_id identifies it) | Domain-level segment rebuild attempt |

---

## 7. PQR Privacy Properties

Does NOT reveal:
- Individual node identities (domain_id type+instance only — same for all nodes in that island)
- Path composition (intermediate relays see only encrypted bytes, including across domain boundaries)
- Per-connection timing at fine granularity (buckets only)
- Cross-domain path structure beyond what client already knows from construction

Does reveal to client only:
- Coarse latency bucket at each hop position (4 values, domain-calibrated)
- Congestion status at each hop position (3 values)
- Which routing domain each hop is in (domain_id — client already knows this from path construction)

---

## 8. Extension Negotiation

PQR enabled when extension flag bit 3 accepted in handshake. Domain-agnostic — same negotiation in all routing domains. If a node does not support PQR (bit 3 not accepted), client falls back to end-to-end RTT estimation via KEEPALIVE timing.
