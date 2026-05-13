# BGP-X Node v1 Hardware Specification

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2

---

## 1. Overview

The BGP-X Node v1 is a **compact, multi-radio BGP-X routing node** designed for community contributors. It is not a home router replacement — it is a device that community members add to an existing BGP-X network to expand mesh coverage, provide relay capacity, or serve as a community gateway.

A typical BGP-X Node v1 deployment: mounted on a rooftop or mast, solar-powered, running 24/7, participating in a mesh island as a relay node or domain bridge. No ISP connection required if the island has an existing gateway. With WAN Ethernet, it becomes a gateway for the island.

### What the Node v1 IS

- A full BGP-X routing node running the complete bgpx-node daemon
- A multi-radio device: LoRa, WiFi 802.11s, BLE — any combination or all simultaneously
- Solar and battery optimized: ultra-low idle power, efficient duty cycle management
- IP67 by default: built for outdoor elevated deployment
- Configurable: same hardware, different software configuration = different role

### What the Node v1 is NOT

- NOT a home router replacement (no multi-port LAN switching by default)
- NOT a pure "dumb" repeater (it runs full BGP-X crypto and routing — it IS a BGP-X node)
- NOT provider-class infrastructure (for that, see BGP-X Gateway v1)

---

## 2. Design Philosophy — Adaptable Reference Implementation

The BGP-X Node v1 specification defines **capability requirements** for the community node use case, not locked silicon. The reference implementation uses specific components, but future revisions may substitute any equivalent hardware.

The BGP-X daemon running on Node v1 is **identical** to the daemon on Router v1 and Gateway v1. Same binary, same protocol, same configuration format, same capabilities — the hardware is more compact and power-optimized, but the software is the same.

**Capability requirements drive compliance — not specific part numbers.**

---

## 3. Capability Requirements (Hardware-Agnostic)

Any implementation claiming BGP-X Node v1 compliance MUST meet:

| Capability | Minimum | Reference Implementation |
|---|---|---|
| CPU | Single or dual-core ARM Cortex-A53 or equivalent, ≥800 MHz | ARM Cortex-A53 @ 1.0–1.3 GHz |
| RAM | 256 MB (relay mode); 512 MB (domain bridge mode) | 256–512 MB DDR4 / LPDDR4 |
| Storage | 2 GB persistent + 64 MB boot | 4–8 GB eMMC + 64–128 MB NAND |
| LoRa | SPI-connected LoRa transceiver (SX1261/SX1262 or equivalent) | SX1262 |
| WiFi | 802.11s mesh capable (optional if LoRa-only configuration) | MT7915 (WiFi 6) or MT7612U (WiFi 5) |
| BLE | BLE 5.0+ module (optional) | Standalone UART BLE module |
| Power | Solar + LiPo/LiFePO4 battery input; OR PoE 802.3af minimum | Solar MPPT + 3.7V LiPo / LiFePO4 |
| WAN (optional) | 1× 10/100/1000 Ethernet (for domain bridge/gateway mode) | 1× GbE (optional) |
| USB | 1× USB 2.0+ (for additional adapters or satellite modem) | 1× USB 2.0 |
| Security | Optional TPM 2.0 for hardware-bound keys | Infineon SLB9670 / SLB9672 TPM 2.0 |
| Linux support | Mainline Linux 5.15+ or vendor BSP | OpenWrt 23.05 |

**Note on memory**: 256 MB supports the BGP-X relay daemon with standard path_id tables (up to ~5,000 concurrent entries). 512 MB enables domain bridge operation with full cross-domain path_id table capacity (up to ~20,000 entries). Community contributors adding coverage without gateway function can use 256 MB.

---

## 4. Configuration Modes

The BGP-X Node v1 runs a single firmware and a single bgpx-node daemon. Configuration in `config.toml` determines the operating mode. All modes use the same hardware — the radio configuration and role settings change, not the hardware.

### Mode 1: Mesh Relay (No WAN Required)

Configuration:
```toml
[node]
role = "relay"

[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["lora", "wifi-mesh"]
```

Behavior: The Node v1 participates in the mesh island as a relay node. It forwards BGP-X traffic between LoRa and WiFi mesh peers. No ISP connection needed. The island's gateway (another node with WAN) provides clearnet access. Ideal for adding coverage at elevation.

### Mode 2: Domain Bridge (With WAN)

Configuration:
```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["lora", "wifi-mesh"]
```

Behavior: With WAN Ethernet connected, the Node v1 bridges clearnet to the mesh island. Clearnet clients can reach mesh island services through this node. The node publishes DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE to the unified DHT. This is a complete domain bridge node.

### Mode 3: Community Gateway (With WAN, Exit Enabled)

Configuration:
```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[exit]
enabled = true
policy_path = "/etc/bgpx/exit_policy.toml"
```

Behavior: In addition to bridge mode, the Node v1 acts as a clearnet exit for the mesh island's traffic. Island users can reach clearnet services through this gateway. The node applies exit policy and provides ECH at exit. One Node v1 per community with WAN connection provides full clearnet access for the entire island.

### Mode 4: Range Extension (Low-Overhead Relay)

Configuration:
```toml
[node]
role = "relay"

[performance]
relay_only_mode = true        # Minimize session origination; prioritize forwarding
max_sessions = 500            # Reduced from default 10,000
path_construction = false     # Do not originate paths; only relay

[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["lora"]         # LoRa only for maximum range
```

Behavior: The Node v1 prioritizes forwarding BGP-X traffic over acting as an active relay. This extends effective LoRa mesh coverage with minimal resource use. The node still runs full BGP-X crypto — it is not a dumb repeater. All security properties are preserved. It simply deprioritizes being selected as a relay in path construction and maximizes its forwarding throughput.

### Mode 5: LoRa-Only Compact Node

Configuration:
```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["lora"]
# WiFi disabled to save power
```

Behavior: Only LoRa radio active. Maximum battery life. Used at range-extension positions where WiFi mesh adds no value (e.g., rural gap node between two LoRa clusters).

### Mode 6: WiFi-Only Compact Node

Configuration:
```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["wifi-mesh"]
# LoRa disabled
```

Behavior: Only WiFi 802.11s mesh active. Used in dense urban deployments where LoRa adds no range benefit over WiFi. Higher throughput than LoRa-only mode.

### Mode 7: Multi-Radio (All Active — Default Recommended)

Configuration:
```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"
transports = ["lora", "wifi-mesh", "ble"]
```

Behavior: All radios active simultaneously. BGP-X daemon selects transport per-path based on latency, throughput, and peer reachability. WiFi mesh for nearby high-bandwidth paths; LoRa for long-range or through-obstruction paths; BLE for very close devices. Most capable configuration.

---

## 5. Reference Hardware — Compute

### Primary Reference Platform (High Performance)

| Component | Specification |
|---|---|
| SoC | MediaTek MT7981B (Filogic 820) |
| CPU | ARM Cortex-A53 dual-core, 1.3 GHz |
| RAM | 512 MB DDR4 |
| Storage (OS/firmware) | 128 MB NAND flash |
| Storage (data) | 8 GB eMMC (node database, reputation, DHT) |
| Power | 5V DC via USB-C or PoE 802.3af |
| Consumption | ~4W idle, ~6W full load |

This platform provides maximum throughput and supports domain bridge and gateway modes with full session capacity. Identical to the Router v1 compute platform, optimized for outdoor deployment.

### Low-Power Reference Platform (Solar Optimized)

| Component | Specification |
|---|---|
| SoC | Allwinner H3/H5 or NXP i.MX8M Mini |
| CPU | ARM Cortex-A53 quad-core, 800–1200 MHz |
| RAM | 512 MB LPDDR4 |
| Storage | 4 GB eMMC + SD card slot |
| Power | 3.3–5V DC, solar + battery input |
| Consumption | <500 mW idle (radios receive-only) |

This platform is optimized for solar deployments where every milliwatt matters. Slightly lower throughput than MT7981B, but sufficient for LoRa-dominated traffic patterns.

### Candidate Compute Platforms

Evaluation criteria: power consumption, LoRa SPI integration, Linux BSP quality, cost, and availability.

- NXP i.MX8M Mini (quad-core A53, very low power)
- Allwinner H3/H5 (established OpenWrt support, low cost)
- MediaTek MT7620 (proven OpenWrt compatibility, very low power)
- Raspberry Pi Zero 2W (broad community support, easy prototyping)
- Custom BGP-X SoC (future option — integrated LoRa + compute)

All BGP-X Node v1 firmware is written to run on any of these platforms via OpenWrt Kconfig.

---

## 6. Reference Hardware — Radios

### LoRa Radio

| Parameter | Reference | Equivalent |
|---|---|---|
| Chipset | SX1262 | SX1261, SX1268, LLCC68, STM32WL (integrated) |
| Interface | SPI | SPI (any clock speed supported by chipset) |
| Max power | +22 dBm | +14 dBm minimum for useful range |
| Sensitivity | -148 dBm at SF12 | -140 dBm minimum |
| Frequency | Region SKU | 433/868/915 MHz |
| Antenna | SMA female | Any 50Ω connector |
| Duty cycle enforcement | Firmware token bucket | Required in all implementations |

**Low-power operation**: In solar/battery deployments, the LoRa radio uses SX1262 sleep mode between transmissions. Current draw in sleep: <1 µA. This enables multi-day battery operation with a 6V 3W solar panel and 3000 mAh LiPo. The BGP-X LoRa driver manages duty cycle and sleep scheduling.

### WiFi Radio (Optional)

| Parameter | Reference | Equivalent |
|---|---|---|
| Chipset | MT7915 (802.11ax/WiFi 6) or MT7612U (802.11ac) | Any Linux mac80211 802.11s capable chip |
| Mode | IEEE 802.11s mesh | Required for WiFi mesh domain participation |
| Interface | PCIe (MT7915) or USB (MT7612U) | Platform-appropriate interface |
| Bands | 2.4 GHz and/or 5 GHz | Single band acceptable for low-cost |

**Flexibility**: The WiFi module can be MT7915 (high performance, integrated with MT7981B SoC), MT7612U (USB dongle), or any SDIO/embedded 802.11ax chip. The only requirement is mac80211 802.11s support in the Linux kernel.

### Bluetooth BLE (Optional)

| Parameter | Specification |
|---|---|
| Version | BLE 5.0+ |
| Interface | UART or USB |
| Stack | BlueZ (Linux) |
| BGP-X use | MESH_BEACON via BLE advertisement; BGP-X BLE transport |

BLE is optional and disabled by default. Enable with `transports = ["ble"]` in routing_domains config.

---

## 7. Power System

The BGP-X Node v1 is solar-and-battery native. This is a first-class design requirement, not an afterthought.

### Solar Power System (Reference Design)

```
6V / 3W solar panel (min)
         │
         ▼
CN3791 MPPT solar charge controller
         │
         ▼
3000-5000 mAh LiPo battery (3.7V) or LiFePO4 (3.2V)
         │
         ▼
5V boost regulator → compute SoC
         │
         ▼
3.3V LDO → LoRa SX1262, BLE module
         │
         ▼
WiFi module (USB or PCIe power rail)
```

**Power budget (typical LoRa + WiFi mesh + BLE, all active)**:
- Idle (receive only): 1.5–2.5W
- LoRa TX peak: +1.0W additional (150ms burst at +14 dBm)
- WiFi TX peak: +2.0W additional (bursts during packet forwarding)
- Maximum sustained: 4–6W

**With 3W solar panel**: charging exceeds discharge at ≥6 hours daily sun. Battery provides overnight operation.
**With 6W solar panel**: comfortable margin; suitable for cloudy conditions.

**Battery capacity for overnight operation at 2W average**: 3000 mAh × 3.7V × 0.9 (efficiency) = 9.99 Wh ÷ 2W = ~5 hours. **Minimum 6000 mAh LiPo recommended for reliable 24/7 off-grid operation with 3W panel**.

### PoE Power Option

For roof-mount or mast deployments with PoE cable run:
- 802.3af (15.4W) minimum — sufficient for all radios active
- 802.3at (30W) — comfortable headroom
- PoE injector at the base of the mast; Cat5e/Cat6 cable runs up to 100m
- Power over Ethernet eliminates solar panel requirement for elevated mounts with wired access

### Battery Chemistry Recommendations

| Chemistry | Advantage | Use Case |
|---|---|---|
| LiPo (3.7V) | Highest energy density | Compact installations |
| LiFePO4 (3.2V) | Wide temperature range (-20°C to +60°C); safer | Outdoor extreme temperature |
| NiMH (1.2V × cells) | No BMS required; simple | Cost-constrained |

For outdoor deployments in cold climates: LiFePO4 strongly recommended. LiPo loses significant capacity below -10°C.

---

## 8. Physical Form Factor

### Default Enclosure (IP67 Outdoor)

| Attribute | Specification |
|---|---|---|
| Material | UV-stabilized ABS or polycarbonate |
| Dimensions | 120 × 80 × 50 mm (reference) |
| IP rating | IP67 (dust-tight; 30-minute submersion to 1m) |
| Temperature | -40°C to +85°C |
| Mounting | Pole clamp (18–50mm dia), wall bracket, DIN rail |
| Antenna ports | 1× SMA (LoRa), 1–2× RP-SMA (WiFi), optional N-type for high-gain LoRa |
| Cable entry | M12 waterproof cable gland for Ethernet + power |

### Internal Mounting

Core PCB mounts inside enclosure on standoffs. Solar charge controller and battery mounted in base section. Top section: main PCB with radios and antennas. Separation prevents battery heat from affecting radio performance.

### Product Variants

| Variant | Description | Enclosure |
|---|---|---|
| BGP-X Node v1 Indoor | Standard relay/entry | ABS plastic, desktop |
| BGP-X Node v1 Outdoor | Weatherproof relay | IP67 polycarbonate, pole mount |
| BGP-X Node v1 Industrial | ATEX-rated (hazardous area) | ATEX Ex e IIC |
| BGP-X Node v1 Mobile | Vehicle-mounted | ABS, shock-mount |

All variants share the same PCB. Variants differ in enclosure, power input, and operating temperature range.

---

## 9. Security — TPM-Based Key Storage

The Node v1 supports optional TPM 2.0 for hardware-bound cryptographic keys.

| Component | Specification |
|---|---|
| TPM Module | Infineon SLB9670 / SLB9672 |
| Interface | SPI |
| Features | Hardware-bound Ed25519 keys, secure boot measurement, HWRNG |

### TPM Key Operations

If TPM is present, the node's Ed25519 private key:
- Generated on TPM (never leaves hardware)
- All signing operations performed within TPM
- Hardware attack resistant (JTAG, SPI bus attacks blocked)
- Key survives OS reinstall (stored in TPM NVIndex)
- Key lost only on TPM clear or hardware destruction

BGP-X daemon communicates with TPM via tpm2-tools and tpm2-tss:

```bash
# Generate node key in TPM
tpm2_create -G ecc256 -u /etc/bgpx/node_key.pub -r /etc/bgpx/node_key_ctx.priv

# Configure BGP-X to use TPM
[node]
private_key_storage = "tpm2"
tpm2_key_context = "/etc/bgpx/node_key_ctx.priv"
tpm2_key_public = "/etc/bgpx/node_key.pub"
```

**Note on Cost Reduction**: The TPM is optional for the Node v1 to reduce BOM cost for community deployments. Without TPM, keys are stored in encrypted eMMC (software keystore). TPM is strongly recommended for domain bridge and gateway nodes.

---

## 10. PCB Design

**Layer stackup**: 4-layer FR4
- Layer 1: Signal + components
- Layer 2: Ground plane
- Layer 3: Power plane (3.3V, 5V)
- Layer 4: Signal

**Key design considerations**:
- SX1262 LoRa: keep RF trace short; 50Ω controlled impedance; ground plane cutout under antenna connector
- MT7981B / H3 / i.MX8M: DDR4 trace length matching required (±5 mil)
- TPM 2.0: SPI bus isolated from main SPI; dedicated ground

**Open hardware files**: KiCad project files, Gerbers, BOM, pick-and-place released under CERN-OHL-S v2.

---

## 11. Software Platform

Same bgpx-node daemon as BGP-X Router v1. Same bgpx-cli. Same configuration format.

| Component | Specification |
|---|---|
| OS | OpenWrt 23.05+ (Node v1 build target) |
| Kernel | Linux 5.15 LTS or 6.1 LTS |
| BGP-X daemon | bgpx-node (same binary; ARM cross-compiled) |
| Configuration | /etc/bgpx/config.toml (same format as Router v1) |
| Web UI | bgpx-luci (same LuCI plugin as Router v1; accessible via WiFi AP if WiFi configured) |
| Power management | bgpx-power daemon: monitors battery state, adjusts radio TX power and duty cycle based on battery level |

### Power Management Daemon

The bgpx-power daemon monitors battery voltage and adjusts BGP-X behavior:

```toml
[power_management]
enabled = true
battery_monitor = "i2c"     # or "adc" depending on platform

# When battery < threshold, reduce LoRa TX power
battery_reduce_tx_threshold_pct = 30
lora_tx_power_reduced_dbm = 10   # from 14 to 10 dBm

# When battery critically low, disable WiFi (LoRa is lower power)
battery_disable_wifi_threshold_pct = 15

# When battery < 5%, enter low-power relay-only mode
battery_emergency_threshold_pct = 5
```

This ensures the node remains operational (forwarding BGP-X traffic) even in extended low-light conditions by reducing power draw in a controlled manner.

---

## 12. BGP-X Node v1 vs BGP-X Router v1

| Feature | BGP-X Router v1 | BGP-X Node v1 |
|---|---|---|
| Target user | Home/office user | Community contributor |
| Replaces home router | Yes | No |
| LAN switching | 3–4× GbE ports | Optional (1× WAN only) |
| NAT/DHCP for LAN | Yes | Optional |
| TPM 2.0 | Yes | Optional (cost reduction) |
| RAM | 512 MB standard | 256–512 MB |
| Power | PoE or DC | Solar + battery native; PoE optional |
| Default enclosure | Desktop or outdoor | Outdoor IP67 (default) |
| LoRa | Yes (integrated SX1262) | Yes (integrated SX1262) |
| WiFi mesh | Yes (MT7915 WiFi 6) | Yes (MT7915 or lower-tier module) |
| BLE | Optional (USB) | Yes (integrated UART module) |
| Full BGP-X daemon | Yes | Yes (same binary) |
| Domain bridge capable | Yes | Yes (with WAN option) |
| Gateway capable | Yes | Yes (with WAN + exit policy) |
| Satellite WAN | Yes (USB modem) | Yes (USB modem) |
| Cost (reference) | Higher | Lower |
| Form factor | Multiple carriers | Compact outdoor (default) |

---

## 13. Comparison to Old "BGP-X Amplifier v1" Concept

The original BGP-X specification included a "BGP-X Amplifier v1" on STM32H7 as a pure LoRa range extender with no BGP-X routing.

**The BGP-X Node v1 replaces and supersedes this concept entirely.**

**Why**: A pure LoRa PHY-level relay with no BGP-X crypto is technically simpler but introduces a security hole — traffic is relayed in an unencrypted intermediate state at the relay's PHY interface. BGP-X Node v1 in Range Extension mode provides all the coverage benefits of a "dumb" amplifier while maintaining full end-to-end BGP-X encryption at every hop.

The STM32H7 is too constrained for the BGP-X daemon (requires Linux for async I/O, full crypto, DHT). Community contributors who want to extend range should use the BGP-X Node v1 in Range Extension mode.

**If someone genuinely wants a LoRa PHY-level amplifier** (bidirectional signal amplifier, no digital processing): standard RF amplifier modules (SX1301-based or custom) exist for this purpose. This is not a BGP-X product — it is a radio hardware component outside the BGP-X ecosystem.

---

## 14. Bill of Materials — Approximate Component Cost (Volume)

| Component | Approximate Cost |
|---|---|
| ARM compute module (SoC + RAM + flash) | $8–15 |
| LoRa SX1262 module | $5–10 |
| WiFi module (MT7612U or equivalent) | $4–8 |
| BLE module (UART) | $2–4 |
| Solar charge controller (CN3791 or equivalent) | $1–3 |
| LiFePO4 3Ah cell | $5–10 |
| PCB (4-layer) | $3–6 |
| Passives, regulators, connectors | $3–6 |
| IP67 enclosure with cable glands | $8–18 |
| Mounting hardware (pole clamp etc.) | $3–8 |
| **Total (LoRa + WiFi + BLE, outdoor, with battery)** | **$42–88** |
| Solar panel 3W (additional) | $8–15 |
