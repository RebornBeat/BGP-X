# BGP-X Hardware

**Version**: 0.1.0-draft

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

BGP-X hardware is organized into four categories:

### Category 1: BGP-X Router v1 — End User AIO

**Target**: Individual users, families, home offices, small organizations

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

**Target**: Community mesh contributors, outdoor relay operators, range extenders, organizations expanding mesh coverage

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

**Target**: ISPs, hosting providers, enterprise operators, organizations running public BGP-X infrastructure

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

### Category 4: BGP-X OpenWrt Package — Compatible Third-Party Hardware

**Target**: Users who already own compatible OpenWrt routers; community members wanting to contribute without new hardware purchase

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

## 3. Hardware Selection Guide

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

---

## 4. Satellite WAN Integration

BGP-X Router v1 and BGP-X Node v1 (with WAN) support **satellite WAN connections** via USB satellite modems:

- **Starlink Gen 3**: USB port on terminal; presents as USB Ethernet adapter; BGP-X daemon detects via USB vendor ID; treats as clearnet domain with LEO latency class (20-40ms)
- **Iridium Certus 100/350**: Serial modem interface; BGP-X satellite transport driver handles AT commands; GEO latency class (600ms+)
- **Inmarsat BGAN/FBB**: Similar serial/IP interface; GEO latency class

**Important**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) provide **internet connectivity over BGP**. From BGP-X's perspective, they are clearnet transports — the same clearnet domain as fiber or cellular, just with higher latency. They are not a separate mesh routing domain. See `satellite_architecture.md` for the full architecture and future vision.

---

## 5. Firmware Overview

| Firmware | Target Hardware | BGP-X Role |
|---|---|---|
| bgpx-router-firmware | BGP-X Router v1 (ARM-class SoC) | AIO end-user; all capabilities |
| bgpx-node-firmware | BGP-X Node v1 (low-power ARM) | Community relay, bridge, range extender |
| bgpx-gateway-firmware | BGP-X Gateway v1 (high-throughput SoC) | Provider exit/bridge |
| bgpx-openwrt | Any OpenWrt 23.05+ compatible device | Software relay/bridge |
| bgpx-modem-firmware | ESP32-S3 / nRF52840 (Meshtastic-compatible) | LoRa radio modem (USB serial) |

All firmware targets run the same bgpx-node daemon binary built for the appropriate target architecture. Firmware is the OpenWrt/Linux environment + drivers + daemon + LuCI UI. The daemon itself is cross-compiled from the same Rust source for all targets.

---

## 6. Power Summary

| Device | Idle | Peak | Primary Power |
|---|---|---|---|
| BGP-X Router v1 (desktop) | 5W | 15W | 12V DC or PoE 802.3at |
| BGP-X Router v1 (outdoor) | 5W | 15W | PoE 802.3at |
| BGP-X Node v1 (LoRa only) | 0.5W | 2W | Solar + LiPo battery |
| BGP-X Node v1 (WiFi + LoRa) | 2W | 8W | Solar + LiPo or PoE |
| BGP-X Node v1 (all radios) | 3W | 12W | PoE 802.3at or solar |
| BGP-X Gateway v1 | 15W | 40W | AC 100-240V / PoE 802.3bt |

---

## 7. Environmental Ratings

| Device | IP Rating | Temperature |
|---|---|---|
| BGP-X Router v1 (Desktop carrier) | IP20 | 0°C to +50°C |
| BGP-X Router v1 (Outdoor IP67 carrier) | IP67 | -40°C to +70°C |
| BGP-X Router v1 (DIN Rail carrier) | IP20 | -20°C to +70°C |
| BGP-X Node v1 | IP67 | -40°C to +85°C |
| BGP-X Gateway v1 (1U Rack) | IP20 | 0°C to +50°C |
| BGP-X Gateway v1 (Outdoor) | IP65 | -20°C to +60°C |

---

## 8. Open Hardware

BGP-X Router v1 and BGP-X Node v1 reference designs are released under **CERN-OHL-S v2** (Strongly Reciprocal Open Hardware License). This means:

- Anyone may manufacture BGP-X-compatible hardware using the reference designs
- Modifications must be released under the same license
- Manufactured units must include source files or instructions for obtaining them
- The BGP-X project makes no warranty on third-party manufactured hardware

Reference design files (KiCad schematics, PCB layouts, BOM, Gerbers) will be published in the `hardware/reference-designs/` directory.

---

## 9. Procurement

BGP-X native hardware (Router v1, Node v1, Gateway v1) will be available from:
- Community manufacturers building from open reference designs
- The BGP-X hardware program (details published at testnet launch)

Compatible third-party hardware is commercially available from GL.iNet, Raspberry Pi, and other vendors. See `compatible_hardware.md`.

Meshtastic hardware for LoRa adaptation is available from LILYGO, Heltec, RAK Wireless, and other vendors. See `meshtastic_adapter.md`.
