# BGP-X Client Node (Tier 2) Hardware Specification

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2

---

## 1. Overview

The BGP-X Client Node is a **low-cost, battery-powered endpoint** for connecting to BGP-X mesh networks. It is designed for individual users, sensors, and portable applications. It does **not** route traffic for others and does not store DHT data.

The Client Node serves as a bridge between a user's device (laptop, smartphone, sensor) and the BGP-X mesh infrastructure. It handles the physical radio layer (LoRa, WiFi, BLE) and the minimal protocol layer required to decrypt traffic intended for it, while offloading all routing decisions and path construction to the connected host or the mesh infrastructure.

### What the Client Node IS

- An endpoint device for connecting to a BGP-X mesh island
- A multi-radio interface: LoRa, WiFi, BLE (any combination)
- Battery-native and solar-capable: designed for field and portable use
- A "stupid pipe" for the BGP-X protocol: it modulates packets, but does not route
- Low cost: target <$40 for the complete device
- Supported by existing development hardware (LILYGO, Heltec, RAK)

### What the Client Node is NOT

- NOT a routing node (no relay for third-party traffic)
- NOT a DHT participant (no storage or forwarding of discovery records)
- NOT a replacement for the BGP-X Node v1 (Community Node) or Router v1
- NOT capable of running the full `bgpx-node` daemon (requires Linux/SoC)

---

## 2. Design Philosophy — Leverage Existing Hardware

The BGP-X Client Node specification defines capability requirements for the endpoint use case. Unlike Tier 1 hardware, we do not design a new PCB for this tier by default. Instead, we provide **reference firmware** for existing, widely-available development boards.

**Rationale**:
1.  **Cost**: Development boards (LILYGO T3S3, Heltec V3) are mass-produced and sold near BOM cost.
2.  **Availability**: Existing devices are in stock globally; no manufacturing lead time.
3.  **Ecosystem**: Leverages existing accessory ecosystem (antennas, cases, battery connectors).
4.  **Openness**: Users can flash BGP-X firmware on hardware they already own.

Custom PCBs designed specifically for BGP-X Client Node compliance are permitted and encouraged for specific applications (e.g., industrial sensors, ruggedized field units), but the reference implementation is software for existing hardware.

---

## 3. Capability Requirements (Hardware-Agnostic)

Any implementation claiming BGP-X Client Node compliance MUST meet:

| Capability | Minimum | Reference Implementation |
|---|---|---|
| CPU | 240 MHz dual-core microcontroller | ESP32-S3 (Xtensa LX7) |
| RAM | 256 KB SRAM (minimum) | 512 KB SRAM + 8 MB PSRAM |
| Flash | 4 MB | 16 MB (typical on reference boards) |
| LoRa | SPI-connected LoRa transceiver | SX1262 (primary) / SX1276 (supported) |
| WiFi | 802.11 b/g/n | Integrated ESP32-S3 WiFi |
| BLE | BLE 5.0+ | Integrated ESP32-S3 BLE |
| USB | USB-C or Micro-USB | USB-C (OTG supported) |
| Power | LiPo battery connector + charging circuit | Internal LiPo + USB charging |
| Security | Optional ATECC608 (or similar) Secure Element | ATECC608B (on some T-Beam variants) |

**Protocol Subset Implementation**:
The firmware on the Client Node MUST implement:
- **Session Management**: Establishing a session with the mesh entry node.
- **Onion Decryption**: Decrypting the outermost layer of incoming packets.
- **Path Request**: Requesting a path to a destination (client-side path construction logic runs on the connected host device, or uses a default "connect to nearest gateway" behavior for standalone mode).
- **Transport Abstraction**: Switching between LoRa, WiFi, and BLE as required.

The firmware MUST NOT include:
- **DHT Storage**: No routing table storage or peer discovery for the network.
- **Relay Logic**: No forwarding of traffic for other network participants.
- **Path Construction for Others**: It does not build paths for third parties.

---

## 4. Configuration Modes

### Mode 1: USB Client (Tethered)

**Use Case**: A laptop or desktop user connecting to a BGP-X mesh.

**Setup**:
```
[Computer] ─── USB ─── [Client Node]
                            │
                            └── LoRa/WiFi ─── BGP-X Mesh Island
```

**Behavior**:
- Client Node appears as a network interface (USB CDC-ECM or RNDIS) or serial port.
- The computer runs the BGP-X client software (or full `bgpx-node` daemon).
- Client Node acts as a radio modem: packets from the computer are transmitted over LoRa/WiFi.

**Firmware**: `bgpx-modem-firmware`

### Mode 2: Standalone Client (IoT / Sensor)

**Use Case**: A remote sensor or IoT device sending data to a mesh island gateway.

**Setup**:
```
[Sensor] ─── Serial/GPIO ─── [Client Node]
                                  │
                                  └── LoRa ─── BGP-X Mesh Island Gateway
```

**Behavior**:
- Client Node runs minimal BGP-X client logic.
- Pre-configured with destination ServiceID or gateway NodeID.
- Wakes from deep sleep, transmits data, receives ACK, returns to sleep.

**Firmware**: `bgpx-client-firmware`

### Mode 3: Mobile Client (Bluetooth)

**Use Case**: A smartphone user connecting to a mesh island.

**Setup**:
```
[Smartphone] ─── BLE ─── [Client Node]
                             │
                             └── LoRa ─── BGP-X Mesh Island
```

**Behavior**:
- Smartphone runs BGP-X mobile app.
- Client Node acts as a LoRa-BLE bridge.
- Phone handles path construction and crypto; Node handles LoRa transmission.

**Firmware**: `bgpx-modem-firmware` (BLE variant)

---

## 5. Reference Hardware — Compute

### Primary Reference Platform: ESP32-S3

The ESP32-S3 is the target architecture for BGP-X Client Node firmware.

| Feature | Specification |
|---|---|
| Core | Xtensa LX7 dual-core, 240 MHz |
| RAM | 512 KB SRAM |
| PSRAM | 8 MB (common on T3S3/T-Beam Supreme) |
| Flash | 16 MB (typical) |
| WiFi | 802.11 b/g/n, 2.4 GHz |
| BLE | BLE 5.0 |
| USB | USB-C OTG |
| Security | Secure Boot, Flash Encryption, optional ATECC608 |

**Why ESP32-S3?**
- Sufficient RAM for BGP-X onion decryption buffers.
- Dual-core allows one core for radio timing, one for protocol logic.
- Native WiFi and BLE eliminate need for extra modules.
- Low power modes (deep sleep < 10 µA).
- Wide availability on reference boards.

### Secondary Reference Platform: nRF52840

For devices emphasizing BLE and ultra-low power over LoRa (if LoRa added via SPI).

| Feature | Specification |
|---|---|
| Core | ARM Cortex-M4F, 64 MHz |
| RAM | 256 KB |
| Flash | 1 MB |
| BLE | BLE 5.0 |
| LoRa | Requires external SPI module |

**Reference Device**: LILYGO T-Echo (nRF52840 + SX1262 + e-ink display).

---

## 6. Reference Hardware — Supported Devices

The following devices are officially supported targets for BGP-X Client Node firmware.

| Device | LoRa Chip | CPU | Battery | GPS | Screen | Status |
|---|---|---|---|---|---|---|
| **LILYGO T3S3** | SX1262 | ESP32-S3 | Optional | No | OLED | **Primary Target** |
| **LILYGO T-Beam Supreme** | SX1262 | ESP32-S3 | Yes | Yes | OLED | Supported |
| **LILYGO T-Echo** | SX1262 | nRF52840 | Yes | Yes | E-Ink | Supported |
| **Heltec V3** | SX1262 | ESP32-S3 | Optional | No | OLED | Supported |
| **RAK WisBlock 4631** | SX1262 | nRF52840 | Yes | Yes | No | Supported |
| **LILYGO T-Beam (Classic)** | SX1276 | ESP32 | Yes | Yes | OLED | Supported (Legacy LoRa Chip) |

**Note on SX1276 vs SX1262**:
BGP-X Client Node firmware supports both chipsets. SX1262 is preferred for better sensitivity (-148 dBm vs -137 dBm) and power efficiency.

---

## 7. Reference Hardware — Radios

### LoRa Radio

| Parameter | Reference (SX1262) | Alternative (SX1276) |
|---|---|---|
| Frequency | 433 / 868 / 915 MHz (SKU dependent) | 433 / 868 / 915 MHz |
| Max Power | +22 dBm | +20 dBm |
| Sensitivity | -148 dBm | -137 dBm |
| Interface | SPI | SPI |

**Antenna**: All reference devices have SMA or u.FL connectors for external antennas. A 3-5 dBi fiberglass antenna is recommended for maximum range.

### WiFi Radio

Integrated in ESP32-S3. Used for:
1. Connecting to local WiFi (client mode) for configuration.
2. Direct WiFi-mesh connection if infrastructure is nearby.

### Bluetooth LE

Integrated in ESP32-S3 / nRF52840. Used for:
1. Smartphone pairing (BGP-X app).
2. BLE transport for mesh traffic (short range).

---

## 8. Power System

### Battery Operation

All supported reference devices include LiPo battery connectors and charging circuits.

**Typical Configuration**:
- Battery: 18650 LiPo 3000-5000 mAh.
- Charging: USB-C 5V input.
- Solar: Optional 5-6V solar panel connected to USB or dedicated solar input (if device supports).

### Power Management (ESP32-S3)

The Client Node firmware implements aggressive power management:

| State | Power Draw | Description |
|---|---|---|
| Active (Transmitting) | 150-300 mA | LoRa TX @ +22 dBm, CPU active |
| Active (Receiving) | 40-80 mA | LoRa RX, CPU idle |
| Light Sleep | 0.8-2 mA | CPU halted, WiFi/BLE off, LoRa in RX or sleep |
| Deep Sleep | 10-50 µA | All radios off, RTC only |

**Battery Life Example (3000 mAh, 1 transmission per minute, LoRa-only)**:
- Active TX burst: 200 mA for 1s = 0.055 mAh
- RX wait: 50 mA for 1s = 0.014 mAh
- Deep sleep: 10 µA for 58s = ~0.0002 mAh
- **Total per minute**: ~0.07 mAh
- **Estimated runtime**: ~3000 / (0.07 * 60 * 24) = **~30 days**.

### Solar Charging

A 2W-5W solar panel (typical portable panel) is sufficient to maintain battery charge with moderate transmission intervals (1-10 minutes). For always-on receiving nodes, a larger panel (5-10W) is recommended.

---

## 9. Physical Form Factor

Client Nodes are typically used in the form factor of the reference device. Custom enclosures are optional.

### Reference Device Form Factors

| Device | Dimensions | Weight | Enclosure Option |
|---|---|---|---|
| LILYGO T3S3 | 58 x 28 mm | ~12g | 3D printed case |
| LILYGO T-Beam | 79 x 48 mm | ~50g | 3D printed case |
| Heltec V3 | 66 x 33 mm | ~18g | 3D printed case |
| RAK WisBlock | 60 x 30 mm | ~15g | IP65 enclosure (commercial) |

### Custom Enclosure Design Guide

For users 3D printing enclosures:
- **Material**: PETG or ABS (UV resistance).
- **Thickness**: 1.5-2mm walls minimum.
- **Antenna clearance**: Keep metal components away from antenna.
- **Waterproofing**: Use cable glands for external antennas.

---

## 10. Security — Secure Element (Optional)

Some reference devices (e.g., T-Beam Supreme) include an ATECC608B or similar secure element.

**Purpose**:
- Hardware-backed private key storage.
- Secure boot attestation.
- Hardware random number generation.

**BGP-X Integration**:
If a secure element is present, the Client Node firmware SHOULD use it for storing the node's identity key. If absent, keys are stored in encrypted flash using ESP32-S3 flash encryption.

---

## 11. PCB Design (Custom Implementation)

If a custom PCB is designed for the Client Node:

**Layer stackup**: 2-layer minimum, 4-layer preferred for RF performance.

**Key Design Considerations**:
- ESP32-S3 requires proper decoupling capacitors.
- LoRa radio (SX1262) RF trace must be 50Ω impedance matched.
- Antenna connector placement must allow clear radiation pattern.
- USB-C connector must follow USB 2.0 specification.

**Open Hardware Files**:
A reference custom PCB design (KiCad) will be provided for users wishing to integrate BGP-X Client Node functionality into a product.

---

## 12. Software Platform

### Firmware

**Primary Target**: ESP32-S3 (ESP-IDF 5.x or Arduino framework).

**Architecture**:
- **HAL**: Hardware Abstraction Layer for GPIO, SPI (LoRa), I2C, UART.
- **Crypto Layer**: ESP-32 hardware accelerated AES/SHA or software ChaCha20 (for BGP-X compatibility).
- **Protocol Layer**: Minimal BGP-X client implementation (decrypt own layer, session handshake).
- **Transport Layer**: Switch between WiFi/BLE/LoRa.

**Build System**: PlatformIO for ease of cross-compilation.

### Configuration

Configuration via JSON file stored in SPIFFS/LittleFS or hard-coded at compile time.

```json
{
  "device_name": "bgpx-client-01",
  "transport": "lora",
  "lora": {
    "frequency": 868.0,
    "spreading_factor": 9,
    "bandwidth": 125,
    "tx_power": 22
  },
  "mesh_island": "lima-district-1",
  "gateway_node_id": "abc123..." // optional fixed gateway
}
```

### Power Management Logic

The firmware implements a power management state machine:

1. **Init**: Wake, init radios.
2. **Connect**: Find mesh entry node (beacon or WiFi scan).
3. **Session**: Establish session (minimal handshake).
4. **Transmit**: Send queued data.
5. **Receive**: Listen for response (windowed timeout).
6. **Sleep**: Enter deep sleep for configured interval.

---

## 13. Comparison: Client Node vs Node v1

| Feature | BGP-X Client Node (Tier 2) | BGP-X Node v1 (Tier 1) |
|---|---|---|
| Target | Endpoint / User | Infrastructure / Relay |
| Routing | No (Client only) | Yes (Full daemon) |
| DHT Storage | No | Yes |
| CPU | ESP32-S3 / nRF52840 (MCU) | MT7981B (Linux SoC) |
| RAM | 512 KB + PSRAM | 256-512 MB |
| OS | Bare metal / RTOS | Linux / OpenWrt |
| Power | Battery native | Solar + Battery / PoE |
| Cost | $25-$40 | $80-$120 |
| Enclosure | Development board / 3D printed | IP67 Commercial |
| Firmware | Subset (Client) | Full (bgpx-node) |
| Role in Mesh | Leaf / Endpoint | Relay / Bridge / Gateway |

---

## 14. Bill of Materials — Approximate Cost (Existing Hardware)

| Device | Price (USD) | Notes |
|---|---|---|
| LILYGO T3S3 | $20-$30 | Primary target; SX1262 + ESP32-S3 |
| Heltec V3 | $22-$28 | Alternative; SX1262 + ESP32-S3 |
| LILYGO T-Beam Supreme | $40-$50 | GPS + Battery included |
| Antenna (LoRa) | $5-$10 | 3-5 dBi fiberglass recommended |
| Battery (18650) | $5-$10 | Optional (if not included) |
| 3D Printed Case | $5-$15 | DIY or printed locally |
| **Total (T3S3 + Antenna + Case)** | **$30-$55** | Well under $40 bare board |

---

## 15. Development & Debugging

**Flashing**: USB-C connection. `esptool` or PlatformIO.

**Debug Output**: UART via USB-C (virtual COM port).

**OTA Updates**: Supported on ESP32-S3. New firmware can be pushed over WiFi or BLE.

---

## 16. Regulatory Notes

All supported hardware operates in ISM bands (LoRa, WiFi, BLE). The user is responsible for:
- Setting correct frequency for region (868 MHz EU, 915 MHz US, etc.)
- Respecting duty cycle limits (EU 868 MHz: 1%).
- Using certified antennas.

BGP-X firmware includes duty cycle enforcement to prevent regulatory violations.
