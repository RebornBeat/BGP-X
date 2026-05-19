# BGP-X Router v1 Hardware Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X Router v1 is the **all-in-one primary product** for end users. It replaces a standard home or office router entirely. One device provides every capability defined in the BGP-X architecture:

- **WAN Connection**: ISP connection via Ethernet or USB satellite modem (clearnet domain).
- **LAN Routing**: Gigabit Ethernet switching for all connected devices.
- **BGP-X Overlay Relay**: Participation in the onion-routed overlay network.
- **WiFi Mesh**: IEEE 802.11s mesh networking (mesh domain).
- **LoRa Mesh**: Long-range radio networking (mesh domain via integrated SX1262).
- **Bluetooth BLE**: Short-range mesh transport (mesh domain via integrated BLE).
- **Domain Bridge**: Capable of bridging clearnet ↔ WiFi mesh ↔ LoRa mesh.
- **Mesh Island Gateway**: Functions as a gateway for a local mesh island to clearnet.
- **Satellite WAN**: Supports USB satellite modems (Starlink, Iridium) for remote deployment.
- **Native Service Hosting**: Runs BGP-X native services (`.bgpx`) directly on the router.

**No user needs to purchase additional hardware** to access any BGP-X capability. The outdoor IP67 carrier board enables all six specialized deployment scenarios (mast, solar, vehicle, maritime, underground, aerial).

---

## 2. Position in Hardware Ecosystem

The BGP-X Router v1 sits at the center of the hardware ecosystem:

| Device | Target User | Key Differentiator |
|---|---|---|
| **BGP-X Router v1** (this spec) | End User (Home/Office/Community) | All-in-one; replaces existing router; full BGP-X node capability. |
| BGP-X Node v1 | Community Contributor | Compact outdoor node; solar-native; runs full daemon but no built-in LAN switching. |
| BGP-X Gateway v1 | Provider/Operator | High-throughput; SFP+ uplink; optimized for datacenter deployment. |
| BGP-X Client Node | Endpoint User | Low-cost, battery-powered endpoint; does not route for others. |
| BGP-X Adapter | Peripheral User | USB LoRa dongle for existing computers. |

The Router v1 is designed for users who want a single device that "just works" for all BGP-X features, including transparent protection of all LAN devices without per-device configuration.

---

## 3. Design Philosophy — Adaptable Reference Implementation

The hardware specification below describes the **current reference implementation** used for initial development and production. This is not a permanent silicon commitment.

BGP-X firmware is written against hardware abstraction layers (HAL). The BGP-X daemon is hardware-agnostic Linux software. Any ARM or x86 SoC meeting the capability requirements below can run BGP-X Router v1 firmware with HAL adaptation only.

Community manufacturers, operators, and future BGP-X hardware revisions may substitute:
- Alternative SoCs meeting capability minimums
- Different LoRa transceivers with equivalent performance (SX1261, SX1268, LLCC68, etc.)
- Different WiFi chipsets with 802.11s mesh capability
- Custom BGP-X ASICs in future revisions (optimized for onion encryption and forwarding)
- Any BLE 5.0+ module

**Capability requirements drive compliance — not specific part numbers.**

---

## 4. Capability Requirements (SoC-Agnostic)

Any SoC or compute module used in a BGP-X Router v1 implementation MUST meet the following minimum requirements:

| Capability | Minimum Requirement | Reference Implementation |
|---|---|---|
| **CPU** | Dual-core ARM Cortex-A53 or equivalent, ≥1.0 GHz | MediaTek MT7981B (Filogic 820) dual-core Cortex-A53 @ 1.3 GHz |
| **RAM** | 512 MB | 512 MB DDR4 |
| **Storage** | 4 GB persistent + 64 MB boot minimum | 128 MB SPI NAND (boot) + 8 GB eMMC 5.1 (data) |
| **Crypto Acceleration** | AES hardware acceleration OR ChaCha20-Poly1305 in constant-time software | MT7981B integrated crypto engine; supports AES and SHA acceleration |
| **Hardware RNG** | True Random Number Generator (TRNG) or equivalent HWRNG | SLB9670 TPM 2.0 integrated HWRNG (feeds kernel entropy pool) |
| **Network Interfaces** | ≥1 GbE WAN + ≥1 GbE LAN | 1× GbE WAN (BCM54210 PHY) + 3× GbE LAN (internal switch) |
| **WiFi** | IEEE 802.11s mesh capable; 802.11ax preferred | MT7915 WiFi 6 (802.11ax) with 802.11s support in `mac80211` |
| **LoRa Radio** | SPI-connected LoRa transceiver (SX1261/SX1262 or equivalent) | Semtech SX1262 connected via SPI1 |
| **USB** | ≥2× USB 2.0+ ports for satellite modem, BLE, expansion | 2× USB 3.0 Type-A ports |
| **Security Element** | TPM 2.0 or equivalent Secure Element for hardware-bound key storage | Infineon SLB9670 TPM 2.0 (SPI bus) |
| **Linux Support** | Mainline Linux 5.15+ or vendor BSP with full driver support | OpenWrt 23.05 / Linux 6.1 LTS |

**Satellite Modem Interface**: USB ports support auto-detection of satellite modems by USB Vendor ID. Supported modems include:
- **Starlink Gen 3**: Vendor ID `0x48bf:0x0800` (USB Ethernet adapter mode)
- **Iridium Certus**: Serial modem interface
- **Inmarsat Terminals**: Vendor-specific interface
- **Any USB Ethernet adapter**: Treated as clearnet WAN; satellite latency class configured manually or via geo-detection.

---

## 5. Reference Implementation — Core Hardware

### 5.1 Compute Platform

| Component | Reference Part | Notes |
|---|---|---|
| **SoC** | MediaTek MT7981B (Filogic 820) | Dual-core ARM Cortex-A53 @ 1.3 GHz; NEON SIMD; 512 KB L2 cache |
| **RAM** | 512 MB DDR4 | 16-bit bus; sufficient for BGP-X session tables and DHT storage |
| **Boot Flash** | 128 MB SPI NAND | Stores bootloader, kernel, device tree |
| **User Storage** | 8 GB eMMC 5.1 | Stores BGP-X daemon, logs, configuration, cached DHT records |
| **Security Element** | Infineon SLB9670 TPM 2.0 | SPI bus; provides HWRNG, Ed25519 key storage, PCR measurements |
| **Crypto Engine** | MT7981B integrated | Offloads AES, SHA; ChaCha20-Poly1305 runs in constant-time software |

### 5.2 Network Interfaces

| Interface | Controller | Specification | Domain |
|---|---|---|---|
| **WAN** | BCM54210 PHY | 1× Gigabit Ethernet RJ45 | Clearnet |
| **LAN** | MT7531 Switch | 3× Gigabit Ethernet RJ45 | LAN |
| **WiFi 2.4 GHz** | MT7915 | 802.11ax, 2×2 MIMO, 802.11s mesh capable | Mesh (WiFi domain) |
| **WiFi 5 GHz** | MT7915 | 802.11ax, 2×2 MIMO, 802.11s mesh capable | Mesh (WiFi domain) |
| **LoRa** | SX1262 via SPI | Sub-GHz transceiver, region-selectable by SKU/antenna | Mesh (LoRa domain) |
| **USB 3.0 × 2** | MT7981B EHCI/XHCI | External satellite modem, BLE, additional LoRa adapters | Extension |

**WiFi Mesh Operation**:
- MT7915 supports IEEE 802.11s in `mac80211` (OpenWrt default driver).
- Both 2.4 GHz and 5 GHz radios can operate simultaneously in mesh mode.
- 2.4 GHz preferred for long-range mesh backhaul.
- 5 GHz preferred for high-bandwidth local mesh.
- Compatible with any other 802.11s-capable device.

**LoRa Interface**:
- SX1262 connected via SPI1 (configurable: SPI0 or SPI1 in firmware).
- GPIO-controlled RF switch for TX/RX antenna path.
- SMA female antenna connector (50Ω).
- Frequency band determined by SKU and antenna:
  - **EU SKU**: 868 MHz (ETSI EN 300 220; 1% duty cycle enforced in firmware).
  - **NA SKU**: 915 MHz (FCC Part 15.247).
  - **Asia SKU**: 433 MHz (region-specific; longest range).

**Bluetooth BLE**:
- Integrated BLE 5.0 module (reference: QCA6390 combo chip or standalone USB/UART module).
- Supports BLE advertisement and scanning (MESH_BEACON transport over BLE).
- GATT-based BGP-X BLE transport for short-range mesh.
- Alternatively: external BLE USB dongle via USB 3.0 port.

### 5.3 Trusted Platform Module (TPM 2.0)

The TPM is a mandatory component for the BGP-X Router v1. It provides hardware-enforced security for cryptographic keys.

| Function | Implementation |
|---|---|
| **Node Key Storage** | Ed25519 node private key stored in TPM NV memory; never exposed to Linux userspace. |
| **Hardware Signing** | Ed25519 signature operations performed inside TPM; private key never leaves the chip. |
| **Hardware RNG** | TPM integrated TRNG provides entropy to Linux `/dev/hwrng`. |
| **Boot Integrity** | TPM Platform Configuration Registers (PCRs) measure boot chain; enables sealed storage. |
| **Sealed Storage** | Configuration secrets (e.g., pool memberships, bridge configurations) can be sealed to known-good PCR values. |

**Adaptability Note**: If a future hardware revision substitutes the TPM 2.0 chip (e.g., using an ARM TrustZone-based Secure Element or a custom BGP-X security IC), the bgpx-node daemon interfaces via the `tpm2-tss` TCTI layer. Swapping the TPM requires only a TCTI driver update; the daemon and protocol remain unchanged. If no TPM is present (community builds), the daemon falls back to software key storage with `mlock()` and explicit zeroization — no protocol change, reduced key isolation.

---

## 6. Carrier Boards

The BGP-X Router v1 core PCB mounts to carrier boards for different deployment contexts. The core PCB exposes: SoC, RAM, eMMC, TPM, SX1262, WiFi module, USB headers, Ethernet controller, and power rails. Carriers add: enclosures, external connectors, power input circuitry, antenna cables, and mounting hardware.

**All carriers use the same core PCB.** Swapping carriers changes the deployment form factor; the firmware and configuration are identical.

### 6.1 Desktop Carrier (Home Router)

- **Form Factor**: 220 × 160 × 40 mm
- **Ports**: 4× RJ45 (1 WAN + 3 LAN) front panel; 2× USB-A rear; DC barrel jack; USB-C PD input.
- **Antennas**: 4× internal WiFi dipoles (2× 2.4 GHz, 2× 5 GHz, ~3 dBi each); 1× SMA external LoRa antenna connector.
- **Power**: 12V DC barrel jack OR USB-C PD 65W.
- **IP Rating**: IP20 (indoor).
- **Mount**: Rubber feet (tabletop) or wall mount bracket.
- **Use Case**: Residential, office, community center.

### 6.2 Outdoor IP67 Carrier

- **Form Factor**: 200 × 150 × 80 mm sealed polycarbonate.
- **Ports**:
  - N-type female antenna connector (LoRa).
  - TNC female antenna connectors (WiFi).
  - Ethernet cable gland (pass-through for WAN/LAN).
  - Weatherproof USB-C connector (power/satellite).
- **Power**: PoE 802.3at (30W) via weatherproof RJ45 with cable gland.
- **IP Rating**: IP67 (outdoor; temporary submersion to 1m).
- **Temperature**: -40°C to +70°C.
- **Mount**: Pole clamp (25–60 mm diameter); DIN rail; flat surface.
- **Use Case**: **Primary carrier for mast, solar, vehicle, maritime, aerial deployments.**

### 6.3 DIN Rail Carrier

- **Form Factor**: 4× DIN rail modules (72 mm wide total).
- **Ports**: Screw terminal network connections; SMA LoRa antenna; USB-A.
- **Power**: 12–48V DC wide-input.
- **IP Rating**: IP20 (cabinet).
- **Use Case**: Telecom/distribution cabinets, industrial environments.

### 6.4 Rack Mount Carrier (1U 19")

- **Form Factor**: 1U 19" rack.
- **Ports**: 4× RJ45 front; 2× USB-A; LoRa SMA rear; IEC C14 power inlet.
- **Power**: AC 100–240V 50/60 Hz.
- **IP Rating**: IP20.
- **Use Case**: Server rooms, NOC deployments, datacenter edge.

---

## 7. Configurations

### 7.1 Standard Configuration (Home / Office / Community)

Target: Most users replacing their home or office router.

| Component | Specification |
|---|---|
| SoC | ARM dual-core ≥1.3 GHz (Ref: MT7981B) |
| RAM | 512 MB DDR4 |
| Storage | 8 GB eMMC + 128 MB SPI NAND |
| WAN | 1× GbE |
| LAN | 3× GbE |
| WiFi | 802.11ax, 802.11s capable, dual-band |
| LoRa | SX1262 or equivalent |
| TPM | TPM 2.0 |
| Power | PoE 802.3at or 12V DC |
| Carrier | Desktop (default) or Outdoor IP67 |

**Suitable For**: All six deployment modes; standard relay; WiFi mesh relay; LoRa relay; domain bridge; gateway with WAN.

### 7.2 Extended Configuration (High-Session / Bridge-Heavy)

Target: Operators running the Router v1 as a community-facing domain bridge or high-traffic gateway in a residential context.

| Component | Specification |
|---|---|
| RAM | 1 GB DDR4 (supports larger cross-domain path_id tables) |
| Storage | 16 GB eMMC |
| WAN | 2.5 GbE (optional carrier upgrade) |
| All Others | Same as Standard |

**Suitable For**: High-traffic domain bridge nodes (>5,000 concurrent cross-domain path_id entries); community exit nodes; densely populated mesh islands.

---

## 8. Software Platform

| Component | Specification |
|---|---|
| **Operating System** | OpenWrt 23.05+ (BGP-X customized build) |
| **Kernel** | Linux 6.1 LTS (OpenWrt default for MT7981B) |
| **BGP-X Daemon** | `bgpx-node` (compiled for `aarch64-linux-musl`) |
| **BGP-X CLI** | `bgpx-cli` |
| **Web UI** | LuCI + `bgpx-luci` plugin |
| **WiFi Mesh** | `wpad-mesh-openssl` (802.11s); `mt7915e` driver |
| **LoRa Driver** | `bgpx-sx1262` kernel module (SPI) |
| **TPM Stack** | `tpm2-tools`, `tpm2-tss` |
| **BLE Stack** | BlueZ (optional feature flag) |

### 8.1 First Boot Wizard

On first boot, the Router v1 presents a setup wizard via LuCI at `192.168.1.1`:

1.  **WAN Configuration**: Select type (DHCP, PPPoE, Static IP, USB Satellite Modem).
2.  **WAN Credentials**: Enter ISP credentials if required.
3.  **BGP-X Node Role**: Select (Relay, Relay+Gateway, Domain Bridge).
4.  **Key Generation**: Generate Ed25519 node keypair in TPM.
5.  **Bootstrap Contacts**: Download initial DHT bootstrap node contacts.
6.  **LoRa Region**: Select region (EU-868, NA-915) or auto-detect (if GPS installed).
7.  **WiFi Mesh ID**: Set mesh island ID (or auto-assign).
8.  **Review & Confirm**: Review configuration and apply.

**Total Setup Time**: Under 5 minutes for basic configuration.

### 8.2 Firmware Updates

- **OTA Updates**: Signed firmware images downloaded via BGP-X overlay (preventing ISP observation of update traffic).
- **Local Updates**: Upload signed image via LuCI web interface.
- **Rollback**: Previous firmware partition retained for rollback on failure.
- **TPM Verification**: Firmware signature verified against TPM-stored public key.

---

## 9. Security Architecture

| Threat | Mitigation |
|---|---|---|
| **Node Private Key Extraction** | Ed25519 key stored in TPM NV memory; never exposed to Linux; signing done in TPM hardware. |
| **Key Material in Swap** | Swap disabled; BGP-X session keys `mlock()`'d to RAM. |
| **Cold Boot Attack on Session Keys** | Session keys in `mlock()`'d pages; zeroized on session close; limited temporal window. |
| **Firmware Tampering** | Signed firmware updates only; TPM PCR chain verifies boot; unsigned firmware rejected. |
| **eMMC Data Extraction** | eMMC optionally encrypted with AES-128-CBC key sealed in TPM. |
| **Physical Access (Outdoor)** | IP67 carrier provides tamper-evident sealed enclosure. |
| **Unauthorized Config Changes** | BGP-X control socket requires local authentication; LuCI password-protected. |

---

## 10. Power and Environmental

### 10.1 Power Requirements

| Carrier | Input Voltage | Typical Power | Peak Power |
|---|---|---|---|
| Desktop | 12V DC / USB-C PD 65W | 5W (idle) | 15W (full load) |
| Outdoor IP67 | PoE 802.3at (48V) | 6W (idle) | 18W (full load + LoRa TX) |
| DIN Rail | 12–48V DC | 5W (idle) | 15W (full load) |
| Rack 1U | AC 100–240V | 8W (idle) | 25W (full load + airflow) |

**LoRa TX Power Impact**: LoRa transmission at +22 dBm adds ~2W peak during TX window.

### 10.2 Environmental Ratings

| Carrier | IP Rating | Temperature Range | Notes |
|---|---|---|---|
| Desktop | IP20 | 0°C to +50°C | Indoor use. |
| Outdoor IP67 | IP67 | -40°C to +70°C | Outdoor; temporary submersion. |
| DIN Rail | IP20 | -20°C to +70°C | Industrial cabinet. |
| Rack 1U | IP20 | 0°C to +50°C | Datacenter/server room. |

---

## 11. Antenna Specifications

### 11.1 WiFi Antennas

- **Desktop Carrier**: 4× internal monopole dipoles (2× 2.4 GHz, 2× 5 GHz), ~3 dBi each.
- **Outdoor Carrier**: 4× external RP-SMA connectors (2× 2.4 GHz, 2× 5 GHz); compatible with any RP-SMA antenna.

### 11.2 LoRa Antenna

- **Connector**: SMA female (50Ω).
- **Default Included**: 2 dBi rubber duck antenna (region-appropriate frequency).
- **Upgrade Options**: 5 dBi fiberglass omni; 9 dBi Yagi (directional); 3 dBi magnetic base.
- **Polarization**: Vertical preferred (horizontal mismatch causes ~20 dB loss).
- **Cable**: Any 50Ω coax with SMA male; LMR-200 or LMR-400 for long runs.

### 11.3 LoRa Range Estimates

*Reference: SX1262, +14 dBm TX (limited by regulatory duty cycle), SF9 BW125, 2 dBi antennas both sides.*

| Deployment | Estimated Range |
|---|---|
| Ground level, urban | 1–3 km |
| Rooftop deployment (10–15m) | 5–15 km |
| Mast deployment (30m) | 10–30 km |
| Tower (60m+) | 20–60 km |
| Line-of-sight rural | 15–50 km |

**Note**: These are baseline estimates. Actual range depends on terrain, vegetation, antenna gain, and interference.

---

## 12. Bill of Materials — Approximate Component Cost (Volume)

| Component | Approximate Cost (Volume) |
|---|---|
| ARM SoC module (MT7981B or equivalent) | $12–20 |
| RAM 512 MB DDR4 | $4–8 |
| eMMC 8 GB | $3–6 |
| Boot flash 128 MB NAND | $1–3 |
| TPM 2.0 IC (SLB9670) | $3–5 |
| LoRa transceiver module (SX1262) | $5–10 |
| WiFi module (MT7915) | $8–15 |
| Network switch PHY (MT7531) | $3–6 |
| BLE module (optional) | $2–4 |
| Main PCB (4-layer) | $4–8 |
| Passives, connectors, crystal, regulators | $5–10 |
| Desktop carrier assembly | $8–15 |
| **Total (Standard, Desktop carrier)** | **$58–110** |
| Outdoor IP67 carrier (additional) | $20–35 |

**Note**: These are component-level estimates. Retail pricing including assembly, test, certification, software development, and support will be higher.

---

## 13. Regulatory Compliance

| Regulation | Status | Notes |
|---|---|---|
| **CE (EU RED Directive)** | Planned | Per-SKU (868 MHz LoRa). |
| **FCC Part 15 (US)** | Planned | Per-SKU (915 MHz LoRa). |
| **ISED RSS-247 (Canada)** | Planned | |
| **ETSI EN 300 220 (LoRa EU)** | Planned | 1% duty cycle enforced in firmware. |
| **FCC Part 15.247 (LoRa US)** | Planned | |
| **RoHS** | Planned | Lead-free. |
| **CERN-OHL-S v2** | Yes | Reference design published under CERN-OHL-S v2. |

**Operator Responsibility**: Operators are responsible for compliance with LoRa frequency regulations in their jurisdiction. The LoRa driver enforces duty cycle limits; operators must select the correct regional frequency SKU. Custom implementations must independently certify their hardware.

---

## 14. Manufacturing Test Requirements

Every BGP-X Router v1 unit must pass the following tests before shipment:

1.  **Power-On Self Test (POST)**: Voltage rails within tolerance; TPM detected.
2.  **TPM Self-Test**: TPM2_SelfTest() returns success; HWRNG generating entropy.
3.  **Ethernet Link Test**: All 4 ports (WAN+LAN) establish Gigabit link.
4.  **WiFi RF Test**: TX power within regulatory limit; packet transmission verified.
5.  **LoRa Loopback Test**: SX1262 internal loopback; RF output verified (conducted test).
6.  **USB Port Test**: USB 3.0 ports enumerate test device.
7.  **Flash Write/Read Test**: eMMC write and verify test pattern.
8.  **Thermal Test**: Run at full load for 1 hour; verify no thermal throttling.
9.  **Firmware Boot Test**: Boot from NAND to OpenWrt; BGP-X daemon starts.
10. **Cross-Domain Bridge Test (Factory)**: Verify packet can be received on clearnet interface and queued for LoRa transmission (and vice versa). *Validates DOMAIN_BRIDGE capability.*

---

## 15. Document References

- `hardware/hardware_spec.md` — Hardware taxonomy and ecosystem overview.
- `hardware/node_spec.md` — BGP-X Node v1 specification (Community contributor device).
- `hardware/gateway_spec.md` — BGP-X Gateway v1 specification (Provider infrastructure).
- `firmware/router_firmware.md` — Firmware build and configuration.
- `protocol/routing_domains.md` — Domain ID specification (clearnet = 0x00000001).
- `deployment/router_setup.md` — End-user setup guide.
