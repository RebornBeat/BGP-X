# BGP-X Meshtastic Adaptation Layer

**Version**: 0.1.0-draft

---

## 1. Overview

Meshtastic is a popular open-source LoRa mesh network project with a large device ecosystem. BGP-X leverages Meshtastic-compatible hardware as **LoRa radio modems**, enabling domain bridge operation on BGP-X routers and gateways without dedicated LoRa radio hardware.

**Important distinction**: BGP-X does not run Meshtastic software or protocol. The Meshtastic hardware (ESP32 or nRF52840 + LoRa radio) is used purely as a LoRa radio modem. The BGP-X adaptation firmware replaces the Meshtastic routing protocol entirely, exposing the LoRa radio directly to the BGP-X daemon via the standard BGP-X Modem Protocol.

This document specifies how Meshtastic hardware acts as a **BGP-X Adapter (Tier 3)** device. For the official BGP-X Adapter hardware product, see `adapter_spec.md`.

---

## 2. The Role of Meshtastic Hardware in BGP-X

### Why Not Run Meshtastic Protocol?

The Meshtastic protocol is designed for text messaging, not for routing encrypted IP traffic. It has:

- **No onion routing or any privacy routing**
- **All messages visible to all mesh relays in plaintext**
- **Flooding-based routing** (not path-based)
- **No IP transport capability**
- **No DHT participation**

BGP-X requires byte-level LoRa radio access to implement its own packet format, reliability layer, fragmentation, and session management.

### What the Adaptation Does

1. Meshtastic device flashed with BGP-X modem firmware (ESP32 or nRF52840 variant)
2. Connected to BGP-X router via USB
3. BGP-X router communicates with the device via BGP-X Modem Protocol (binary serial interface)
4. BGP-X daemon treats the USB serial port as a LoRa radio transport
5. All BGP-X routing, encryption, and protocol logic runs on the router — the ESP32/nRF52840 is just a radio

---

## 3. Compatible Hardware

Meshtastic devices are categorized by their suitability as BGP-X radio modems.

### Tier 1 (Recommended — Fully Tested)

These devices have sufficient resources, modern LoRa radios (SX1262 preferred), and stable USB connections.

| Device | MCU | LoRa Chip | Frequency | USB | Battery | BGP-X Support |
|---|---|---|---|---|---|---|
| LILYGO T-Beam S3 Supreme | ESP32-S3 | SX1262 | 868/915 MHz | Yes (native USB-C) | 18650 | **Yes (Primary)** |
| LILYGO T3S3 | ESP32-S3 | SX1262 | 868/915 MHz | Yes (native USB-C) | Optional | **Yes** |
| Heltec WiFi LoRa 32 V3 | ESP32-S3 | SX1262 | 868/915 MHz | Yes (USB-C) | Optional | **Yes** |
| LILYGO T-Echo | nRF52840 | SX1262 | 868/915 MHz | Yes (native USB) | LiPo | **Yes** |
| RAK WisBlock 4631 | nRF52840 | SX1262 | 868/915 MHz | Yes | Yes | **Yes** |

### Tier 2 (Compatible with Caveats)

These devices work but use the older SX1276 LoRa radio, which has lower sensitivity and range than SX1262.

| Device | MCU | LoRa | USB | Notes |
|---|---|---|---|---|
| LILYGO T-Beam (Classic) | ESP32 | SX1276 | USB-C | SX1276 has ~11 dB lower sensitivity |
| Heltec WiFi LoRa 32 V2 | ESP32 | SX1276 | Micro-USB | As above |
| TTGO LoRa32 V2.1 | ESP32 | SX1276 | Micro-USB | As above |

### Not Compatible

| Device | Issue |
|---|---|---|
| RAK811 | STM32L151 MCU; insufficient RAM for BGP-X modem firmware |
| Dragino LA66 | LoRaWAN only; cannot use raw LoRa mode needed by BGP-X |
| Any LoRaWAN-only device | BGP-X requires raw LoRa mode, not LoRaWAN |

**Recommendation**: For new deployments, use **SX1262-based devices** (Tier 1). The improved sensitivity (-148 dBm vs -137 dBm at SF12) provides significantly better range.

---

## 4. BGP-X Modem Firmware

The BGP-X modem firmware is flashed to the ESP32/nRF52840 device. It replaces the Meshtastic routing stack with:

### What the Firmware Does

- Sets LoRa radio to raw mode (no LoRaWAN, no Meshtastic protocol)
- Exposes the **BGP-X Modem Protocol** over USB CDC-ACM (virtual serial port)
- Receives TX commands from BGP-X daemon
- Transmits packets via LoRa radio
- Receives LoRa packets and forwards them to the host
- Handles LoRa duty cycle enforcement (EU 1% / configurable elsewhere)
- Reports RSSI, SNR, and radio statistics

### Firmware Flash Procedure

**For ESP32-S3 devices (T-Beam Supreme, T3S3, Heltec V3):**

```bash
esptool.py \
    --chip esp32s3 \
    --port /dev/ttyUSB0 \
    --baud 921600 \
    write_flash 0x0 bgpx-modem-firmware-v0.1.0-esp32s3.bin
```

**For nRF52840 devices (T-Echo, RAK4631):**

```bash
adafruit-nrfutil dfu serial \
    --package bgpx-modem-firmware-v0.1.0-nrf52840.zip \
    -p /dev/ttyACM0 \
    -b 115200
```

**Verify firmware version (via serial):**
```bash
echo "VER" > /dev/ttyACM0
# Expected response: BGP-X Modem v0.1.0
```

---

## 5. BGP-X Modem Protocol

The Meshtastic adapter communicates with the BGP-X daemon using the **BGP-X Modem Protocol** (binary serial interface). This is the same protocol used by the official BGP-X Adapter product.

**Physical Layer**: USB CDC-ACM (virtual serial port). No baud rate (native USB speed).

**Framing**: Length-prefixed binary frames.

**Command Format (Host → Modem):**

```
[0x42 0x47] [Length: uint16 BE] [Command: uint8] [Payload: bytes]
```

**Response Format (Modem → Host):**

```
[0x42 0x47] [Length: uint16 BE] [Type: uint8] [Payload: bytes]
```

### Commands (Host → Modem)

| Code | Command | Payload | Description |
|---|---|---|---|
| 0x01 | `TRANSMIT` | BGP-X packet bytes | Transmit payload on LoRa immediately. |
| 0x02 | `SET_FREQUENCY` | `frequency_hz: uint32` | Set LoRa frequency (e.g., 868000000 for 868 MHz). |
| 0x03 | `SET_SF` | `sf: uint8` (7-12) | Set Spreading Factor. |
| 0x04 | `SET_BW` | `bw_khz: uint16` (125, 250, 500) | Set Bandwidth. |
| 0x05 | `SET_POWER` | `power_dbm: int8` (-9 to +22) | Set TX power. |
| 0x06 | `GET_STATUS` | (none) | Request radio status. |
| 0x07 | `RESET` | (none) | Soft reset the modem. |

### Responses (Modem → Host)

| Code | Type | Payload | Description |
|---|---|---|---|
| 0x81 | `RECEIVED` | `RSSI: int16`, `SNR: int8`, `packet_bytes` | LoRa packet received. |
| 0x82 | `TRANSMIT_ACK` | (none) | Transmission queued or started. |
| 0x83 | `STATUS` | `freq`, `sf`, `bw`, `pwr`, `duty_cycle_pct` | Current radio configuration. |
| 0x84 | `ERROR` | `error_code: uint8`, `message` | Error occurred. |

---

## 6. Hardware Connection to BGP-X Router

### USB Connection (Recommended)

Connect the Meshtastic device to a BGP-X Router v1 or Node v1 via USB. The device appears as `/dev/ttyUSB0` or `/dev/ttyACM0`.

```toml
# In BGP-X config.toml on the router:

[[routing_domains]]
domain_type = "lora-regional"
domain_id = "0x00000004cc330044"
region_id = "eu-868"
transport = "serial"
serial_device = "/dev/ttyUSB0"
baud_rate = 921600           # USB native speed
spreading_factor = 9
bandwidth_khz = 125
tx_power_dbm = 14            # EU limit: 14 dBm conducted
duty_cycle_percent = 1.0    # EU regulatory requirement
```

### Power Considerations

Meshtastic devices connected via USB are powered by the router's USB port.

- **Receive mode**: 50-80 mA at 3.3V
- **Transmit mode**: 200-400 mA at 3.3V (depends on TX power)

Most routers' USB ports supply 500-900 mA, sufficient for one device. For multiple devices, use a powered USB hub.

### Solar + Battery Operation

For outdoor deployment with solar power:

1. Solar panel charges battery via charge controller.
2. Raspberry Pi or BGP-X router powered from battery.
3. RAK4631 connected via USB.
4. BGP-X modem firmware manages LoRa radio; BGP-X daemon manages routing.

This forms an off-grid domain bridge node without any ISP connection (mesh island relay mode).

---

## 7. Range and Coverage

### Free-Space Path Loss Estimates (LoRa at 915 MHz, SF9, +14 dBm TX)

| Scenario | Estimated Range |
|---|---|
| Ground level, urban | 1-3 km |
| Ground level, rural open terrain | 5-15 km |
| Rooftop deployment (15m) | 8-25 km |
| Mast deployment (30m) | 15-40 km |

### Antenna Recommendations

- **Standard (included)**: 2 dBi rubber duck. Adequate for <3 km.
- **Fiberglass omni (5-8 dBi)**: Doubles typical range. Recommended for rooftops.
- **Yagi directional (8-12 dBi)**: For point-to-point links.

**Antenna polarization**: LoRa antennas should be vertically polarized. Mismatched polarization causes up to 20 dB loss.

---

## 8. BGP-X Configuration for Meshtastic Adapter

### Full Domain Bridge Configuration (Clearnet + LoRa Mesh)

This configuration turns a BGP-X Router v1 with a USB Meshtastic adapter into a full domain bridge.

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [
  { protocol = "udp", address = "0.0.0.0", port = 7474 }
]

[[routing_domains]]
domain_type = "lora-regional"
domain_id = "0x00000004cc330044"
region_id = "na-915"
transport = "serial"
serial_device = "/dev/ttyACM0"
spreading_factor = 9
bandwidth_khz = 125
tx_power_dbm = 20           # FCC allows up to 30 dBm
duty_cycle_percent = 100.0  # No EU duty cycle in NA

[[bridges]]
from_domain = "clearnet"
to_domain = "lora-regional:na-915"
bridge_latency_ms = 300
available = true
```

With this configuration:
- The router bridges clearnet (UDP/7474) to LoRa mesh.
- Clearnet clients can reach LoRa mesh island services.
- LoRa mesh clients can reach clearnet via this gateway.

---

## 9. Operating the Adapter

### Verify the Adapter is Working

```bash
# Check serial port
ls -la /dev/ttyUSB0 /dev/ttyACM0

# Check BGP-X sees LoRa transport
bgpx-cli domains bridges --health

# Check LoRa transport statistics
bgpx-cli node stats | grep lora

# Test LoRa TX/RX with nearby node
bgpx-cli node ping <nearby_lora_node_id> --domain lora-regional:na-915
```

### Duty Cycle Monitoring (EU)

```bash
# Check remaining duty cycle budget
bgpx-cli node stats | grep lora_duty_cycle_remaining_pct

# View LoRa-domain congestion events
bgpx-cli events --domain lora-regional:eu-868 --last 24h
```

### Antenna Inspection

Physical antenna inspection should be part of monthly maintenance. A disconnected or damaged antenna causes all transmissions to fail silently (TX command returns OK but no radio signal is transmitted).

---

## 10. Power and Battery

| Device | Internal Battery | Expected Runtime (BGP-X modem) |
|---|---|---|
| T-Beam Supreme | 18650 (3400 mAh) | ~15-20 hours |
| T-Echo | LiPo 700 mAh | ~6-8 hours |
| Heltec v3 | LiPo 1800 mAh (optional) | ~10-15 hours |

For extended deployments: external LiPo via JST connector or USB power bank.

---

## 11. Regulatory and Frequency Selection

Configure the adapter for the correct LoRa frequency band:

| Region | Frequency | Notes |
|---|---|---|
| EU | 868 MHz | 1% duty cycle; ETSI EN 300 220 |
| US/CA | 915 MHz | FCC Part 15; no duty cycle |
| AU/NZ | 915-928 MHz | AS/NZS 4268 |
| India | 865-867 MHz | WPC type approval |
| Japan | 920-925 MHz | ARIB STD-T108 |
| Brazil | 915-928 MHz | Anatel |

**Operators are responsible for ensuring their LoRa configuration complies with local radio regulations.** Transmitting on incorrect frequencies or at excessive power is illegal in most jurisdictions.

---

## 12. Comparison: Meshtastic Adapter vs. BGP-X Adapter Product

| Feature | Meshtastic Adapter | BGP-X Adapter v1 |
|---|---|---|
| **Hardware** | LILYGO T3S3, T-Beam, etc. | Custom BGP-X designed dongle |
| **Target User** | Community contributor | End user |
| **Form Factor** | Development board | Compact USB dongle |
| **Enclosure** | Optional (3D printed) | Commercial enclosure |
| **Antenna** | SMA connector (external) | SMA or PCB antenna |
| **Firmware** | BGP-X Modem Firmware | BGP-X Modem Firmware |
| **Protocol** | BGP-X Modem Protocol | BGP-X Modem Protocol |
| **Cost** | $25-40 | $15-25 (target) |
| **Integration** | Custom USB cable | Integrated USB-C or USB-A |

Both implement the same Modem Protocol and work identically with the BGP-X daemon. The Meshtastic adapter is for users who already own compatible hardware or prefer the flexibility of development boards. The BGP-X Adapter v1 is the commercial product for users who want a plug-and-play experience.
