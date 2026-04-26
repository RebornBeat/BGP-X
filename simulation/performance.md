# BGP-X Performance Specification and Benchmarks

**Version**: 0.1.0-draft

---

## 1. Performance Targets

### Core Relay Performance (Unchanged)

| Metric | Target | Minimum Acceptable |
|---|---|---|
| Per-packet forwarding latency (RELAY) | <1ms (p99) | <3ms (p99) |
| Maximum relay throughput (8 cores) | >1 Gbps | >100 Mbps |
| Maximum concurrent sessions | 10,000 | 1,000 |
| Session setup time (3-way handshake) | <50ms | <200ms |
| Path construction time (single domain, 4 hops) | <200ms (p95) | <1000ms (p99) |

### Cross-Domain Path Construction

| Metric | Target | Minimum Acceptable |
|---|---|---|
| 2-domain path (1 bridge), p95 | <800ms | <3000ms p99 |
| 3-domain path (2 bridges), p95 | <1200ms | <5000ms p99 |
| 4+ domain path, p95 | <1500ms | <8000ms p99 |
| Bridge node discovery (DHT, cache miss) | <300ms | <1000ms |
| Bridge node discovery (cached) | <5ms | <20ms |

### Domain Bridge Overhead

| Metric | Target | Minimum |
|---|---|---|
| DOMAIN_BRIDGE hop processing latency | <2ms (p99) | <5ms (p99) |
| Cross-domain path_id table lookup | <100ns (p99) | <500ns (p99) |
| Transport selection per domain | <50ns (p99) | <200ns (p99) |

### Cross-Domain Throughput by Path Type

| Domain Path | Expected Throughput | Limiting Factor |
|---|---|---|
| clearnet only (4 hops) | ~105 Mbps | Relay overhead |
| clearnet → WiFi mesh | >10 Mbps | WiFi mesh segment |
| clearnet → LoRa | <50 Kbps | LoRa duty cycle |
| clearnet → satellite (LEO) | >20 Mbps | Satellite link |
| mesh (WiFi) → clearnet | >10 Mbps | Bridge overhead |
| mesh-to-mesh (WiFi→WiFi via overlay) | >5 Mbps | Dual bridge overhead |
| clearnet → mesh → clearnet | >5 Mbps | Mesh bottleneck |

---

## 2. N-Hop Path Performance

| Path Length | Routing Domains | Estimated Throughput | Notes |
|---|---|---|---|
| 3 hops | 1 | ~140 Mbps | Minimum path |
| 4 hops | 1 | ~105 Mbps | Default clearnet |
| 7 hops | 1 | ~60 Mbps | High-security single-domain |
| 7 hops | 2 + 1 bridge | ~45 Mbps | Default cross-domain |
| 10 hops | 3 + 2 bridges | ~35 Mbps | Three-domain path |
| 15 hops | 3 + 2 bridges | ~28 Mbps | Multi-domain high-security |
| 20 hops | 4 + 3 bridges | ~21 Mbps | Ultra-high-security |

No protocol-enforced hop ceiling. Performance degrades with hop count as expected from overlay overhead. N-hop unlimited policy confirmed by performance testing — no paths rejected.

---

## 3. Hardware-Specific Targets

### OpenWrt Router (MT7981B, 512MB RAM)

| Metric | Target |
|---|---|
| Max sessions | 2,000 |
| Throughput | >50 Mbps |
| Path construction | <500ms |
| CPU utilization at max load | <80% |
| RAM usage | <200 MB |
| Cross-domain bridge transition overhead | <3ms |

### Raspberry Pi 4 (ARM Cortex-A72, 2GB RAM)

| Metric | Target |
|---|---|
| Max sessions | 5,000 |
| Throughput | >200 Mbps |
| Path construction | <200ms |

### x86 Server (8 cores, 8GB RAM)

| Metric | Target |
|---|---|
| Max sessions | 10,000 |
| Throughput | >1 Gbps |
| Path construction | <100ms |

---

## 4. Pool and Multi-Segment Overhead

| Overhead Type | Amount |
|---|---|
| Single pool lookup (cached) | <5ms |
| Single pool lookup (DHT) | <200ms |
| Individual node verification | <1ms |
| Cross-domain path construction vs. single-domain | +500–1500ms |
| Bridge node discovery (cache miss) | +300ms |
| DOMAIN_BRIDGE hop overhead vs. RELAY | +1–2ms per bridge |
| Cross-domain path_id table lookup | <100ns |

---

## 5. Mesh Transport Performance

| Transport | Single-Hop Latency | Throughput |
|---|---|---|
| WiFi 802.11s | 1-20ms | 10-100 Mbps |
| LoRa SF7 | 100-500ms | ~50 Kbps |
| LoRa SF9 | 500ms-2s | ~10 Kbps |
| LoRa SF12 | 2-5s | ~0.3 Kbps |
| BLE | 10-100ms | 1-2 Mbps |

---

## 6. Micro-Benchmarks Required

All previous benchmarks retained. New:

| Benchmark | Target |
|---|---|
| DOMAIN_BRIDGE onion layer construction | <2ms |
| DOMAIN_BRIDGE onion unwrapping at bridge node | <1ms |
| Cross-domain path_id table insert | <1µs |
| Cross-domain path_id table lookup | <100ns |
| Domain transport selection | <50ns |
| Bridge node discovery from DHT | <300ms (network-bound) |
| Bridge pair record signature verification | <5ms |
| Mesh island advertisement verification | <5ms |
| N-hop path construction (20 hops) | <1500ms |
| Cross-domain diversity check (20 nodes) | <100µs |

---

## 7. Performance Regression Testing

```bash
# Baseline benchmarks
cargo bench --bench forwarding > baseline.txt
cargo bench --bench cross_domain_forwarding > baseline_cross_domain.txt
cargo bench --bench path_construction > baseline_path.txt

# After changes
cargo bench --bench forwarding > current.txt
cargo bench --bench cross_domain_forwarding > current_cross_domain.txt

# Compare (fail if >5% regression)
bgpx-bench compare baseline.txt current.txt --max-regression 0.05
bgpx-bench compare baseline_cross_domain.txt current_cross_domain.txt --max-regression 0.05
```
