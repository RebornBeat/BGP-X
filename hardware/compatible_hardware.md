# BGP-X Compatible Hardware

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X runs on a wide range of commodity hardware. This document lists tested and community-verified compatible devices for running the BGP-X daemon (`bgpx-node`).

All listed devices run standard Linux (OpenWrt or Ubuntu/Debian). The BGP-X daemon has no hardware-specific dependencies beyond:
- ARM or x86 processor
- 512MB RAM minimum
- 4GB storage minimum
- Network interface(s)
- Linux 5.15+ kernel

**Domain Bridge Support column**: indicates whether the device supports domain bridge operation (bridging two or more routing domains simultaneously). Domain bridge support requires appropriate network interfaces for each domain to be bridged.

**Satellite Internet Clarification**: Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) are **clearnet domain** in BGP-X. They provide BGP-routed IP connectivity. From the protocol's perspective, a satellite WAN is a clearnet interface with high latency, not a separate routing domain.

---

## 2. OpenWrt Routers

### GL.iNet GL-MT6000 (Flint 2)

| Parameter | Value |
|---|---|
| SoC | MediaTek MT7988A quad-core 1.8 GHz |
| RAM | 1 GB DDR4 |
| Storage | 8 GB eMMC |
| Network | 2× 2.5GbE WAN/LAN, 4× GbE LAN |
| WiFi | MT7986 WiFi 6, 802.11ax, 802.11s capable |
| USB | 1× USB 3.0 |
| OS | OpenWrt 23.05+ |
| BGP-X package | bgpx-node (OpenWrt ipk) |
| **Domain Bridge Support** | **Yes (native)** — WiFi 802.11s (mesh domain) + 2.5GbE WAN (clearnet domain) simultaneously; LoRa domain requires USB adapter |
| Notes | Recommended for standard relay, WiFi mesh bridge nodes, and gateway roles. High throughput. |

### GL.iNet GL-MT3000 (Beryl AX)

| Parameter | Value |
|---|---|
| SoC | MT7981B dual-core 1.3 GHz |
| RAM | 512 MB DDR4 |
| Storage | 256 MB NAND |
| Network | 1× 2.5GbE WAN, 1× GbE LAN |
| WiFi | MT7915E WiFi 6, 802.11ax, 802.11s capable |
| USB | 1× USB 3.0 |
| OS | OpenWrt 23.05+ |
| BGP-X package | bgpx-node (OpenWrt ipk) |
| **Domain Bridge Support** | **Yes (limited)** — WiFi 802.11s + clearnet WAN; 512MB RAM limits cross-domain path_id table to ~5,000 entries |
| Notes | Travel router; suitable for mesh+clearnet bridge in low-traffic environments. Low power, compact. |

### GL.iNet AXT1800 (Slate AX)

| Parameter | Value |
|---|---|
| SoC | IPQ6000 quad-core 1.2 GHz (Qualcomm) |
| RAM | 512 MB DDR3 |
| Storage | 256 MB NAND |
| Network | 1× GbE WAN, 3× GbE LAN |
| WiFi | IPQ6000 WiFi 6, 802.11s capable |
| USB | 1× USB 3.0 |
| OS | OpenWrt 23.05+ |
| BGP-X package | bgpx-node (OpenWrt ipk) |
| **Domain Bridge Support** | **Yes (limited)** — WiFi 802.11s + clearnet WAN; same RAM constraint as MT3000 |
| Notes | Good performance for relay and entry node roles. |

### Banana Pi BPi-R3

| Parameter | Value |
|---|---|
| SoC | MT7986A quad-core ARM Cortex-A53 1.8 GHz |
| RAM | 2 GB DDR4 |
| Storage | SPI NAND 128 MB + M.2 NVMe slot |
| Network | 2× 2.5GbE + 1× 10GbE (SFP+) + 4× GbE |
| WiFi | MT7915 WiFi 6, 802.11s capable |
| USB | 2× USB 3.0, 1× USB-C |
| OS | OpenWrt 23.05+ or Ubuntu |
| BGP-X package | bgpx-node (OpenWrt ipk or Debian deb) |
| **Domain Bridge Support** | **Yes (full)** — 2GB RAM, multiple interfaces; recommended for high-traffic domain bridge nodes; add USB LoRa adapter for LoRa domain |
| Notes | Developer-friendly; M.2 NVMe slot allows large storage for high-traffic nodes. Best performance. |

---

## 3. Raspberry Pi and Single-Board Computers

### Raspberry Pi 5 (4GB or 8GB)

| Parameter | Value |
|---|---|
| SoC | Broadcom BCM2712 quad-core ARM Cortex-A76 2.4 GHz |
| RAM | 4 GB or 8 GB LPDDR4X |
| Storage | MicroSD or M.2 NVMe HAT |
| Network | 1× GbE RJ45 |
| WiFi | BCM43455 WiFi 5 (802.11ac) — NOT 802.11s; see note |
| USB | 2× USB 3.0, 2× USB 2.0 |
| OS | Ubuntu 24.04 LTS, Debian 12 |
| BGP-X package | bgpx-node (Debian deb) |
| **Domain Bridge Support** | **Yes (with adapters)** — clearnet via GbE; WiFi mesh requires USB WiFi adapter (802.11s capable); LoRa via USB adapter or SPI HAT |
| Notes | Built-in WiFi is NOT 802.11s capable; Meshtastic USB adapter enables LoRa domain. High performance. |

### Raspberry Pi 4 (2GB, 4GB, or 8GB)

| Parameter | Value |
|---|---|
| SoC | BCM2711 quad-core ARM Cortex-A72 1.8 GHz |
| RAM | 2 GB / 4 GB / 8 GB |
| Network | 1× GbE + WiFi 5 (not 802.11s) |
| OS | Ubuntu 22.04+ / Debian 11+ |
| BGP-X package | bgpx-node (Debian deb) |
| **Domain Bridge Support** | **Yes (with adapters)** — same as Pi 5; clearnet via GbE; mesh requires external adapters |
| Notes | Lower priority than Pi 5; still functional for all BGP-X use cases. |

### Orange Pi 5 (4GB or 8GB)

| Parameter | Value |
|---|---|
| SoC | RK3588S octa-core (4× Cortex-A76, 4× Cortex-A55) |
| RAM | 4 GB / 8 GB / 16 GB LPDDR4 |
| Network | 1× GbE |
| OS | Ubuntu 22.04 / Debian 11 |
| BGP-X package | bgpx-node (Debian deb) |
| **Domain Bridge Support** | **Yes (with adapters)** — clearnet via GbE; WiFi mesh and LoRa via USB adapters |
| Notes | Very high performance for SBC class. |

---

## 4. x86 Servers and Mini-PCs

### Any x86 Linux Server (Ubuntu 24.04 / Debian 12)

| Parameter | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores, 1 GHz | 4 cores, 2 GHz |
| RAM | 512 MB | 1 GB (2 GB for high-traffic bridge nodes) |
| Storage | 4 GB | 16 GB SSD |
| Network | 1× GbE | 2× GbE or 1× 10GbE |
| BGP-X package | bgpx-node (Debian/Ubuntu deb) | — |
| **Domain Bridge Support** | **Yes (clearnet-to-clearnet trivially)** — for mesh bridge, add USB WiFi (802.11s) or USB LoRa adapter |

Standard VPS or dedicated server. Suitable for high-capacity relay nodes and exit gateways. Add USB adapters for mesh domain bridge operation.

---

## 5. 802.11s-Capable USB WiFi Adapters (for Mesh Domain)

These adapters enable WiFi 802.11s mesh domain support on devices whose built-in WiFi does not support 802.11s (e.g., Raspberry Pi).

| Device | Chipset | 802.11s | Notes |
|---|---|---|---|
| ALFA AWUS036AXM | MT7921AU | ✅ | WiFi 6; well-supported in Linux mac80211 |
| ALFA AWUS036ACM | MT7612U | ✅ | WiFi 5; reliable 802.11s support |
| TP-Link TL-WN722N (v1 only) | AR9271 | ✅ | Older device; v2/v3 NOT compatible |
| Generic MT7601U adapters | MT7601U | Limited | 802.11s support varies by firmware |

**Important**: Not all USB WiFi adapters support 802.11s. Verify chipset compatibility with mac80211 802.11s before purchase. MT7612U and MT7921AU have best Linux support.

---

## 6. USB LoRa Adapters (for LoRa Domain)

These adapters enable LoRa domain participation on devices without native SX1262.

| Device | Chipset | Interface | Notes |
|---|---|---|---|
| RAK5146 USB | SX1303 | USB | Full LoRa concentrator; 8-channel; enterprise grade |
| RAK2287 Mini PCIe | SX1302 | mPCIe | For devices with mPCIe slot (BPi-R3, some servers) |
| DRAGINO DLOS8 | SX1301 | USB | Compatible with BGP-X LoRa driver |
| Custom SX1262 USB adapter | SX1262 | USB-serial | BGP-X reference design SX1262 USB adapter (see `meshtastic_adapter.md` for ESP32 variant) |

**Note on LoRa adapters**: Multi-channel concentrators (RAK5146, RAK2287) allow the BGP-X node to communicate with multiple LoRa nodes simultaneously on different channels. Single-channel SX1262 adapters are simpler and lower-cost but can only communicate on one channel at a time. For mesh island gateway operation, multi-channel concentrators are recommended.

---

## 7. Domain Bridge Support Summary

| Device | Clearnet | WiFi Mesh | LoRa | BLE | Bridge Native |
|---|---|---|---|---|---|
| BGP-X Node v1 | ✅ | ✅ | ✅ | ✅ via USB | ✅ All combos |
| GL-MT6000 | ✅ | ✅ | ✅ via USB | ✅ via USB | ✅ (WiFi+clearnet native) |
| GL-MT3000 | ✅ | ✅ | ✅ via USB | ✅ via USB | ⚠️ RAM limited |
| BPi-R3 | ✅ | ✅ | ✅ via USB/mPCIe | ✅ via USB | ✅ (best for high traffic) |
| Raspberry Pi 5 | ✅ | ✅ via USB | ✅ via USB | ✅ via USB | ✅ (with adapters) |
| Raspberry Pi 4 | ✅ | ✅ via USB | ✅ via USB | ✅ via USB | ✅ (with adapters) |
| x86 Server | ✅ | ✅ via USB | ✅ via USB | ✅ via USB | ✅ (with adapters) |

---

## 8. Hardware Not Compatible

The following hardware types are NOT compatible with BGP-X:

- **Consumer routers with <512 MB RAM** (many sub-$30 routers): insufficient memory for BGP-X daemon and cross-domain path_id tables.
- **MIPS-based OpenWrt routers** (older Atheros/Qualcomm): BGP-X Rust binary requires ARMv7+/x86-64; MIPS target not supported in v1.
- **Windows or macOS servers**: BGP-X daemon requires Linux for TUN interface, SO_REUSEPORT, mlock(), and mesh transport drivers.

---

## 9. Hardware Selection Guide

| Priority | Recommended Hardware |
|---|---|
| Best overall (home/office) | GL.iNet GL-MT6000 |
| Best portability | GL.iNet GL-MT3000 |
| Best throughput | Banana Pi BPi-R3 |
| Best for PoE mast deploy | GL-MT3000 + PoE splitter |
| Best solar | GL-MT3000 (lowest power) |
| Best cost/performance | GL.iNet GL-MT3000 |
| Vehicle/marine | GL-MT3000 in IP67 enclosure |
| Domain bridge (WiFi+clearnet) | GL-MT6000 or BPi-R3 |
| Domain bridge (LoRa+clearnet) | Any compatible device + USB LoRa adapter |
| SBC development/testing | Raspberry Pi 5 |

---

## 10. Community Hardware Reports

Community-verified hardware reports are maintained in the BGP-X repository. Submit hardware compatibility reports via the standard contribution process.

Format for compatibility reports:
```
Device: <manufacturer> <model> (<year>)
SoC: <chip>
RAM: <size>
OS: <distro + version>
BGP-X version: <version>
Domains tested: clearnet / wifi-mesh / lora / ble
Domain bridge tested: yes/no; which pairs
Notes: <any issues or recommendations>
Contributor: <github handle> (<date>)
```
