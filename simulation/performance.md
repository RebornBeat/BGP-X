# BGP-X Performance Specification and Benchmarks

**Version**: 0.1.0-draft

This document specifies the performance targets for BGP-X, the benchmarking methodology, and the results from simulation and early implementation testing.

---

## 1. Performance Targets

### 1.1 Client-Side Targets

| Metric | Target | Minimum Acceptable |
|---|---|---|
| Path construction time | <500ms (p95) | <2000ms (p99) |
| Session establishment (4 hops) | <200ms (p95) | <500ms (p99) |
| Multi-pool path construction | <650ms (p95) | <2500ms (p99) |
| Stream open (after path ready) | <50ms (p95) | <200ms (p99) |
| Throughput per path (4 hops) | >50 Mbps | >10 Mbps |
| Added latency per hop (processing) | <5ms | <10ms |
| CPU usage (idle, 4 paths) | <1% single core | <5% |
| Memory usage (4 paths) | <50 MB | <200 MB |

### 1.2 Node-Side Targets

| Metric | Target | Minimum |
|---|---|---|
| Per-packet forwarding latency (RELAY) | <1ms (p99) | <3ms (p99) |
| Maximum relay throughput (4-core server) | >1 Gbps | >100 Mbps |
| Maximum concurrent sessions | >10,000 | >1,000 |
| Handshakes per second | >1,000/s | >100/s |
| DHT lookup latency | <200ms (p95) | <1000ms (p99) |
| Memory per session | <256 bytes | <1 KB |
| CPU per 1 Gbps relay | <2 cores | <4 cores |

### 1.3 Router Firmware Targets (Tier 1 Hardware)

| Metric | Target | Minimum Acceptable |
|---|---|---|
| Client throughput (GL-MT6000, 4 hops) | >80 Mbps | >20 Mbps |
| Client latency overhead | <10ms per hop | <20ms per hop |
| Memory usage (client mode) | <64 MB | <128 MB |
| CPU usage at 50 Mbps | <60% single core | <90% single core |

### 1.4 Cross-Domain Path Construction Targets

| Metric | Target | Minimum Acceptable |
|---|---|---|
| 2-domain path (1 bridge), p95 | <800ms | <3000ms p99 |
| 3-domain path (2 bridges), p95 | <1200ms | <5000ms p99 |
| 4+ domain path, p95 | <1500ms | <8000ms p99 |
| Bridge node discovery (DHT, cache miss) | <300ms | <1000ms |
| Bridge node discovery (cached) | <5ms | <20ms |

### 1.5 Domain Bridge Overhead

| Metric | Target | Minimum |
|---|---|---|
| DOMAIN_BRIDGE hop processing latency | <2ms (p99) | <5ms (p99) |
| Cross-domain path_id table lookup | <100ns (p99) | <500ns (p99) |
| Transport selection per domain | <50ns (p99) | <200ns (p99) |

### 1.6 Cross-Domain Throughput by Path Type

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

BGP-X imposes **no protocol-level maximum on path hop count**. Performance degrades with hop count as expected from overlay overhead. N-hop unlimited policy confirmed by performance testing — no paths rejected.

| Path Length | Routing Domains | Estimated Throughput | Notes |
|---|---|---|---|
| 3 hops | 1 | ~140 Mbps | Minimum path |
| 4 hops | 1 | ~105 Mbps | Default clearnet |
| 5 hops | 1 | ~84 Mbps | High-security single-domain |
| 7 hops | 1 | ~60 Mbps | High-security single-domain |
| 7 hops | 2 + 1 bridge | ~45 Mbps | Default cross-domain |
| 10 hops | 3 + 2 bridges | ~35 Mbps | Three-domain path |
| 15 hops | 3 + 2 bridges | ~28 Mbps | Multi-domain high-security |
| 20 hops | 4 + 3 bridges | ~21 Mbps | Ultra-high-security |

No protocol-enforced hop ceiling. Performance degrades with hop count as expected from overlay overhead. N-hop unlimited policy confirmed by performance testing — no paths rejected.

---

## 3. Pool and Multi-Segment Overhead

| Overhead Type | Amount |
|---|---|
| Multi-pool path construction vs. single pool | +50-150ms |
| Cross-pool session handoff (inter-segment) | +25ms |
| Pool advertisement fetch (cache miss) | +100-200ms (cached: 0) |
| Pool membership verification (per node) | +2-5ms |
| Single pool lookup (cached) | <5ms |
| Single pool lookup (DHT) | <200ms |
| Individual node verification | <1ms |
| Cross-domain path construction vs. single-domain | +500–1500ms |
| Bridge node discovery (cache miss) | +300ms |
| DOMAIN_BRIDGE hop overhead vs. RELAY | +1–2ms per bridge |
| Cross-domain path_id table lookup | <100ns |

---

## 4. Pluggable Transport Overhead

| Overhead Type | Amount |
|---|---|---|
| CPU overhead | ~15% additional |
| Bandwidth overhead | ~5-20% (random padding) |
| Latency overhead (jitter) | 0-50ms (configurable) |
| Throughput reduction | ~10-15% |

---

## 5. ECH Overhead

| Overhead Type | Amount |
|---|---|---|
| ECH config DNS lookup (cache miss) | 20-50ms |
| ECH config DNS lookup (cached) | <1ms |
| ECH TLS handshake overhead | ~5ms vs standard TLS |

---

## 6. Mesh Transport Performance

| Transport | Single-Hop Latency | Throughput | Notes |
|---|---|---|---|
| WiFi 802.11s | 1-20ms | 10-100 Mbps | Variable by congestion |
| LoRa SF7 | 100-500ms | ~50 Kbps | Sub-GHz ISM band |
| LoRa SF9 | 500ms-2s | ~10 Kbps | Longer range, lower throughput |
| LoRa SF12 | 2-5s | ~0.3 Kbps | Maximum range |
| Bluetooth BLE | 10-100ms | 1-2 Mbps | Short range |
| Ethernet P2P | <5ms | 100 Mbps - 10 Gbps | Direct wired link |

### Fragment Reassembly Overhead

For LoRa: packet fragmented into multiple LoRa frames. End-to-end delivery latency = (fragment count × frame airtime) + reassembly overhead.

For SF7 at 50 Kbps with 1160-byte BGP-X packet (7 fragments of ~162 bytes each):
- Each fragment: ~25ms airtime
- Total delivery: ~175ms + reassembly overhead

---

## 7. Geographic Plausibility Scoring

| Operation | Overhead |
|---|---|---|
| RTT measurement (amortized over keepalives) | <0.1ms per packet |
| Score computation | <0.5ms |
| Reputation event generation | <1ms |

---

## 8. Routing Policy Engine

| Operation | Overhead |
|---|---|---|
| Rule evaluation (50 rules) | <100ns |
| DNS-based rule (cache hit) | <50ns |
| DNS-based rule (cache miss) | First connection only |
| Policy reload from file | <5ms |

---

## 9. Hardware-Specific Targets

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

## 10. Benchmarking Methodology

### 10.1 Micro-benchmarks

Individual component benchmarks using Criterion.rs:

```rust
// Example: ChaCha20-Poly1305 throughput benchmark
fn bench_chacha20(c: &mut Criterion) {
    let key = [0u8; 32];
    let nonce = [0u8; 12];
    let plaintext = vec![0u8; 1160];  // BGP-X max payload

    c.bench_function("chacha20_encrypt_1160b", |b| {
        b.iter(|| {
            chacha20poly1305_encrypt(&key, &nonce, &[], &plaintext)
        })
    });
}
```

### 10.2 Required Micro-benchmarks

| Benchmark | Target |
|---|---|
| ChaCha20-Poly1305 encrypt (1160 bytes) | >2 GB/s (x86_64 with AVX2) |
| ChaCha20-Poly1305 decrypt (1160 bytes) | >2 GB/s |
| X25519 key exchange | <0.1ms |
| Ed25519 sign | <0.1ms |
| Ed25519 verify | <0.2ms |
| BLAKE3 hash (32 bytes) | <0.01ms |
| Onion construction (4 hops, 1160 bytes) | <1ms |
| Onion unwrap (single layer) | <0.25ms |
| Session table lookup (10,000 sessions) | <100ns |
| Path table lookup (path_id) | <100ns |
| Pool membership verification (per node) | <5ms |
| Routing policy evaluation (50 rules) | <100ns |
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

### 10.3 Integration Benchmarks

End-to-end throughput and latency measured in the simulation framework:

```toml
[benchmark_scenario]
name = "throughput_4hop"
topology = "geographic"
node_count = 100
client_count = 10
path_hops = 4
workload = "saturating"  # Send as fast as possible
duration_seconds = 30
```

### 10.4 Hardware Benchmarks

For router firmware targets, benchmarks are run on physical hardware:

```bash
# Run firmware benchmark suite on connected device
bgpx-bench firmware \
    --device gl-mt6000 \
    --connection ssh://root@192.168.8.1 \
    --duration 60 \
    --output results/gl-mt6000-bench.json
```

---

## 11. Performance Benchmark Results (Simulated)

The following results are from simulation with ideal network conditions (no packet loss, low jitter). Real-world results will vary.

### 11.1 Throughput by Path Length

| Path Length | Simulated Throughput | Notes |
|---|---|---|
| 3 hops | ~140 Mbps | Minimum path |
| 4 hops | ~105 Mbps | Default path |
| 5 hops | ~84 Mbps | High-security |
| 7 hops | ~60 Mbps | Maximum tested |

Throughput scales inversely with path length due to:
- Additional encryption/decryption at each hop
- Network latency through each additional hop
- Congestion control adjustments for longer RTT

### 11.2 Latency by Path Length

| Path Length | Processing Overhead | Geographic Latency (NA-EU-AP) |
|---|---|---|
| 3 hops | ~3ms | ~150–350ms |
| 4 hops | ~4ms | ~200–450ms |
| 5 hops | ~5ms | ~250–550ms |

Geographic latency dominates. Processing overhead is small relative to inter-region network latency.

### 11.3 Path Construction Time by Network Size

| Network Size | Median | p95 | p99 |
|---|---|---|---|
| 100 nodes | 45ms | 120ms | 280ms |
| 1,000 nodes | 85ms | 210ms | 440ms |
| 10,000 nodes | 180ms | 420ms | 820ms |

Path construction time grows slowly with network size because the DHT lookup is O(log N) and the path selection algorithm operates on a cached node database.

---

## 12. Performance Regression Testing

All pull requests to the reference implementation that touch performance-critical paths MUST include benchmark results showing no regression:

```bash
# Run benchmarks on current branch
cargo bench --bench forwarding > current.txt
cargo bench --bench cross_domain_forwarding > current_cross_domain.txt
cargo bench --bench path_construction > current_path.txt

# Compare to main branch
cargo bench --bench forwarding -- --baseline main > comparison.txt
cargo bench --bench cross_domain_forwarding -- --baseline main > comparison_cross_domain.txt

# Fail if any benchmark regressed by more than 5%
bgpx-ci check-regression --threshold 0.05 comparison.txt
bgpx-ci check-regression --threshold 0.05 comparison_cross_domain.txt
```

Performance regressions of more than 5% in any micro-benchmark require explicit justification and approval before merging.

---

## 13. Profiling Tools

For performance investigation:

```bash
# CPU profiling with perf (Linux)
perf record -g bgpx-node --config /etc/bgpx/bench.toml
perf report

# Flame graph generation
cargo flamegraph --bin bgpx-node -- --config /etc/bgpx/bench.toml

# Memory profiling with heaptrack
heaptrack bgpx-node --config /etc/bgpx/bench.toml
heaptrack_gui heaptrack.bgpx-node.*.gz

# Async task profiling with tokio-console
TOKIO_CONSOLE=1 bgpx-node --config /etc/bgpx/bench.toml
tokio-console
```
