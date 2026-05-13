# BGP-X Gateway v1 Hardware Specification

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2 (Hardware), MIT (Firmware/Software), CC BY 4.0 (Documentation)

---

## 1. Overview

The BGP-X Gateway v1 is **provider-grade infrastructure** for operators running public BGP-X exit nodes, high-capacity domain bridge nodes, and backbone relay nodes. It is **not a consumer device**.

### Target Users

- ISPs offering BGP-X relay or exit service to their customers
- Hosting providers colocating BGP-X infrastructure
- Enterprise operators running private BGP-X relay pools
- Community organizations operating high-capacity mesh island gateways with dedicated server infrastructure
- Maritime operators requiring network infrastructure with high availability

### Optimization Targets

The Gateway v1 is optimized for:

- **Maximum clearnet throughput** (target: >1 Gbps)
- **High concurrent session count** (target: 50,000 sessions)
- **Low per-packet forwarding latency** (<500µs in hot path)
- **24/7 unattended operation** in datacenter or weatherproof outdoor enclosures
- **Multi-WAN redundancy** for ISP failover

---

## 2. Design Philosophy — Adaptable Reference Implementation

The Gateway v1 specification defines **capability requirements and reference implementation**. The specific SoC, chipset, and board design represent the current reference. Future revisions may substitute:

- Higher-performance ARM SoCs
- x86 processors for gateway workloads
- Custom BGP-X forwarding ASICs
- Different network interfaces (higher-speed SFP28, QSFP, etc.)

The `bgpx-node` daemon and `bgpx-gateway` daemon are hardware-agnostic. Any platform providing standard Linux network interfaces, sufficient CPU, and RAM can run Gateway v1 firmware.

**Capability requirements drive compliance — not specific part numbers.**

---

## 3. Capability Requirements (Hardware-Agnostic)

| Capability | Minimum | Reference Implementation |
|---|---|---|
| CPU | Quad-core ARM Cortex-A53 or equivalent, ≥1.5 GHz | MediaTek MT7988A quad-core Cortex-A73 @ 1.8 GHz |
| NPU / Offload | Optional; reduces CPU load for forwarding | MT7988A integrated Network Processing Unit |
| RAM | 1 GB DDR4 | 1 GB DDR4 @ 3200 MHz |
| Storage (boot) | 128 MB persistent | 128 MB SPI NAND |
| Storage (data) | 8 GB persistent | 16 GB eMMC 5.1 |
| Network WAN | ≥1 GbE; SFP+ for fiber uplink preferred | SFP+ (10 Gbps) + 2× 2.5GbE |
| Network LAN | Not required for gateway-only operation | 4× Gigabit Ethernet |
| Cellular | Optional mPCIe/USB for LTE backup | 1× mPCIe slot (Sierra Wireless EM7430 / Quectel EC25) |
| USB | ≥1× USB 3.0 for optional LoRa adapter | 2× USB 3.0 |
| Security element | TPM 2.0 recommended; software key fallback | SLB9670 TPM 2.0 |
| Linux support | Mainline Linux 6.1+ preferred | OpenWrt 23.05 / Linux 6.1 |
| Throughput target | ≥500 Mbps BGP-X relay forwarding | >1 Gbps (MT7988A NPU) |
| Session target | ≥10,000 concurrent BGP-X sessions | 50,000 (provider mode) |

---

## 4. Reference Implementation — Core Hardware

### Compute Platform

| Component | Reference Part | Notes |
|---|---|---|
| SoC | MediaTek MT7988A (Filogic 880) | Quad-core ARM Cortex-A73 @ 1.8 GHz |
| NPU | MT7988A integrated Network Processing Unit | Hardware packet classification, reduces CPU load |
| RAM | 1 GB DDR4 @ 3200 MHz | Large session table + cross-domain path_id tables |
| Boot flash | 128 MB SPI NAND | Bootloader + OS |
| Storage | 16 GB eMMC 5.1 | Daemon, exit policy, logs, DHT cache |
| TPM | SLB9670 TPM 2.0 | Key storage; hardware signing; HWRNG |
| Power | 12V DC @ 3A (36W max) OR PoE 802.3bt (60W for outdoor) | Dual power options |
| Consumption | ~8W idle, ~18W full load (rack) | Varies with enclosure and load |

**Adaptability note**: The MT7988A is the reference SoC. For x86-based gateway deployments, any x86-64 server with Linux and sufficient NICs is compatible with `bgpx-gateway-firmware` (Debian/Ubuntu package). The daemon has no ARM dependency.

### Network Interfaces

| Interface | Specification | Use |
|---|---|---|
| SFP+ | 10 Gbps fiber/copper (SFP+ cage) | Primary WAN — fiber ISP uplink |
| WAN 1 | 2.5GbE (RJ45) | Secondary WAN or failover |
| WAN 2 | 2.5GbE (RJ45) | Secondary WAN or bonding |
| LAN × 4 | Gigabit Ethernet (RJ45) | Local management / testing |
| mPCIe | 1× mPCIe slot | 4G/LTE module for WAN backup |
| USB 3.0 × 2 | External adapters | Optional LoRa adapter, satellite modem |
| WiFi (optional) | MT7915 WiFi 6, 802.11ax | Mesh domain if needed; not primary use |

### Multi-WAN Configuration

Gateway v1 supports dual-WAN for ISP redundancy:

```
WAN 1: Fiber/VDSL via SFP+ or 2.5GbE (primary)
WAN 2: 4G/LTE via mPCIe module (backup)

BGP-X configuration:
[network]
wan_primary = "eth0"      # WAN 1
wan_backup = "wwan0"      # WAN 2 (4G via ModemManager)
failover_enabled = true
```

### Compatible LTE Modules

| Module | Region | Category | Notes |
|---|---|---|---|
| Sierra Wireless EM7430 | Global | Cat 9 | Primary recommendation |
| Quectel EC25-E | Europe/APAC | Cat 4 | Cost-effective |
| Quectel EC25-A | Americas | Cat 4 | Cost-effective |

SIM card slot on PCB adjacent to mPCIe cage.

### LoRa and Domain Bridge Capability

While the Gateway v1 is primarily a clearnet exit/relay provider, it can also function as a **domain bridge node**. Use cases:

- Large-scale mesh island gateway with high-bandwidth clearnet uplink
- Multi-island bridge node connecting multiple mesh communities to clearnet
- Provider running a certified domain bridge pool

For domain bridge operation, add **USB LoRa adapter** or **USB WiFi 802.11s adapter**. The gateway's high-bandwidth SFP+ clearnet uplink combined with LoRa mesh transport creates a high-capacity clearnet-to-LoRa bridge.

**Note**: Unlike the Router v1 and Node v1, Gateway v1 does NOT have integrated LoRa radio on-board. Provider-grade rack deployments typically use external LoRa gateways or USB adapters with external antennas, not integrated chip antennas.

---

## 5. Enclosure Variants

### 5.1 1U Rack Mount (Primary — Datacenter Colocation)

| Attribute | Specification |
|---|---|
| Form factor | 1U 19" rack |
| Depth | 200-250 mm (half-depth) |
| Ports (front) | 4× RJ45 (LAN), 2× USB-A, status LEDs |
| Ports (rear) | SFP+ cage, 2× 2.5GbE (WAN), IEC 320 C14 power inlet, LoRa SMA (if USB adapter installed) |
| Power | AC 100-240V, 50/60 Hz; IEC 320 C14 inlet; <40W typical |
| Cooling | 40mm fan (thermostatically controlled); fanless option for low-ambient deployments |
| IP rating | IP20 |
| Temperature | 0°C to +45°C |

### 5.2 Outdoor Weatherproof Enclosure (Remote Operator Deployment)

| Attribute | Specification |
|---|---|
| Form factor | Sealed aluminium enclosure |
| Dimensions | 300 × 200 × 100 mm |
| Ports | N-type WiFi (if used), SMA LoRa, Ethernet cable gland, DC power cable gland |
| Power | PoE 802.3bt (60W) or DC 24-48V wide input |
| IP rating | IP65 |
| Temperature | -20°C to +60°C |
| Mount | Pole mount, wall mount |

### 5.3 Maritime Enclosure (Marine / Offshore)

| Attribute | Specification |
|---|---|
| Form factor | Sealed marine-grade aluminium / 316 stainless steel |
| Dimensions | 350 × 250 × 120 mm |
| Ports | Waterproof connectors for power, Ethernet, LoRa antenna |
| Power | DC 12-24V wide input; PoE optional |
| IP rating | IP68 |
| Temperature | -20°C to +60°C |
| Vibration | Tested for marine vibration profiles |
| Mount | Bulkhead mount, mast mount |

---

## 6. Software Platform

| Component | Specification |
|---|---|
| OS | OpenWrt 23.05+ (Gateway v1 build) or Debian 12 (x86 variant) |
| Kernel | Linux 6.1 LTS with network performance patches |
| BGP-X daemon | `bgpx-node` (gateway build: max_sessions=50,000, large buffers) |
| BGP-X gateway | `bgpx-gateway` (exit policy engine, ECH, DoH) |
| BGP-X CLI | `bgpx-cli` |
| Web UI | `bgpx-luci` (provider profile) |
| Exit policy tools | `bgpx-exitpolicy` (sign/verify) |
| Monitoring | `prometheus-node-exporter` (optional; provider monitoring) |
| Time sync | `chrony` (strict NTP; max offset 100ms warning, 500ms critical) |
| TPM | `tpm2-tss`, `tpm2-tools` |

### Provider-Optimized Configuration Defaults

```toml
[node]
role = "exit,domain_bridge"

[network]
wan_primary = "eth0"           # SFP+ or 2.5GbE port
wan_backup = "wwan0"           # LTE via mPCIe
failover_enabled = true

[sessions]
max_sessions = 50000
idle_timeout_secs = 120
session_rehandshake_age_hours = 24

[performance]
socket_buffer_mb = 64          # SO_RCVBUF/SO_SNDBUF
worker_threads = 0             # 0 = all CPU cores
sender_threads = 8
path_table_shards = 256        # vs. 64 default
cross_domain_table_shards = 256

[paths]
default_hops = 4
min_hops = 3
# No max_hops: N-hop unlimited

[exit]
enabled = true                 # Provider exit nodes enable exit
ech_cache_size = 10000
doh_threads = 4
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"

[metrics]
prometheus_listen = "0.0.0.0:9090"
```

---

## 7. Performance Targets

At 18W TDP with MT7988A quad-core 1.8 GHz:

| Role | Expected Throughput | Notes |
|---|---|---|
| Exit gateway (TCP) | 500-800 Mbps | Depends on TLS/ECH overhead |
| Relay only | 1-2 Gbps | Limited by NPU capacity |
| Mesh bridge (WiFi) | 100-300 Mbps | WiFi transport bottleneck |
| Crypto operations | >4 Gbps | AES/ChaCha20 with hardware acceleration |
| Session capacity | 50,000 sessions | With 1 GB RAM |

---

## 8. Gateway v1 vs. Router v1 vs. Node v1

| Feature | Router v1 | Node v1 | Gateway v1 |
|---|---|---|---|
| Target | End user (AIO) | Community contributor | Provider/operator |
| Replaces home router | Yes | No | No |
| LAN switching | Yes | No | Optional |
| Throughput | ~500 Mbps | ~100 Mbps | >1 Gbps |
| Max sessions (default) | 10,000 | 2,000 | 50,000 |
| RAM | 512 MB | 256-512 MB | 1 GB |
| Integrated LoRa | Yes (SX1262) | Yes (SX1262) | No (USB only) |
| WiFi 802.11s | Yes (MT7915 WiFi 6) | Yes | Optional |
| TPM 2.0 | Yes | Optional | Yes |
| Solar/battery native | No | Yes | No |
| SFP+ uplink | No | No | Yes |
| LTE backup | No | No | Yes (mPCIe) |
| Exit node certified | Optional | Optional | Primary use |
| Typical deployment | Home/office | Rooftop/mast | Datacenter rack |
| Cost tier | Mid | Low | High |

---

## 9. Deployment — Datacenter Colocation

The typical Gateway v1 provider deployment:

```
ISP fiber uplink (1-10 Gbps)
         │ SFP+
         ▼
BGP-X Gateway v1 (1U rack)
         │ bgpx-node daemon
         ├── Clearnet relay: accepts BGP-X sessions from overlay
         ├── Exit node: connects to internet on behalf of final-hop clients
         ├── Exit policy: signed; published to DHT
         └── ECH: resolves ECH config for destinations; hides domains at exit
         │
         │ Ethernet management port
         ▼
Management switch → operator SSH access
```

### Minimum Requirements for Certified Tier 4 Exit Node

- Dedicated server (no VPS neighbors) or Gateway v1 hardware
- SFP+ fiber uplink ≥1 Gbps
- Signed exit policy published to DHT
- No session logs (technically enforced)
- Logging policy = "none" verified
- ECH support functional
- DoH with DNSSEC and ECS stripping

---

## 10. Regulatory and Legal Considerations

Gateway v1 operators running public exit nodes should:

- Register as a legal entity operating the exit node
- Consult legal counsel in their jurisdiction regarding exit node operation
- Maintain an abuse contact address
- Publish their exit policy publicly (it is already in the DHT)
- Review `/legal/liability.md` and `/legal/privacy_policy.md`
- Consider RPKI-validated IP space to reduce BGP hijacking risk

See `production/sop.md` for operational procedures specific to exit node and domain bridge node operation.

---

## 11. Bill of Materials — Approximate Component Cost (Volume)

| Component | Approximate Cost |
|---|---|
| MT7988A SoC module + RAM | $20–35 |
| eMMC 16 GB | $5–10 |
| Boot NAND 128 MB | $2–4 |
| TPM 2.0 (SLB9670) | $3–5 |
| SFP+ cage + PHY | $8–15 |
| 2.5GbE PHY × 2 | $5–10 |
| GbE switch (4-port) | $4–8 |
| USB 3.0 controller | $2–4 |
| mPCIe slot | $1–2 |
| PCB (4-layer) | $5–10 |
| Passives + connectors | $5–10 |
| 1U chassis + fan | $15–30 |
| **Total (1U rack)** | **$75–143** |

---

## 12. Security Model

The Gateway v1 implements hardware-enforced security:

- **TPM 2.0**: All node keys generated and stored within TPM; private key never leaves TPM
- **Secure boot**: Verified boot chain from power-on to `bgpx-node` daemon
- **Encrypted storage**: eMMC partition for exit policy and DHT cache encrypted with TPM-sealed key
- **Hardware RNG**: TPM provides entropy for cryptographic operations
- **Physical tamper evidence**: Chassis intrusion detection (optional)

For provider deployments requiring additional assurance:
- Hardware Security Module (HSM) via USB (Nitrokey HSM, YubiHSM 2)
- Out-of-band management (IPMI or serial console) for secure remote access
