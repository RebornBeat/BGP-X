# BGP-X Adapter v1 Hardware Specification

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2

---

## 1. Overview

The BGP-X Adapter v1 (also referred to as the **BGP-X Dongle**) is a **USB LoRa radio modem**. It is not a standalone routing device. It is a peripheral that connects a host computer (laptop, desktop, server, or OpenWrt router) to a BGP-X mesh network via LoRa radio.

The host computer runs the full `bgpx-node` daemon, which handles all routing logic, cryptographic operations (onion encryption, DHT participation, session management), and protocol state. The Adapter v1 handles only the physical and link layers: it modulates packets onto the LoRa radio and demodulates received packets back to the host.

### What the Adapter v1 IS

- A **USB radio peripheral**: provides LoRa RF capability to devices without integrated LoRa hardware.
- A **dumb modem**: contains no BGP-X routing logic. It speaks a simple binary "Modem Protocol" for TX, RX, and radio configuration.
- A **bridge for existing hardware**: turns any Linux/macOS/Windows computer or compatible OpenWrt router into a BGP-X node.
- **USB bus-powered**: requires no external power supply.
- Low-cost: target manufacturing cost under $25.

### What the Adapter v1 is NOT

- NOT a standalone BGP-X node (cannot route traffic independently).
- NOT a WiFi or BLE device (LoRa only — other transports are the host's responsibility).
- NOT a battery-powered device (USB bus power only).
- NOT a router (no IP stack, no NAT, no DHCP — all networking is handled by the host).

---

## 2. Design Philosophy — Minimal Intelligence, Maximum Compatibility

The BGP-X Adapter v1 follows the "smart host, dumb peripheral" model. This design choice is deliberate:

1.  **Maximum Compatibility**: Any system with a USB port and a driver for the modem protocol can participate in a BGP-X mesh. Users do not need to purchase specialized hardware to test BGP-X on their existing laptops or servers.
2.  **Firmware Simplicity**: The firmware on the adapter is simple and stable. It deals only with LoRa SPI communication and USB serial data transfer. It does not need to be updated when the BGP-X protocol changes, provided the modem protocol interface remains stable.
3.  **Cost Reduction**: Offloading all crypto and logic to the host reduces the adapter's MCU requirements, enabling a lower BOM cost.

**Capability requirements drive compliance — not specific part numbers.**

---

## 3. Capability Requirements (Hardware-Agnostic)

Any implementation claiming BGP-X Adapter v1 compliance MUST meet:

| Capability | Minimum | Reference Implementation |
|---|---|---|
| MCU | ARM Cortex-M4 or Xtensa LX7, ≥240 MHz | ESP32-S3 or nRF52840 |
| RAM | 256 KB minimum | 512 KB SRAM |
| Flash | 1 MB minimum | 4–16 MB |
| LoRa | SPI-connected LoRa transceiver (SX1261/SX1262 or equivalent) | SX1262 |
| USB | USB 2.0 Full Speed (12 Mbps) or higher | USB 2.0 FS (native in ESP32-S3) |
| Interface | USB CDC-ACM (virtual serial port) or USB HID | USB CDC-ACM (standard serial driver) |
| Antenna | External SMA connector or integrated PCB antenna | SMA female |
| Power | USB bus power (max 500 mA at 5V) | 5V @ 200 mA peak |
| Host Driver | Linux, macOS, Windows, OpenWrt | Cross-platform `bgpx-adapter-driver` |

**Note on USB Class**: CDC-ACM (Communications Device Class - Abstract Control Model) is the preferred interface. It appears as a standard serial port (`/dev/ttyACMx` on Linux, `COMx` on Windows) without requiring custom kernel drivers. This maximizes compatibility across operating systems.

---

## 4. Reference Hardware — Compute

The reference implementation uses the ESP32-S3, which provides native USB OTG, ample flash, and sufficient processing power for the LoRa SPI driver.

### Primary Reference Platform

| Component | Specification |
|---|---|
| MCU | Espressif ESP32-S3-WROOM-1 |
| CPU | Xtensa LX7 dual-core, 240 MHz |
| RAM | 512 KB SRAM + 8 MB PSRAM |
| Flash | 16 MB QSPI |
| USB | USB-C connector, native USB OTG (CDC-ACM profile) |
| LoRa | SX1262 (SPI interface) |
| Power | 5V via USB bus; 3.3V LDO regulation onboard |

### Alternative Platform (Nordic Semiconductor)

| Component | Specification |
|---|---|
| MCU | nRF52840 |
| CPU | ARM Cortex-M4F, 64 MHz |
| RAM | 256 KB |
| Flash | 1 MB |
| USB | USB-C, native USB device |
| LoRa | SX1262 (SPI) |

Both platforms support the required Modem Protocol firmware. The ESP32-S3 is the preferred reference due to its higher clock speed (more headroom for SPI throughput) and lower cost.

---

## 5. Reference Hardware — Radio

### LoRa Radio

| Parameter | Reference | Equivalent |
|---|---|---|
| Chipset | SX1262 | SX1261, SX1268, LLCC68 |
| Interface | SPI | SPI |
| Max power | +22 dBm | +14 dBm minimum |
| Sensitivity | -148 dBm at SF12 | -140 dBm minimum |
| Frequency | Region SKU | 433/868/915 MHz variants |
| Antenna | SMA female (external) | PCB antenna option for ultra-compact form |

**Radio Control**: The adapter's firmware controls the radio directly via SPI. The host sends high-level commands (e.g., "transmit this data," "set frequency to 915 MHz"). The firmware manages the SX1262 registers, FIFO, DIO interrupts, and sleep states.

**Duty Cycle Enforcement**: The host's `bgpx-node` daemon is responsible for regulatory duty cycle compliance. The adapter executes transmission commands immediately when received. However, the firmware MAY implement a local safety limiter (e.g., refusing commands that would exceed 10% duty cycle in the last minute) to protect against misconfigured host software.

---

## 6. Power System

The Adapter v1 is **USB bus-powered**. It has no battery, no solar input, and no external DC jack.

### Power Budget

| Component | Current Draw (Typical) |
|---|---|
| MCU (ESP32-S3 active) | 80 mA @ 3.3V (~0.26W) |
| LoRa SX1262 (RX) | 10 mA @ 3.3V |
| LoRa SX1262 (TX +20 dBm) | 120 mA @ 3.3V (~0.4W) |
| USB Interface | 5 mA @ 5V |
| **Total (Peak TX)** | ~205 mA @ 3.3V + conversion losses ≈ **150 mA @ 5V USB** |
| **Total (Idle/RX)** | ~95 mA @ 3.3V ≈ **70 mA @ 5V USB** |

This is well within the USB 2.0 specification (500 mA max). No external power is required.

**Low-Power Mode**: When the host is idle (no BGP-X traffic for a configurable period), the daemon can send a `CMD_SLEEP` command to the adapter. The firmware puts the MCU in light sleep and the LoRa radio in standby/sleep mode. Wake-on-USB or `CMD_WAKE` restores operation.

---

## 7. Physical Form Factor

### Reference Enclosure

| Attribute | Specification |
|---|---|---|
| Form Factor | USB Dongle (direct plug) or short cable |
| Dimensions | 50 × 20 × 10 mm (reference) |
| Material | ABS plastic (no IP rating required for typical desktop use) |
| Connector | USB-C (preferred) or USB-A |
| Antenna | SMA female on the side or top; optional magnet-mount external antenna kit |

**Portable Option**: A variant with a short 10cm USB-C cable and a clip allows the dongle to be positioned away from the laptop for better antenna placement (laptops are often placed on metal surfaces that block RF).

### Product Variants

| Variant | Description | Antenna |
|---|---|---|
| BGP-X Adapter v1 Desktop | Direct plug USB-C | SMA connector |
| BGP-X Adapter v1 Compact | Ultra-small, PCB antenna | Integrated PCB antenna |
| BGP-X Adapter v1 Outdoor Kit | Weatherproof case + magnetic mount antenna | External high-gain antenna |

---

## 8. Software Platform — Modem Firmware

The Adapter v1 runs the **BGP-X Modem Firmware** (specified in `/firmware/modem_firmware.md`). This firmware implements the **BGP-X Modem Protocol** over USB CDC-ACM.

### Modem Protocol (Summary)

The host communicates with the adapter via a simple binary protocol over the virtual serial port.

**Command Structure (Host → Adapter)**:

```
[Start Byte (0x02)] [Command ID] [Length (2 bytes)] [Payload...] [Checksum (CRC16)]
```

**Response Structure (Adapter → Host)**:

```
[Start Byte (0x02)] [Response ID] [Status] [Length (2 bytes)] [Payload...] [Checksum]
```

**Core Commands**:

| Command ID | Name | Payload | Description |
|---|---|---|---|
| 0x01 | `CMD_TX` | `[Data]` | Transmit payload on LoRa immediately. |
| 0x02 | `CMD_RX_MODE` | `[Timeout]` | Enter receive mode for `Timeout` ms. |
| 0x03 | `CMD_CONFIG` | `[Freq, SF, BW, CR, TX_Pwr]` | Configure LoRa radio parameters. |
| 0x04 | `CMD_SLEEP` | None | Enter low-power sleep mode. |
| 0x05 | `CMD_WAKE` | None | Wake from sleep. |
| 0x06 | `CMD_GET_STATUS` | None | Return RSSI, SNR, firmware version, errors. |

**Asynchronous Events (Adapter → Host)**:

| Event ID | Name | Payload | Description |
|---|---|---|---|
| 0x81 | `EVT_RX` | `[Data, RSSI, SNR, Timestamp]` | LoRa packet received. |
| 0x82 | `EVT_TX_DONE` | None | Transmission complete. |
| 0x83 | `EVT_ERROR` | `[Error Code]` | Radio or firmware error. |

The host's `bgpx-node` daemon implements a `bgpx-transport-lora` driver that speaks this protocol. To the daemon, the adapter appears as just another LoRa transport interface.

---

## 9. Host Integration

The Adapter v1 is useless without host software. The BGP-X project provides:

1.  **Modem Driver**: A userspace library (`libbgpx-adapter`) linked into `bgpx-node`.
2.  **Daemon Support**: `bgpx-node` detects USB-serial devices and can be configured to use an adapter as a LoRa transport.

**Configuration Example (`config.toml`)**:

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "my-island-id"

# Use the USB adapter for LoRa transport
transports = ["lora"]

[transports.lora]
type = "adapter"
device = "/dev/ttyACM0"   # USB CDC-ACM device
baud_rate = 921600       # Max for USB-serial
frequency_mhz = 915.0
spreading_factor = 9
bandwidth_khz = 125
tx_power_dbm = 20
```

When this configuration is active, `bgpx-node` sends all LoRa traffic through the USB adapter. The daemon performs all BGP-X crypto and DHT logic. The adapter is a transparent radio interface.

---

## 10. Security

The Adapter v1 performs **no cryptographic operations**. It has no access to BGP-X session keys, NodeIDs, or private keys. It is a "bump-in-the-wire" radio modem.

**Attack Surface**:
- Physical access to the adapter allows an adversary to replace the firmware. Mitigation: firmware read-out protection (eFuse) should be set at the factory.
- USB traffic interception (if host is compromised). This is out of BGP-X's threat model (compromised host = total loss).

**Firmware Integrity**:
- The modem firmware is unsigned (to reduce complexity and cost). The host must trust the firmware.
- For high-security applications, the host can verify the adapter's firmware version via `CMD_GET_STATUS` and compare it against a known-good hash (requires the firmware build to publish a hash).

---

## 11. Comparison to Other Hardware Tiers

| Feature | Adapter v1 (Tier 3) | Client Node v1 (Tier 2) | Node v1 (Tier 1) |
|---|---|---|---|
| **Role** | LoRa Radio Modem | Endpoint (Client) | Relay / Bridge (Infrastructure) |
| **Intelligence** | None (Modem Protocol only) | Client-side BGP-X subset | Full BGP-X Daemon |
| **Hardware** | USB dongle | Standalone, battery-powered | Standalone, solar/wall-powered |
| **Host Required** | Yes (laptop/PC/router) | No (self-contained) | No (self-contained) |
| **Crypto** | None (host does it) | Session-level decryption only | Full onion routing |
| **DHT** | None | Query only (no storage) | Full participation + storage |
| **Power** | USB bus power | Internal LiPo battery | Solar / PoE / DC |
| **Cost** | <$25 | $25–40 | $80–120 |
| **Use Case** | Developer testing; instant LoRa capability for existing computers; turning a router into a bridge node | Portable client; IoT sensor; field researcher | Permanent community mesh infrastructure |

---

## 12. Bill of Materials — Approximate Component Cost

| Component | Approximate Cost |
|---|---|
| MCU Module (ESP32-S3) | $3–6 |
| LoRa Module (SX1262) | $5–8 |
| USB Connector (USB-C) | $0.50 |
| LDO Regulator (3.3V) | $0.20 |
| PCB (2-layer) | $0.50–1.00 |
| SMA Connector | $0.50 |
| Passives | $0.50 |
| Enclosure (simple plastic) | $1.00–2.00 |
| **Total (Approx.)** | **$11–19** |

This BOM target allows a retail price under $25 while maintaining margin.

---

## 13. Firmware Update Mechanism

The Adapter v1 supports firmware updates via USB.

1.  **Bootloader Mode**: The adapter ships with a USB DFU (Device Firmware Upgrade) compatible bootloader.
2.  **Entering DFU**: A hardware button on the dongle (or a specific USB command sequence) forces entry into DFU mode.
3.  **Updating**: The host uses standard tools (e.g., `dfu-util` for ESP32-S3) to flash new modem firmware.

The BGP-X project will provide signed firmware images for the reference hardware. Advanced users may compile and flash their own.

---

## 14. USB Driver Support

**Linux**: Kernel support for CDC-ACM is built-in. The device appears as `/dev/ttyACMx`. No proprietary driver required.
**macOS**: Kernel support for CDC-ACM is built-in. The device appears as `/dev/cu.usbmodem*`.
**Windows**: Standard USB serial drivers (USB SER) are typically used. A signed `.inf` file will be provided by the BGP-X project for seamless installation.
**OpenWrt**: Standard `kmod-usb-acm` package provides support.

The `bgpx-node` daemon communicates with the device using standard POSIX serial I/O (`open()`, `read()`, `write()`, `tcsetattr()`).

---

## 15. Environmental and Regulatory

The Adapter v1 is a desktop device. It is not designed for harsh outdoor environments. For outdoor LoRa deployment, use the **BGP-X Node v1**.

- **Operating Temperature**: 0°C to +50°C (standard desktop range).
- **IP Rating**: None (IP20).
- **Regulatory**: The device is a radio transmitter. It must be certified (FCC, CE, ISED) for the target region before sale. The modular approach (certified LoRa module + certified USB device) simplifies this process compared to a fully custom radio design.
