# BGP-X Router v1 Hardware Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X Router v1 is the **all-in-one primary product** for end users. It replaces a standard home or office router entirely. One device provides every capability defined in the BGP-X architecture:

- ISP WAN connection (clearnet domain)
- LAN routing and switching for connected devices
- BGP-X overlay relay participation
- WiFi 802.11s mesh networking
- LoRa radio mesh networking
- Bluetooth BLE mesh
- Domain bridge node (any combination of its interfaces)
- Mesh island gateway
- Satellite WAN (via USB satellite modem)
- BGP-X native service hosting

No user needs to purchase additional hardware to access any BGP-X capability. The outdoor IP67 carrier board enables all six specialized deployment scenarios (mast, solar, vehicle, maritime, underground, aerial).

---

## 2. Design Philosophy — Adaptable Reference Implementation

The hardware specification below describes the **current reference implementation** used for initial development and production. This is not permanent silicon commitment.

BGP-X firmware is written against hardware abstraction layers. The BGP-X daemon is hardware-agnostic Linux software. Any ARM or x86 SoC meeting the capability requirements below can run BGP-X Router v1 firmware with HAL adaptation only. Community manufacturers, operators, and future BGP-X hardware revisions may substitute:

- Alternative SoCs meeting capability minimums
- Different LoRa transceivers with equivalent performance (SX1261, SX1268, LLCC68, etc.)
- Different WiFi chipsets with 802.11s mesh capability
- Custom BGP-X ASIC in future revisions
- Any BLE 5.0+ module

**Capability requirements drive compliance — not specific part numbers.**

---

## 3. Capability Requirements (SoC-Agnostic)

Any SoC or compute module used in a BGP-X Router v1 implementation MUST meet:

| Capability | Minimum | Reference Implementation |
|---|---|---|
| CPU | Dual-core ARM Cortex-A53 or equivalent, ≥1.0 GHz | MT7981B dual-core 1.3 GHz |
| RAM | 512 MB | 512 MB DDR4 |
| Storage | 4 GB persistent + 128 MB boot | 128 MB NAND + 8 GB eMMC |
| Crypto acceleration | AES hardware or ChaCha20 in constant-time SW | MT7981B crypto engine |
| Hardware RNG | TRNG or equivalent (HWRNG) | TPM 2.0 integrated HWRNG |
| Network interfaces | ≥1 GbE WAN + ≥1 GbE LAN | 1× GbE WAN + 3× GbE LAN |
| WiFi | 802.11s mesh capable, 802.11ax preferred | MT7915 WiFi 6 |
| LoRa | SPI-connected LoRa transceiver (SX1261/SX1262 or equivalent) | SX1262 |
| USB | ≥2× USB 2.0+ for satellite modem, BLE module, LoRa adapter | 2× USB 3.0 |
| Security element | TPM 2.0 or equivalent SE for key storage | SLB9670 TPM 2.0 |
| Linux support | Mainline Linux 5.15+ or vendor BSP with full driver support | OpenWrt 23.05 / Linux 6.1 |

---

## 4. Reference Implementation — Core Hardware

### Compute Platform

| Component | Reference Part | Minimum Equivalent |
|---|---|---|
| SoC | MediaTek MT7981B dual-core ARM Cortex-A53 @ 1.3 GHz | Any dual-core ARM Cortex-A53 ≥1.0 GHz |
| RAM | 512 MB DDR4 | 512 MB LPDDR3/4/4X |
| Flash (boot) | 128 MB SPI NAND | 64 MB SPI NOR or equivalent |
| Storage (daemon) | 8 GB eMMC 5.1 | 4 GB eMMC or SD/NVMe |
| Security element | SLB9670 TPM 2.0 (dedicated IC) | Any FIPS 140-2 certified TPM 2.0 |
| Hardware RNG | TPM integrated HWRNG | Any TRNG feeding kernel entropy pool |

**Note on adaptability**: The MT7981B is the reference SoC. Future revisions may use the MT7986 (higher performance), custom BGP-X ASIC (lower power, onion crypto acceleration), or any ARM SoC with Linux BSP and equivalent capability. The firmware build system uses Kconfig for hardware selection; adding a new SoC target requires updating the device tree and HAL, not the daemon or protocol code.

### Network Interfaces

| Interface | Specification | Domain |
|---|---|---|
| WAN | 1× Gigabit Ethernet (RJ45, BCM54210 PHY) | Clearnet |
| LAN | 3× Gigabit Ethernet (RJ45, internal switch) | LAN |
| WiFi 2.4 GHz | MT7915 802.11ax, 2×2 MIMO, 802.11s mesh | Mesh (WiFi domain) |
| WiFi 5 GHz | MT7915 802.11ax, 2×2 MIMO, 802.11s mesh | Mesh (WiFi domain) |
| LoRa | SX1262 via SPI, region-selectable, SMA antenna | Mesh (LoRa domain) |
| USB 3.0 × 2 | External adapters: satellite modem, BLE, additional LoRa | Satellite / BLE / extension |

**WiFi mesh**: MT7915 supports IEEE 802.11s in mac80211. Both 2.4 GHz and 5 GHz can operate simultaneously in mesh mode. 2.4 GHz used for long-range mesh backhaul; 5 GHz for high-bandwidth mesh when nodes are closer. Compatible with any other 802.11s-capable device.

**LoRa**: SX1262 connected via SPI1. BGP-X firmware includes SX1262 driver as kernel module. Antenna connector: SMA female. Frequency: hardware-selectable by SKU jumper:
- EU SKU: 868 MHz (ETSI EN 300 220, 1% duty cycle enforced)
- NA SKU: 915 MHz (FCC Part 15.247)
- Asia SKU: 433 MHz (region-specific)

**Satellite WAN**: USB 3.0 port connects to satellite modem. BGP-X daemon detects satellite modems by USB vendor ID at startup:
- Starlink Gen 3: `0x48bf:0x0800` (USB Ethernet adapter mode)
- Iridium Certus: serial modem, driver loaded by vendor ID
- Inmarsat terminals: vendor-specific
- Any USB Ethernet adapter or LTE modem works as WAN; satellite-class latency configuration is software

### LoRa Specifications (SX1262 Reference)

| Parameter | Specification |
|---|---|
| Chipset | Semtech SX1262 (reference) — or SX1261, SX1268, LLCC68 equivalent |
| Max output power | +22 dBm (158 mW) |
| Receive sensitivity | -148 dBm at SF12 BW125 |
| Spreading factors | SF5 – SF12 |
| Bandwidths | 125, 250, 500 kHz |
| Data rates | ~250 bps (SF12) to ~37.5 Kbps (SF5 BW500) |
| Interface to SoC | SPI (configurable: SPI0 or SPI1) |
| Antenna port | SMA female (50Ω) |

### Bluetooth BLE

Integrated BLE 5.0 module (reference: QCA6390 combo chip or standalone BLE module on USB/UART). Supports:
- BLE advertisement and scanning (MESH_BEACON via BLE)
- GATT-based BGP-X BLE transport for short-range mesh
- BLE is an optional mesh transport; not required for basic BGP-X operation
- Alternatively: external BLE USB dongle via USB 3.0 port

### Trusted Platform Module (TPM 2.0)

| Function | Implementation |
|---|---|
| Ed25519 node key storage | TPM NV memory; private key never exposed to OS |
| Ed25519 signing | Hardware signing in TPM; SPI or I2C bus |
| HWRNG | TPM integrated TRNG; feeds Linux /dev/hwrng |
| PCR measurements | Boot integrity; sealed storage |
| Sealed key storage | Configuration secrets sealed to known-good boot state |

**Adaptability note**: TPM 2.0 chip may be substituted with any SE (secure element) or HSM module implementing TPM 2.0 commands. The bgpx-node daemon uses `tpm2-tss` library for TPM operations. If no TPM is present (community builds), bgpx-node falls back to software key storage with mlock() and explicit zeroization — no protocol change, reduced key isolation.

---

## 5. Carrier Boards

The BGP-X Router v1 core PCB mounts to carrier boards for different deployment contexts. The core PCB exposes: SoC, RAM, eMMC, TPM, SX1262, WiFi module, USB headers, Ethernet controller, power rails. Carriers add: enclosures, connectors, power input circuitry, antenna cables, and mounting hardware.

All carriers use the same core PCB. Swapping carriers changes the deployment form factor; the firmware and configuration are identical.

### Desktop Carrier (Home Router)

- Form factor: 220 × 160 × 40 mm
- Ports: 4× RJ45 front panel, 2× USB-A rear, DC barrel jack + USB-C PD input
- Antennas: 4× internal WiFi dipoles + 1× SMA external LoRa antenna
- Power: 12V DC or USB-C PD 65W
- IP rating: IP20 (indoor)
- Mount: rubber feet (tabletop) or wall mount bracket

### Outdoor IP67 Carrier

- Form factor: 200 × 150 × 80 mm sealed polycarbonate
- Ports: N-type female antenna (LoRa), TNC female (WiFi), Ethernet cable gland
- Power: PoE 802.3at (30W) via weatherproof RJ45 with cable gland
- IP rating: IP67 (outdoor, temporary submersion)
- Temperature: -40°C to +70°C
- Mount: pole clamp (25-60 mm diameter), DIN rail, flat surface
- **Primary carrier for mast, solar, vehicle, maritime, aerial deployments**

### DIN Rail Carrier

- Form factor: 4× DIN rail modules (72 mm wide)
- Ports: screw terminal network connections, SMA LoRa, USB-A
- Power: 12-48V DC wide-input
- IP rating: IP20 (cabinet)
- For deployment in telecom/distribution cabinets

### Rack Mount Carrier (1U 19")

- Form factor: 1U 19" rack
- Ports: 4× RJ45 front, 2× USB-A, LoRa SMA rear, IEC power inlet
- Power: AC 100-240V 50/60 Hz
- IP rating: IP20
- For server rooms, NOC deployments

---

## 6. Configurations

### Standard (Home / Office / Community)

Target: most users replacing their home or office router.

| Component | Specification |
|---|---|
| SoC | ARM dual-core ≥1.3 GHz (ref: MT7981B) |
| RAM | 512 MB |
| Storage | ≥8 GB eMMC + ≥64 MB boot flash |
| WAN | 1× GbE |
| LAN | ≥3× GbE |
| WiFi | 802.11ax, 802.11s capable, dual-band |
| LoRa | SX1262 or equivalent |
| TPM | TPM 2.0 |
| Power | PoE 802.3at or 12V DC |
| Carrier | Desktop (default) or outdoor IP67 |

Suitable for: all six deployment modes; standard relay; WiFi mesh relay; LoRa relay; domain bridge; gateway with WAN.

### Extended (High-Session / Bridge-Heavy)

Target: operators running the Router v1 as a community-facing domain bridge or high-traffic gateway in residential context.

| Component | Specification |
|---|---|
| RAM | 1 GB (larger cross-domain path_id tables) |
| Storage | ≥16 GB eMMC |
| WAN | 2.5GbE (optional carrier upgrade) |
| All others | Same as Standard |

Suitable for: high-traffic domain bridge nodes (>5,000 concurrent cross-domain path_id entries); community exit nodes.

---

## 7. Software Platform

| Component | Specification |
|---|---|
| Operating System | OpenWrt 23.05+ (BGP-X customized build) |
| Kernel | Linux 6.1 LTS (OpenWrt default for MT7981B) |
| BGP-X daemon | bgpx-node (compiled for aarch64-linux-musl) |
| BGP-X CLI | bgpx-cli |
| Web UI | LuCI + bgpx-luci plugin |
| WiFi mesh | wpad-mesh-openssl (802.11s), mt7915e driver |
| LoRa driver | bgpx-sx1262 kernel module (SPI) |
| TPM | tpm2-tools, tpm2-tss |
| BLE | BlueZ (optional feature flag) |

### First Boot Wizard

On first boot, the Router v1 presents a setup wizard via LuCI at `192.168.1.1`:

1. Select WAN type (DHCP, PPPoE, static IP, USB satellite modem)
2. Set WAN credentials if needed
3. Choose BGP-X node role (relay, relay+gateway, domain bridge)
4. Generate Ed25519 node keypair in TPM
5. Download initial DHT bootstrap contacts
6. Configure LoRa region (EU-868, NA-915, or auto-detect via firmware GPS if installed)
7. Set WiFi mesh island ID (or auto-assign)
8. Review and confirm configuration

Total setup time: under 5 minutes for basic configuration.

---

## 8. Security Architecture

| Threat | Mitigation |
|---|---|
| Node private key extraction | Ed25519 key stored in TPM NV; never exposed to Linux; signing done in TPM hardware |
| Key material in swap | Swap disabled; BGP-X session keys mlock()'d to RAM |
| Cold boot attack on session keys | Session keys in mlock()'d pages; zeroized on session close; limited window |
| Firmware tampering | Signed firmware updates only; TPM PCR chain verifies boot; unsigned firmware rejected |
| eMMC data extraction | eMMC encrypted with AES-128-CBC key sealed in TPM |
| Physical access | IP67 carrier provides tamper-evident sealed enclosure for outdoor units |
| Unauthorized config changes | BGP-X control socket requires local authentication; LuCI password-protected |

**Adaptability note on TPM**: If a future hardware revision substitutes the TPM 2.0 chip (e.g., using an ARM TrustZone-based secure element or a custom BGP-X security IC), the bgpx-node daemon interfaces via the same `tpm2-tss` TCTI layer. Swapping the TPM requires only a TCTI driver update — no daemon changes.

---

## 9. Antenna Specifications

### WiFi Antennas

Desktop carrier: 4× internal monopole dipoles (2× 2.4 GHz, 2× 5 GHz), ~3 dBi each.
Outdoor carrier: 4× external RP-SMA (2× 2.4 GHz, 2× 5 GHz); compatible with any RP-SMA 2.4/5 GHz antenna.

### LoRa Antenna

- Connector: SMA female (50Ω)
- Default included antenna: 2 dBi rubber duck (region-appropriate frequency)
- Upgrade options: 5 dBi fiberglass omni, 9 dBi Yagi (directional), 3 dBi magnetic base
- Polarization: vertical preferred; horizontal polarization mismatch causes ~20 dB loss
- Cable: any 50Ω coax with SMA male termination; LMR-200 or LMR-400 for long runs

### LoRa Range (SX1262 reference, +14 dBm TX, SF9 BW125, 2 dBi antennas both sides)

| Deployment | Estimated Range |
|---|---|
| Ground level, urban | 1-3 km |
| Rooftop deployment (10-15m) | 5-15 km |
| Mast deployment (30m) | 10-30 km |
| Tower (60m+) | 20-60 km |
| Line-of-sight rural | 15-50 km |

---

## 10. Bill of Materials — Approximate Component Cost (Volume)

| Component | Approximate Cost (Volume) |
|---|---|
| ARM SoC module (MT7981B or equivalent) | $12–20 |
| RAM 512 MB | $4–8 |
| eMMC 8 GB | $3–6 |
| Boot flash 128 MB NAND | $1–3 |
| TPM 2.0 IC | $3–5 |
| LoRa transceiver module (SX1262) | $5–10 |
| WiFi module (MT7915 or equivalent) | $8–15 |
| Network switch (GbE) | $3–6 |
| BLE module (optional) | $2–4 |
| Main PCB (4-layer) | $4–8 |
| Passives, connectors, crystal, regulators | $5–10 |
| Desktop carrier assembly | $8–15 |
| **Total (Standard, Desktop carrier)** | **$58–110** |
| Outdoor IP67 carrier (additional) | $20–35 |

These are component-level estimates. Retail pricing including assembly, test, certification, software, and support will be higher.

---

## 11. Regulatory Compliance

| Regulation | Status | Notes |
|---|---|---|
| CE (EU Red Directive) | Planned | Per-SKU (868 MHz LoRa) |
| FCC Part 15 (US) | Planned | Per-SKU (915 MHz LoRa) |
| ISED RSS-247 (Canada) | Planned | |
| ETSI EN 300 220 (LoRa EU) | Planned | 1% duty cycle enforced in firmware |
| FCC Part 15.247 (LoRa US) | Planned | |
| RoHS | Planned | |
| CERN-OHL-S (open hardware) | Yes | Reference design published under CERN-OHL-S v2 |

Operators are responsible for compliance with LoRa frequency regulations in their jurisdiction. The LoRa driver enforces duty cycle limits in firmware; operators must select the correct regional frequency SKU. Custom implementations must independently certify their hardware.
