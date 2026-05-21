# BGP-X Hardware

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2 (Hardware), MIT (Firmware/Software), CC BY 4.0 (Documentation)

---

## 1. Design Philosophy — Adaptable and Portable

BGP-X hardware specifications define **capability requirements and reference implementations**, not locked silicon. The chips and board designs documented here represent the current reference implementation used for development and initial production. Future revisions may substitute:

- Alternative SoCs with equivalent or superior specifications
- Custom BGP-X ASICs (optimized for onion encryption and forwarding pipeline)
- Different radio chipsets with equivalent LoRa/WiFi/BLE capability
- Custom board designs for specialized deployments

All BGP-X firmware is written against hardware abstraction layers. The BGP-X daemon is hardware-agnostic Linux software. No component of the software stack assumes specific silicon. When hardware changes, only the HAL (hardware abstraction layer) in firmware needs updating — the routing daemon, protocol stack, and all specifications remain unchanged.

**This means**: communities, manufacturers, and operators are free to build BGP-X-compatible hardware using the capability specifications in this directory. Compliance with capability requirements (not chip part numbers) is what matters.

---

## 2. Hardware Ecosystem Overview

BGP-X hardware is organized into six categories, spanning from infrastructure-grade routing nodes to low-cost client endpoints.

### Hardware Roles Summary

| Role | Function | Form Factor | Tier |
|---|---|---|---|
| BGP-X Router v1 | End-user AIO; replaces home router | Desktop / Outdoor IP67 | Tier 1 |
| BGP-X Node v1 | Community mesh relay, domain bridge | Outdoor IP67 | Tier 1 |
| BGP-X Gateway v1 | Provider exit/relay infrastructure | Rack / Outdoor | Tier 1 |
| BGP-X Client Node | Low-cost mesh endpoint | Handheld / sensor mount | Tier 2 |
| BGP-X Adapter/Dongle | USB LoRa modem for existing computers | USB dongle | Tier 3 |
| OpenWrt Package | Software-only for compatible routers | Any | Software |

---

### Category 1: BGP-X Router v1 — End User AIO

**Target**: Individual users, families, home offices, small organizations.

The BGP-X Router v1 is the **all-in-one primary product**. It replaces a standard home or office router entirely. A user purchases one BGP-X Router v1 and it handles everything:

- WAN connection to ISP (clearnet domain)
- LAN routing for all connected devices
- BGP-X overlay participation (clearnet relay node)
- WiFi 802.11s mesh networking (mesh domain)
- LoRa radio mesh networking (mesh domain via integrated SX1262)
- Bluetooth BLE mesh (mesh domain via integrated BLE)
- Domain bridge node (clearnet ↔ mesh, any combination of its interfaces)
- Mesh island gateway (when configured with WAN + mesh radio)
- Native service hosting (BGP-X native services via SDK)
- Satellite WAN (via USB satellite modem — Starlink USB, Iridium Certus, etc.)

A user who owns a BGP-X Router v1 needs **no additional hardware** for any of the six deployment modes defined in the architecture. The outdoor carrier board (IP67) enables mast, solar, maritime, underground, vehicle, and aerial deployments.

**Full specification**: `router_spec.md`

---

### Category 2: BGP-X Node v1 — Community Contributor

**Target**: Community mesh contributors, outdoor relay operators, range extenders, organizations expanding mesh coverage.

The BGP-X Node v1 is a compact, multi-radio BGP-X node designed for community deployment. Unlike the Router v1 (which replaces a home router), the Node v1 is designed to be **added to an existing network** to expand BGP-X mesh coverage.

Key distinction from Router v1:
- Does NOT replace a home router (no multi-port LAN switching, no NAT by default)
- IS a full BGP-X routing node — runs complete bgpx-node daemon
- Solar and battery optimized (low-power ARM platform)
- Default IP67 outdoor enclosure — built for elevated deployment
- Optional WAN port enables gateway/domain bridge mode
- Configurable radio mix: LoRa only, WiFi only, BLE only, or any combination

Configuration modes (software-selectable):
- **Mesh Relay**: routes BGP-X traffic via WiFi/LoRa/BLE mesh; no WAN needed
- **Domain Bridge**: with WAN Ethernet, bridges mesh to clearnet
- **Range Extension**: prioritizes forwarding over session origination; lower resource use
- **Community Gateway**: with WAN, provides clearnet exit for island mesh users

This device is what community members deploy on rooftops, masts, in distribution boxes, or on solar-powered poles to extend mesh coverage. A group of Node v1 devices in a neighborhood can form a complete mesh island, with one or more connected to WAN acting as the gateway.

**Full specification**: `node_spec.md`

---

### Category 3: BGP-X Gateway v1 — Provider Infrastructure

**Target**: ISPs, hosting providers, enterprise operators, organizations running public BGP-X infrastructure.

The BGP-X Gateway v1 is high-throughput provider infrastructure. It is NOT a consumer device — it is for operators who want to run public exit nodes, high-capacity domain bridge nodes, or backbone relay nodes.

Key characteristics:
- Rack-mount or outdoor weatherproof enclosure
- MT7988A quad-core ARM Cortex-A53 @ 1.8 GHz (or equivalent high-throughput SoC)
- 1 GB RAM, SFP+ uplink, 2×2.5GbE WAN
- Optimized for >1 Gbps clearnet throughput
- High concurrent session capacity
- No integrated LoRa on board (datacenter context; USB LoRa available if needed)
- Intended for colocation facilities, server rooms, or outdoor installation by providers

A typical deployment: an operator colocates a BGP-X Gateway v1 in an EU datacenter and configures it as a certified public exit node with a signed exit policy. The device handles thousands of simultaneous BGP-X exit connections while the datacenter's fiber uplink provides the bandwidth.

**Full specification**: `gateway_spec.md`

---

### Category 4: BGP-X Client Node — Low-Cost Mesh Endpoint (Tier 2)

**Target**: Individual users, sensor deployments, portable field operation, IoT devices, community members contributing personal endpoints without routing for others.

The BGP-X Client Node is a **low-cost, battery-powered endpoint** for connecting to BGP-X mesh networks. It does not route traffic for others. It is designed for individual users, sensors, and portable use.

**Reference Hardware**: LILYGO T3S3, LILYGO T-Beam, LILYGO T-Echo, Heltec V3, RAK WisBlock 4631. These are ESP32-S3 or nRF52840 based development boards with integrated SX1262 LoRa radios.

Key characteristics:
- **Microcontroller-based**: ESP32-S3 (Xtensa LX7, 240 MHz) or nRF52840 (ARM Cortex-M4)
- **Insufficient for full bgpx-node daemon**: Runs BGP-X Client Firmware — a subset implementation
- **Battery-native**: Integrated LiPo charging; solar panel input
- **Low cost**: $25-40 retail for reference hardware
- **No routing for others**: Endpoint only; does not participate in DHT storage or relay traffic

**What the Client Node CAN do**:
- Connect to a BGP-X mesh island as a client
- Send and receive BGP-X encrypted traffic
- Query the DHT (no storage)
- Request paths to services
- Operate on battery for extended periods

**What the Client Node CANNOT do**:
- Route traffic for other nodes
- Store DHT records
- Participate as a relay in BGP-X paths
- Run the full bgpx-node daemon (insufficient RAM/CPU)

**Firmware**: BGP-X Client Firmware (ESP-IDF or Arduino framework). Implements client-side BGP-X protocol only.

**Full specification**: `client_node_spec.md`

---

### Category 5: BGP-X Adapter/Dongle — USB LoRa Modem (Tier 3)

**Target**: Users with existing computers (laptops, desktops, servers) who want to connect to a BGP-X mesh island without purchasing dedicated routing hardware.

The BGP-X Adapter is a **USB dongle that acts as a LoRa radio modem**. It contains no routing intelligence. The host computer runs the full BGP-X daemon; the dongle handles only LoRa transmission and reception.

Key characteristics:
- **Form factor**: USB-A or USB-C dongle
- **Core hardware**: ESP32-S3 or nRF52840
- **Radio**: SX1262 LoRa transceiver
- **Interface**: USB CDC-ACM (virtual serial port) or USB HID
- **Cost**: $15-25 target
- **Firmware**: BGP-X Modem Firmware — a minimal binary that accepts AT-style commands for TX/RX

**How it works**:
```
Host Computer (bgpx-node daemon)
         │
         │ USB serial (CDC-ACM)
         ▼
BGP-X Adapter (ESP32-S3 + SX1262)
         │
         │ LoRa radio
         ▼
BGP-X Mesh Island
```

The host computer handles all crypto, DHT queries, path construction, and session management. The adapter is a transparent pipe to the LoRa radio. This allows any laptop or server with a USB port to become a BGP-X mesh client or domain bridge node (when combined with clearnet WAN on the host).

**Use cases**:
- Developer testing BGP-X on a desktop workstation
- User with a laptop connecting to a mesh island via LoRa
- Turning an OpenWrt router without LoRa into a domain bridge node

**Full specification**: `adapter_spec.md`

---

### Category 6: BGP-X OpenWrt Package — Compatible Third-Party Hardware

**Target**: Users who already own compatible OpenWrt routers; community members wanting to contribute without new hardware purchase.

BGP-X runs as a software package on any compatible OpenWrt 23.05+ router. No hardware purchase required beyond what the user already owns.

Compatible hardware: GL.iNet GL-MT6000, GL-MT3000, AXT1800; Banana Pi BPi-R3; Raspberry Pi 4/5; x86 servers.

Capabilities without adapters:
- Clearnet relay node
- WiFi 802.11s mesh (if device WiFi chip supports mac80211 mesh mode)

Capabilities with USB adapters:
- LoRa mesh (USB LoRa adapter or Meshtastic USB adapter)
- Domain bridge node (clearnet + LoRa or clearnet + WiFi mesh)

**Adapter documentation**: `meshtastic_adapter.md`, `compatible_hardware.md`

---

## 3. Capability Requirements (Hardware-Agnostic)

These specifications define the minimum requirements for each hardware tier. Any implementation claiming compliance must meet these capabilities. Specific part numbers listed under "Reference Implementation" are the current recommended choices, not mandatory requirements.

### Tier 1 Capability Requirements

| Capability | Minimum | Router v1 Reference | Node v1 Reference | Gateway v1 Reference |
|---|---|---|---|---|
| **CPU** | Dual-core ARM Cortex-A53, ≥1.0 GHz | MT7981B, A53 @ 1.3 GHz | MT7981B, A53 @ 1.0-1.3 GHz | MT7988A, A73 @ 1.8 GHz |
| **RAM** | 256 MB (relay), 512 MB (bridge), 1 GB (gateway) | 512 MB DDR4 | 256-512 MB DDR4 | 1 GB DDR4 |
| **Storage (boot)** | 64 MB | 128 MB NAND | 64-128 MB NAND | 128 MB NAND |
| **Storage (data)** | 2 GB | 8 GB eMMC | 4-8 GB eMMC | 16 GB eMMC |
| **Network** | 1× GbE minimum | 1× WAN + 3× LAN GbE | Optional 1× GbE WAN | SFP+ + 2× 2.5GbE + 4× LAN |
| **WiFi** | 802.11s capable (optional on Gateway) | MT7915 (WiFi 6) | MT7915 or MT7612U | Optional via USB |
| **LoRa** | SPI-connected SX1261/62/68 | SX1262 integrated | SX1262 integrated | USB adapter required |
| **BLE** | BLE 5.0+ | Integrated module | Integrated module | Optional USB |
| **Security** | TPM 2.0 recommended | SLB9670 TPM 2.0 | Optional TPM 2.0 | SLB9670 TPM 2.0 |
| **USB** | 1× USB 2.0+ | 2× USB 3.0 | 1× USB 2.0 | 2× USB 3.0 |
| **Linux Support** | Mainline 5.15+ or vendor BSP | OpenWrt 23.05 | OpenWrt 23.05 | OpenWrt 23.05 |

### Tier 2 Capability Requirements (Client Node)

| Capability | Minimum | Reference Implementation |
|---|---|---|
| MCU | 240 MHz dual-core microcontroller | ESP32-S3 (Xtensa LX7) or nRF52840 (ARM Cortex-M4) |
| RAM | 256 KB SRAM (minimum) | 512 KB SRAM + 8 MB PSRAM |
| Flash | 4 MB | 16 MB |
| LoRa | SPI-connected SX1262 or SX1276 | SX1262 |
| WiFi | 802.11 b/g/n | Integrated ESP32-S3 WiFi |
| BLE | BLE 5.0+ | Integrated ESP32-S3 BLE |
| USB | USB-C or Micro-USB | USB-C (OTG supported) |
| Power | LiPo battery connector + charging | Internal LiPo + USB charging |
| Security | Optional ATECC608 | ATECC608B (some variants) |

### Tier 3 Capability Requirements (Adapter)

| Capability | Minimum | Reference Implementation |
|---|---|---|
| MCU | ARM Cortex-M4 or Xtensa LX7, ≥240 MHz | ESP32-S3 or nRF52840 |
| RAM | 256 KB | 512 KB SRAM |
| Flash | 1 MB | 4-16 MB |
| LoRa | SPI-connected SX1262 | SX1262 |
| USB | USB 2.0 Full Speed (12 Mbps) | USB 2.0 FS (native in ESP32-S3) |
| Interface | USB CDC-ACM or USB HID | USB CDC-ACM |
| Antenna | External SMA or integrated PCB | SMA female |
| Power | USB bus power (max 500 mA) | 5V @ 200 mA peak |

---

## 4. Reference Implementation — Core Hardware Details

### 4.1 BGP-X Router v1 Core

| Component | Specification |
|---|---|
| SoC | MediaTek MT7981B (Filogic 820) |
| CPU | ARM Cortex-A53 dual-core, 1.3 GHz |
| RAM | 512 MB DDR4 |
| Storage | 128 MB SPI NAND (boot) + 8 GB eMMC 5.1 (data) |
| Security | Infineon SLB9670 TPM 2.0 |
| Crypto Engine | MT7981B integrated (AES, SHA acceleration) |

### 4.2 BGP-X Node v1 Core

| Component | Specification |
|---|---|
| SoC | MediaTek MT7981B (primary) or Allwinner H3/H5, NXP i.MX8M Mini (low-power) |
| CPU | ARM Cortex-A53 dual-core or quad-core, 800-1300 MHz |
| RAM | 256-512 MB DDR4 / LPDDR4 |
| Storage | 64-128 MB NAND + 4-8 GB eMMC |
| Security | Optional TPM 2.0 (Infineon SLB9670/SLB9672) |

### 4.3 BGP-X Gateway v1 Core

| Component | Specification |
|---|---|
| SoC | MediaTek MT7988A (Filogic 880) |
| CPU | ARM Cortex-A73 quad-core, 1.8 GHz |
| NPU | MT7988A integrated Network Processing Unit |
| RAM | 1 GB DDR4 @ 3200 MHz |
| Storage | 128 MB SPI NAND + 16 GB eMMC 5.1 |
| Security | Infineon SLB9670 TPM 2.0 |

### 4.4 Gateway v1 Network Interfaces

| Interface | Specification | Use |
|---|---|---|
| SFP+ | 10 Gbps fiber/copper cage | Primary WAN — fiber ISP uplink |
| WAN 1 | 2.5GbE RJ45 | Secondary WAN or failover |
| WAN 2 | 2.5GbE RJ45 | Secondary WAN or bonding |
| LAN × 4 | Gigabit Ethernet RJ45 | Local management / testing |
| mPCIe | 1× mPCIe slot | 4G/LTE module for WAN backup |

### 4.5 Radio Modules

#### WiFi Module (Integrated on Router and Node)

| Parameter | Specification |
|---|---|
| Chipset | MediaTek MT7915 (WiFi 6, 802.11ax) |
| Bands | 2.4 GHz + 5 GHz simultaneous |
| 802.11s | Full mesh profile support |
| Antennas | 2 × U.FL or RP-SMA connectors |
| Max power | 20 dBm |

#### LoRa Module (Integrated on Router and Node)

| Parameter | Specification |
|---|---|
| Chipset | Semtech SX1262 |
| Frequency | Region SKU: 433, 868, 915 MHz |
| Output power | Up to +22 dBm |
| Sensitivity | -148 dBm (SF12) |
| Interface | SPI |

#### Optional BLE Module

| Parameter | Specification |
|---|---|
| Chipset | Nordic nRF52840 |
| Protocol | Bluetooth 5.0 LE |
| Max power | +8 dBm |
| Interface | UART or SPI |

---

## 5. Enclosure Variants

### Router v1 Enclosures

| Variant | IP Rating | Mounting | Use Case |
|---|---|---|---|
| Indoor Desktop | IP20 | None | Home, office |
| Outdoor Mast | IP67 | Mast/pole | Outdoor deployment |
| Maritime | IP68 | Bulkhead | Marine/waterfront |
| Industrial | IP67/ATEX | DIN rail | Industrial/hazardous |
| Mobile/Vehicle | IP65 | Vehicle mount | Mobile mesh |

### Node v1 Enclosures

| Variant | IP Rating | Mounting | Use Case |
|---|---|---|---|
| Standard | IP67 | Pole/Wall | Community outdoor relay |
| Industrial | IP67/ATEX | DIN rail | Industrial environments |

### Gateway v1 Enclosures

| Variant | IP Rating | Mounting | Use Case |
|---|---|---|---|
| 1U Rack | IP20 | 19" rack | Datacenter colocation |
| Outdoor | IP65 | Pole/Wall | Remote operator deployment |
| Maritime | IP68 | Bulkhead | Marine/offshore |

---

## 6. Domain Bridge Hardware Requirements

Domain bridge nodes require:
- Both a clearnet network interface (WAN)
- And at least one mesh radio interface (WiFi mesh or LoRa)

The GL.iNet GL-MT3000 with external LoRa USB module is the recommended minimum-cost domain bridge node. The BGP-X Gateway v1 is recommended for production provider deployments.

**Domain Bridge Hardware Requirements by Transport**:

| Target Domain Transport | Additional Hardware Required |
|---|---|
| WiFi 802.11s mesh | 802.11s-capable WiFi radio (most modern WiFi chips) |
| LoRa mesh | LoRa transceiver module or USB adapter (SX1262/1276 recommended) |
| Bluetooth BLE | BLE 5.0+ adapter |
| Satellite | Compatible satellite modem with API access |

**CPU/RAM for bridge nodes**: Domain bridge nodes run two concurrent path_id tables (single-domain and cross-domain). Each table entry is ~128 bytes. At 10,000 concurrent path_id entries per table: ~2.5 MB per table. Bridge nodes handling high cross-domain traffic should provision at least 1 GB RAM; 2 GB recommended for production.

---

## 7. Mesh Island Gateway Hardware Requirements

**Use case**: Bridge between a mesh island and clearnet; publishes island to unified DHT.

**Requirements beyond domain bridge**:
- Both clearnet (ISP WAN) and mesh radio (WiFi 802.11s or LoRa) operational simultaneously
- Sufficient LoRa antenna height and gain for island coverage

**Minimum**: 1 BGP-X Node v1 or equivalent + WAN connection + elevated antenna mount.
**Recommended**: 2+ independent gateways from different operators.

---

## 8. Satellite WAN Integration

BGP-X Router v1 and BGP-X Node v1 (with WAN) support **satellite WAN connections** via USB satellite modems:

- **Starlink Gen 3**: USB port on terminal; presents as USB Ethernet adapter; BGP-X daemon detects via USB vendor ID; treats as clearnet domain with LEO latency class (20-40ms)
- **Iridium Certus 100/350**: Serial modem interface; BGP-X satellite transport driver handles AT commands; GEO latency class (600ms+)
- **Inmarsat BGAN/FBB**: Similar serial/IP interface; GEO latency class

---

## 9. Commercial Hardware (Recommended)

For operators not building custom hardware, these commercial devices run BGP-X firmware:

| Device | Tier | Best For |
|---|---|---|
| GL.iNet GL-MT6000 (Flint 2) | Tier 1 | Gateway, high-throughput relay |
| GL.iNet GL-MT3000 (Beryl AX) | Tier 1 | Relay, indoor node |
| GL.iNet GL-AXT1800 (Slate AX) | Tier 1 | Relay, travel gateway |
| Banana Pi BPi-R3 | Tier 1 | High-performance relay |
| GL.iNet GL-AR750S (Slate) | Tier 2 | Lightweight client |
| Raspberry Pi 4 / 5 | Tier 1 | Standalone relay, development |
| x86 mini PC | Tier 1 | High-performance relay, gateway |

See `compatible_hardware.md` for complete list with specifications and installation notes.

---

## 10. Compatible Third-Party Hardware

BGP-X daemon runs on existing hardware:

| Hardware | BGP-X Support |
|---|---|
| GL.iNet GL-MT3000 | Full (including 802.11s) |
| GL.iNet GL-MT6000 | Full |
| Raspberry Pi 4 / 5 | Full |
| Orange Pi R1 Plus | Full |
| Banana Pi BPi-R3 | Full |
| x86 mini PC | Full |
| Turris Omnia | Full |
| OpenWrt-compatible | Via opkg package |
| Meshtastic devices | LoRa modem only (via adaptation layer) |

**Domain Bridge on Third-Party Hardware**: The GL.iNet GL-MT3000 with external LoRa USB module is the recommended minimum-cost domain bridge node. The BGP-X Gateway v1 is recommended for production provider deployments.

---

## 11. Hardware Selection Guide

### I want to protect my home network and all my devices

→ **BGP-X Router v1**. Plug it in where your current home router is. It replaces it. All devices on your LAN are automatically protected.

### I want to contribute mesh coverage in my neighborhood

→ **BGP-X Node v1**. Mount it on your rooftop or a pole. Configure LoRa + WiFi mesh. It becomes a relay node in your local mesh island. No ISP connection required at the node if the island has a gateway elsewhere.

### I want to be a gateway for my community (provide internet exit)

→ **BGP-X Node v1** with WAN Ethernet connected to your ISP. Configure as domain bridge + gateway. Your Node v1 becomes the mesh island's internet gateway.

### I already have a compatible router and want to contribute

→ **BGP-X OpenWrt Package**. Install bgpx-node on your existing router. Add USB LoRa adapter for mesh capability.

### I'm an ISP or operator running public BGP-X infrastructure

→ **BGP-X Gateway v1**. High-throughput, rack-mountable, SFP+ uplink. Built for provider deployments.

### I want to extend LoRa range in my mesh island cheaply

→ **BGP-X Node v1** in Range Extension mode with LoRa radio only. Mount at elevation. Solar-powered. Forwards BGP-X traffic between LoRa peers.

### I want a portable device to connect to the mesh from my laptop

→ **BGP-X Client Node (Tier 2)** or **BGP-X Adapter (Tier 3)**. The Client Node is a standalone battery-powered device. The Adapter plugs into your laptop's USB port. Both provide LoRa connectivity to the mesh without routing for others.

### I want to deploy a sensor or IoT device on the mesh

→ **BGP-X Client Node (Tier 2)**. Low-cost, battery or solar powered, runs client firmware only. Ideal for sensors, environmental monitoring, or asset tracking on a BGP-X mesh island.

---

## 12. Firmware and Software

| Hardware | OS | BGP-X Component |
|---|---|---|
| BGP-X Router v1 | OpenWrt 23.05+ (customized) | bgpx-node (full daemon) |
| BGP-X Node v1 | OpenWrt 23.05+ (customized) | bgpx-node (full daemon) |
| BGP-X Gateway v1 | OpenWrt 23.05+ or Debian 12 | bgpx-node + bgpx-gateway |
| BGP-X Client Node | ESP-IDF / Arduino | bgpx-client-firmware (subset) |
| BGP-X Adapter | ESP-IDF / Arduino | bgpx-modem-firmware (modem only) |
| GL.iNet GL-MT6000 | OpenWrt 23.05+ | bgpx-node (OpenWrt package) |
| Raspberry Pi 4/5 | Ubuntu 24.04 / Debian 12 | bgpx-node (Debian package) |
| x86 server | Ubuntu 24.04 / Debian 12 | bgpx-node (Debian package) |
| Android | Android 11+ | bgpx-sdk (embedded mode) |
| iOS | iOS 16+ | bgpx-sdk (embedded mode) |

---

## 13. Power Requirements Summary

| Device | Idle Power | Max Power | PoE | Solar |
|---|---|---|---|---|
| BGP-X Router v1 (Desktop) | 5W | 15W | 802.3at (30W) | Compatible (12V) |
| BGP-X Router v1 (Outdoor) | 5W | 15W | 802.3at | PoE 802.3at |
| BGP-X Node v1 (LoRa only) | 0.5W | 2W | No | Solar + LiPo battery |
| BGP-X Node v1 (WiFi + LoRa) | 2W | 8W | No | Solar + LiPo or PoE |
| BGP-X Node v1 (All radios) | 3W | 12W | PoE 802.3at or solar | PoE 802.3at or solar |
| BGP-X Gateway v1 | 15W | 40W | AC 100-240V / PoE 802.3bt | AC/POE |
| BGP-X Client Node | 0.1W | 1W | USB bus power | Solar + LiPo |
| BGP-X Adapter | 0.05W | 0.5W | USB bus power | No |

---

## 14. Enclosure and Environmental Ratings

| Device | IP Rating | Temperature | Mounting |
|---|---|---|---|
| BGP-X Router v1 (Desktop carrier) | IP20 | 0°C to +50°C | Tabletop |
| BGP-X Router v1 (Outdoor IP67 carrier) | IP67 | -40°C to +70°C | Pole/Mast/Wall |
| BGP-X Router v1 (DIN Rail carrier) | IP20 | -20°C to +70°C | DIN Rail |
| BGP-X Node v1 | IP67 | -40°C to +85°C | Pole/Mast/Wall |
| BGP-X Gateway v1 (1U Rack) | IP20 | 0°C to +50°C | 19" Rack |
| BGP-X Gateway v1 (Outdoor) | IP65 | -20°C to +60°C | Pole/Wall |
| BGP-X Client Node | IP20-IP65 (variant-dependent) | -20°C to +60°C | Handheld / Fixed |
| BGP-X Adapter | IP20 | 0°C to +50°C | USB dongle |

---

## 15. Open Hardware

BGP-X Router v1, Node v1, and Gateway v1 reference designs are released under **CERN-OHL-S v2** (Strongly Reciprocal Open Hardware License). This means:

- Anyone may manufacture BGP-X-compatible hardware using the reference designs
- Modifications must be released under the same license
- Manufactured units must include source files or instructions for obtaining them
- The BGP-X project makes no warranty on third-party manufactured hardware

Reference design files (KiCad schematics, PCB layouts, BOM, Gerbers) will be published in the `hardware/reference-designs/` directory.

---

## 16. Procurement

BGP-X native hardware (Router v1, Node v1, Gateway v1) will be available from:
- Community manufacturers building from open reference designs
- The BGP-X hardware program (details published at testnet launch)

BGP-X Client Node and Adapter hardware:
- Reference designs for custom builds (CERN-OHL-S)
- Pre-built devices available from LILYGO, Heltec, RAK Wireless, and other vendors

Compatible third-party hardware is commercially available from GL.iNet, Raspberry Pi, and other vendors. See `compatible_hardware.md`.

Meshtastic hardware for LoRa adaptation is available from LILYGO, Heltec, RAK Wireless, and other vendors. See `meshtastic_adapter.md`.
