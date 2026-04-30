# BGP-X Firmware Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X firmware is the operating system and runtime environment for BGP-X hardware. The core BGP-X software (bgpx-node daemon, bgpx-cli, bgpx-sdk) is hardware-agnostic. Firmware provides the hardware-specific layer: device drivers, kernel configuration, init system, and first-boot wizard.

**All firmware targets run the same bgpx-node daemon binary** (cross-compiled for the appropriate CPU architecture). Protocol behavior, configuration format, and capabilities are identical across all targets. Hardware differences affect only drivers and power management — not protocol, not routing, not cryptography.

---

## 2. Firmware Adaptability Principle

BGP-X firmware is designed for adaptability. The current hardware targets represent reference implementations. As hardware evolves:

- New SoC targets: add a device tree file and update HAL — daemon unchanged
- New radio modules: add a kernel driver — daemon transport API unchanged
- Custom BGP-X ASIC: implement the hardware transport abstraction — daemon unchanged
- Alternative security elements: implement the TPM 2.0 TCTI interface — daemon unchanged

The bgpx-node daemon interfaces with hardware ONLY through:
1. Standard Linux network interfaces (TUN, Ethernet, WiFi)
2. Linux kernel serial device (`/dev/ttyUSBx`, `/dev/ttySx`)
3. TPM character device (`/dev/tpm0` via tpm2-tss)
4. Linux kernel power management (`/sys/class/power_supply/`)

Nothing else. Any hardware that provides these standard Linux interfaces can run bgpx-node.

---

## 3. Firmware Variants

### 3.1 bgpx-router-firmware

**Target hardware**: BGP-X Router v1 (reference: ARM dual-core Cortex-A53 SoC, see `router_spec.md`)

**Purpose**: All-in-one end-user router firmware. Supports all capabilities simultaneously.

**Base OS**: OpenWrt 23.05+ (custom BGP-X build target)

**Kernel**: Linux 6.1 LTS

**Capabilities enabled**:
- WAN routing, NAT, DHCP server
- BGP-X overlay (clearnet domain)
- WiFi 802.11s mesh (via mac80211 + wpad-mesh-openssl)
- LoRa mesh transport (via bgpx-sx1262 kernel module)
- Bluetooth BLE (optional; disabled by default; enable via `bgpx-luci`)
- Domain bridge (clearnet ↔ mesh, any combination)
- Mesh island gateway
- Satellite WAN auto-detection (USB modem vendor ID)
- TPM 2.0 key storage
- Full LuCI web UI (bgpx-luci plugin)
- First-boot wizard

**Software stack**:
```
OpenWrt 23.05 (kernel 6.1 LTS)
├── bgpx-node daemon (compiled for target arch)
├── bgpx-cli (control utility)
├── bgpx-luci (LuCI web UI plugin)
├── bgpx-sx1262 (LoRa SPI driver kernel module)
├── bgpx-tpm (TPM 2.0 integration: tpm2-tss + tpm2-tools)
├── bgpx-firstboot (first-boot wizard)
├── wpad-mesh-openssl (WiFi 802.11s)
├── kmod-mt7915e (WiFi driver for MT7915 reference; substitute for other chipsets)
├── kmod-usb-net (USB satellite modem support)
├── kmod-bluetooth (BLE support; optional)
└── luci (standard LuCI web interface)
```

**Configuration at first boot**:
1. WAN type selection (DHCP/PPPoE/Static/USB satellite)
2. Node role (relay, relay+gateway, domain bridge)
3. Ed25519 key generation (TPM if available; software fallback)
4. DHT bootstrap
5. LoRa region selection (EU-868, NA-915, Asia-433)
6. WiFi mesh island configuration
7. LuCI admin password set

**Build architecture**: OpenWrt external package feed targeting the BGP-X Router v1 board. Build system generates sysupgrade .bin image suitable for flashing to device or OTA update.

---

### 3.2 bgpx-node-firmware

**Target hardware**: BGP-X Node v1 (reference: low-power ARM Cortex-A53 or equivalent, see `node_spec.md`)

**Purpose**: Community contributor node firmware. Multi-radio mesh relay, domain bridge, and range extension node with solar/battery power management.

**Base OS**: OpenWrt 23.05+ (BGP-X Node v1 custom build target)

**Kernel**: Linux 5.15 LTS (stable, wide low-power ARM BSP support) or Linux 6.1 LTS

**Capabilities enabled**:
- BGP-X mesh relay (LoRa + WiFi mesh + BLE, configurable)
- BGP-X domain bridge (with optional WAN Ethernet)
- BGP-X range extension mode
- Community gateway (with WAN + exit policy)
- Solar/battery power management daemon (bgpx-power)
- LoRa duty cycle enforcement
- BLE mesh transport
- Satellite WAN support (USB modem, optional)
- LuCI web UI (accessible via WiFi AP or Ethernet when configured)
- TPM optional (software key fallback is default for cost-sensitive configurations)

**Software stack**:
```
OpenWrt 23.05 (kernel 5.15 or 6.1 LTS)
├── bgpx-node daemon (compiled for target arch, low-power build)
├── bgpx-cli
├── bgpx-luci (minimal profile)
├── bgpx-sx1262 (LoRa SPI driver)
├── bgpx-power (solar/battery power management daemon)
├── bgpx-modem-bridge (serial protocol bridge for LoRa USB modem, if not native SPI)
├── wpad-mesh-openssl (WiFi 802.11s; optional if WiFi-disabled build)
└── kmod-bluetooth (BLE; optional)
```

**Low-power build differences** from router firmware:
- Reduced maximum session count (default 2,000 vs. 10,000)
- Reduced path_id table capacity (5,000 entries vs. unlimited)
- Power-aware scheduling: BGP-X daemon yields to power management events
- Radio duty cycle integrated with bgpx-power battery state
- Swap disabled (same as all BGP-X firmware — key material must not be in swap)
- Optional: no web UI (CLI only, saves ~4 MB RAM)

**bgpx-power daemon**:

The bgpx-power daemon monitors battery state and adjusts BGP-X operation:

| Battery Level | Action |
|---|---|
| 100-50% | Normal operation; all radios at configured TX power |
| 50-30% | Reduce LoRa TX power by 4 dBm |
| 30-15% | Disable WiFi mesh if LoRa mesh available; LoRa TX further reduced |
| 15-5% | Enter range extension mode only; minimal session origination |
| <5% | Emergency: relay-only, minimum power, disable BLE |
| 0% (threshold) | Graceful shutdown: publish NODE_WITHDRAW; wait for withdrawal propagation; power off |

---

### 3.3 bgpx-gateway-firmware

**Target hardware**: BGP-X Gateway v1 (reference: high-throughput ARM quad-core, see `gateway_spec.md`)

**Purpose**: Provider-grade exit node and domain bridge firmware. Optimized for maximum throughput and concurrent sessions.

**Base OS**: OpenWrt 23.05+ (BGP-X Gateway v1 build target)

**Kernel**: Linux 6.1 LTS with network performance patches (NAPI tuning, RSS, interrupt affinity)

**Capabilities enabled**:
- BGP-X clearnet relay (high throughput)
- BGP-X exit node (exit policy engine, ECH, DoH)
- BGP-X domain bridge (clearnet + LoRa/mesh via USB adapters)
- Mesh island gateway
- SFP+ uplink
- Provider monitoring (Prometheus metrics exporter)

**Software stack**:
```
OpenWrt 23.05 (kernel 6.1 LTS)
├── bgpx-node daemon (gateway build: max session 50,000; large buffer pools)
├── bgpx-gateway (exit policy engine extensions)
├── bgpx-cli
├── bgpx-luci (full provider profile)
├── bgpx-exitpolicy (exit policy signing/verification tool)
├── kmod-sfp (SFP+ driver)
├── kmod-usb-net (USB LoRa adapter support)
├── prometheus-node-exporter (optional; provider monitoring)
└── chrony (NTP with strict max offset enforcement)
```

**Provider-optimized configuration differences**:
- max_sessions = 50,000
- SO_RCVBUF = SO_SNDBUF = 64 MB (vs. 16 MB default)
- Worker threads = CPU_CORES (all cores, no OS reserve)
- Sender threads = 8 (vs. 4 default)
- Session idle timeout = 120 seconds (vs. 90 seconds)
- Path_id table shards = 256 (vs. 64 default)
- ECH cache size = 10,000 entries
- DoH resolver threads = 4

---

### 3.4 bgpx-openwrt

**Target hardware**: Any compatible OpenWrt 23.05+ router (see `compatible_hardware.md`)

**Purpose**: Software-only package for third-party compatible routers. No hardware customization — installs as standard OpenWrt packages.

**Package set** (installed via opkg):
```
opkg install bgpx-node bgpx-cli bgpx-luci

# Optional for LoRa domain bridge:
opkg install bgpx-modem-driver    # USB LoRa serial protocol driver

# Optional for WiFi mesh (if device supports 802.11s):
opkg install wpad-mesh-openssl    # Must replace wpad-basic if already installed
```

**Configuration**: user manually edits `/etc/bgpx/config.toml` or uses bgpx-luci web UI (accessible via standard LuCI at `192.168.1.1`).

**Limitations vs. native firmware**:
- No TPM (software key storage only — still secure via mlock, just no hardware key isolation)
- No integrated LoRa (USB LoRa adapter required for LoRa domain)
- WiFi 802.11s availability depends on device WiFi chipset (see `compatible_hardware.md`)
- No first-boot wizard — manual configuration required
- Power management: OpenWrt standard (no BGP-X-specific power daemon)

**Hardware key capability auto-detection**:
bgpx-node checks for TPM at startup:
```
if /dev/tpm0 exists and tpm2_startup succeeds:
    key_backend = "tpm"
else:
    key_backend = "software"
    log "No TPM detected; using software key storage with mlock protection"
```

No configuration change needed when moving from TPM-less to TPM hardware — the daemon detects and uses whatever is available.

---

### 3.5 bgpx-modem-firmware

**Target hardware**: ESP32-S3 or nRF52840 (Meshtastic-compatible devices — see `meshtastic_adapter.md`)

**Purpose**: Flash Meshtastic-compatible hardware to act as a BGP-X LoRa radio modem. This firmware replaces the Meshtastic routing protocol with a simple serial modem interface exposing raw LoRa radio access to a connected BGP-X router.

**Variants**:
- `bgpx-modem-firmware-esp32s3.bin`: for ESP32-S3 based devices (Heltec WiFi LoRa 32 V3, LILYGO T3S3, LILYGO T-Beam S3 Supreme)
- `bgpx-modem-firmware-nrf52840.zip`: for nRF52840 based devices (RAK4631)

**Runtime**: bare-metal or RTOS (FreeRTOS); NOT Linux. Small footprint.

**Interface**:

USB CDC-ACM serial (115200 baud, no flow control):
```
Commands (router → modem):
  TX:<hex_bytes>\n           Transmit hex payload via LoRa
  CFG:SF=9,BW=125,TX=14\n   Configure radio parameters
  STAT\n                     Request statistics
  VER\n                      Request firmware version

Responses (modem → router):
  OK\n                       TX accepted
  TXOK:<seq>\n               TX complete
  TXERR:<seq>:<reason>\n     TX failed (DUTY_CYCLE / RADIO_ERROR)
  RX:<rssi>:<snr>:<hex>\n    Received packet
  STAT:<json>\n              Statistics JSON
  VER:BGP-X Modem v0.1.0\n  Version response
```

**Duty cycle enforcement**: firmware maintains a per-region token bucket. EU 1% duty cycle: 36 seconds TX per hour maximum. Exceeding budget returns `TXERR:<seq>:DUTY_CYCLE`.

**Flash procedure**:
```bash
# ESP32-S3:
esptool.py --chip esp32s3 --port /dev/ttyUSB0 write_flash 0x0 bgpx-modem-firmware-esp32s3.bin

# nRF52840:
adafruit-nrfutil dfu serial --package bgpx-modem-firmware-nrf52840.zip -p /dev/ttyACM0 -b 115200
```

---

## 4. Common Firmware Requirements (All Variants)

These requirements apply to ALL BGP-X firmware variants regardless of target hardware:

### 4.1 Security

- **No swap**: swap must be disabled or encrypted on all BGP-X firmware. Session keys may not be paged to unencrypted storage. `swapoff -a` in all boot scripts.
- **mlock enforcement**: bgpx-node daemon requests mlock capability at startup; firmware must allow `CAP_IPC_LOCK`.
- **SSH key-only access**: password authentication disabled for SSH. No root SSH login.
- **Automatic security updates**: firmware MUST support signed OTA updates. Update mechanism verifies Ed25519 signature against release public key before applying.
- **No debug ports in production**: UART debug console disabled on production units; JTAG disabled after TPM enrollment.

### 4.2 Time Synchronization

NTP must be synchronized within 500ms maximum offset. bgpx-node checks NTP sync at startup and refuses to start if sync not established within 60 seconds. Chrony preferred over systemd-timesyncd for accuracy.

Time accuracy matters for: DHT record timestamp validation (signed_at must be within ±60s of current time); session re-handshake scheduling; advertisement expiry.

### 4.3 Prohibited Log Content

All BGP-X firmware must configure logging to prevent prohibited content:

```toml
[log]
level = "info"           # valid: error, warn, info
# Prohibited: any log output containing:
# - IP addresses of BGP-X session participants
# - session_id values
# - path_id values
# - routing domain traversal details per session
# - stream content
# - destination addresses in relay context
```

The bgpx-node daemon enforces this internally. Firmware must not add additional logging at the system level that captures network traffic.

### 4.4 OTA Update Security

All firmware supports over-the-air updates. Update security model:

1. Update package: compressed archive containing: new firmware image, version file, Ed25519 signature
2. Device fetches update package from verified BGP-X update server (via BGP-X overlay, not clearnet DNS)
3. Device verifies Ed25519 signature against embedded release public key
4. If signature valid: apply update (A/B partition scheme for atomic update)
5. If signature invalid: reject update; alert operator via LuCI and bgpx-cli event

Release public key is embedded in firmware at build time and cannot be changed without a signed update (bootstrap trust).

---

## 5. Firmware Build System

BGP-X firmware is built using OpenWrt's build system with BGP-X as an external package feed.

### Repository Structure

```
firmware/
├── bgpx-router-firmware/
│   ├── target/linux/mediatek/mt7981b/   ← Board device tree
│   ├── packages/                         ← OpenWrt package feeds
│   └── Makefile
├── bgpx-node-firmware/
│   ├── target/linux/<node-soc>/          ← TBD based on final SoC
│   ├── packages/
│   └── Makefile
├── bgpx-gateway-firmware/
│   ├── target/linux/mediatek/mt7988a/
│   ├── packages/
│   └── Makefile
└── bgpx-modem-firmware/
    ├── esp32s3/                          ← Rust embedded for Xtensa
    ├── nrf52840/                         ← Rust embedded for ARM Cortex-M4
    └── Makefile
```

### Cross-Compilation

bgpx-node is compiled for each target architecture:

| Target | Architecture | Rust target triple |
|---|---|---|
| BGP-X Router v1 | ARM Cortex-A53 (64-bit) | aarch64-linux-musl |
| BGP-X Node v1 | ARM Cortex-A53 (64-bit) or 32-bit ARM | aarch64-linux-musl or armv7-linux-musleabihf |
| BGP-X Gateway v1 | ARM Cortex-A53 (64-bit) | aarch64-linux-musl |
| bgpx-openwrt (x86) | x86-64 | x86_64-linux-musl |
| bgpx-modem (ESP32-S3) | Xtensa LX7 | xtensa-esp32s3-none-elf |
| bgpx-modem (nRF52840) | ARM Cortex-M4F | thumbv7em-none-eabihf |

### Adding a New Hardware Target

To add support for new hardware:

1. Create `target/linux/<new_soc>/` with device tree files
2. Add board-specific kernel configuration (`.config`)
3. Implement any needed kernel drivers for new radio chipsets
4. Update HAL in bgpx-transport-lora and bgpx-transport-wifi-mesh if using different radio ICs
5. Add cross-compilation target to CI
6. All other code (daemon, protocol, SDK) requires no changes

---

## 6. Firmware Version Scheme

BGP-X firmware version: `FIRMWARE_MAJOR.FIRMWARE_MINOR.DAEMON_VERSION`

Example: `1.0.0.1.0` = Firmware major 1, firmware minor 0, patch 0, bgpx-node daemon version 1.0.

Firmware major version increments for hardware breaking changes (new device tree, new radio driver ABI).
Firmware minor version increments for software-only updates (new daemon version, new packages).

OTA updates are allowed within the same firmware major version. Cross-major-version updates require manual procedure (hardware-breaking change may require re-configuration).

---

## 7. Factory Test Procedure

All BGP-X hardware units MUST pass factory test before shipping. The factory test runs `bgpx-factory-test` which verifies:

### BGP-X Router v1 Factory Test

```
[ ] TPM self-test: tpm2_selftest passes
[ ] TPM HWRNG: 32 bytes entropy generated in <100ms
[ ] LoRa SX1262 SPI communication: read version register, expect 0x22
[ ] LoRa TX: transmit test packet at +14 dBm, verify duty cycle accounting
[ ] LoRa RX: receive test packet from test jig, verify RSSI within -80 to -40 dBm
[ ] WiFi 2.4 GHz: 802.11s association with test AP, verify >-70 dBm RSSI
[ ] WiFi 5 GHz: 802.11s association with test AP, verify >-70 dBm RSSI
[ ] WAN GbE: link up at 1 Gbps, 1000 packet loopback test
[ ] LAN GbE × 3: link up on all three LAN ports
[ ] USB 3.0: enumerate test device on both ports
[ ] eMMC: write/read 64 MB test pattern, verify no errors
[ ] Secure boot: verify TPM PCR[0] matches expected boot measurement
[ ] Domain bridge test: verify DOMAIN_BRIDGE hop dispatch via WiFi + clearnet simultaneously
[ ] Cross-domain path_id table: verify insert, lookup, expiry in both tables
[ ] BGP-X daemon start: verify daemon starts, publishes test advertisement to local DHT
```

### BGP-X Node v1 Factory Test

```
[ ] LoRa SX1262 SPI: read version register
[ ] LoRa TX/RX loopback with test jig
[ ] WiFi 802.11s: association with test AP (if WiFi enabled)
[ ] BLE: advertisement detected by test scanner
[ ] Battery/solar circuit: MPPT charging at expected current from test source
[ ] Solar voltage sense: accurate within 2% of reference
[ ] eMMC/flash: write/read test
[ ] bgpx-node start in relay mode
[ ] bgpx-power: battery level correctly reported
```
