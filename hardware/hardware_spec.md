# BGP-X Hardware Specification

**Version**: 0.1.0-draft

---

## 1. Unified Hardware Platform

BGP-X defines a unified hardware platform with carrier board + enclosure variants. Single PCB design reduces manufacturing complexity.

---

## 2. BGP-X Node Core (Compute Module)

| Component | Specification |
|---|---|
| SoC | MediaTek MT7981B (ARM Cortex-A53 dual-core 1.3GHz) |
| RAM | 512MB DDR4 |
| Flash | 128MB SPI NAND + 8GB eMMC |
| Security | TPM 2.0 (Infineon SLB9670) |
| Crypto acceleration | Hardware AES-128/256, SHA-256/384/512 |
| USB | USB 3.0 host, USB 2.0 device |
| Form factor | M.2 2242 compatible compute module |
| Power | 3.3V, 3W typical |

---

## 3. Radios

### WiFi Module (Integrated)
```
Chip:         MediaTek MT7915 (WiFi 6, 802.11ax)
Bands:        2.4GHz + 5GHz simultaneous
802.11s:      Full mesh profile support
Antennas:     2 × U.FL connectors
Max power:    20 dBm
```

### LoRa Module (Integrated on gateway variant, USB on others)
```
Chip:         Semtech SX1262
Frequency:    Configurable: 433, 868, 915 MHz
Output power: Up to +22 dBm
Sensitivity:  -148 dBm (SF12)
Interface:    SPI
```

### Optional BLE Module
```
Chip:         Nordic nRF52840
Protocol:     Bluetooth 5.0 LE
Max power:    +8 dBm
Interface:    UART or SPI
```

---

## 4. Gateway Hardware (MT7988A Variant)

For domain bridge nodes and gateway deployments requiring higher throughput.

| Component | Specification |
|---|---|
| SoC | MediaTek MT7988A (ARM Cortex-A73 quad-core 1.8GHz) |
| RAM | 1GB DDR4 |
| Flash | 256MB NAND + 8GB eMMC |
| WAN | 2 × 2.5GbE RJ45 |
| LAN | 4 × 1GbE RJ45 |
| SFP+ | 1 × 10GbE SFP+ |
| mPCIe | 1 slot (4G/5G modem option) |
| USB | 2 × USB 3.0 |
| Power | 12V DC, 15W typical |

---

## 5. Amplifier Hardware

Pure range extension. No routing intelligence.

| Component | Specification |
|---|---|
| MCU | STMicroelectronics STM32H7 |
| Radio | SX1262 LoRa |
| Battery | 10,000 mAh Li-Ion (optional solar) |
| Power | <800mW active |
| Enclosure | IP67 weatherproof |

---

## 6. Enclosure Variants

| Variant | IP Rating | Mounting | Use Case |
|---|---|---|---|
| Indoor Desktop | IP20 | None | Home, office |
| Outdoor Mast | IP67 | Mast/pole | Outdoor deployment |
| Maritime | IP68 | Bulkhead | Marine/waterfront |
| Industrial | IP67/ATEX | DIN rail | Industrial/hazardous |
| Mobile/Vehicle | IP65 | Vehicle mount | Mobile mesh |

---

## 7. Compatible Third-Party Hardware

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

---

## 8. Domain Bridge Hardware Requirements

Domain bridge nodes require:
- Both a clearnet network interface (WAN)
- And at least one mesh radio interface (WiFi mesh or LoRa)

The GL.iNet GL-MT3000 with external LoRa USB module is the recommended minimum-cost domain bridge node. The BGP-X gateway hardware (MT7988A) is recommended for production deployments.
