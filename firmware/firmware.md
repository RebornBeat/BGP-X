# BGP-X Firmware Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X firmware is the operating system and runtime environment for BGP-X hardware. The core BGP-X software (bgpx-node daemon, bgpx-cli, bgpx-sdk) is hardware-agnostic. Firmware provides the hardware-specific layer: device drivers, kernel configuration, init system, first-boot wizard, and platform-specific optimizations.

**All firmware targets run the same bgpx-node daemon binary** (cross-compiled for the appropriate CPU architecture). Protocol behavior, configuration format, and capabilities are identical across all targets. Hardware differences affect only drivers and power management — not protocol, not routing, not cryptography.

### 1.1 Firmware Adaptability Principle

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

## 2. Firmware Variants

BGP-X firmware is organized into five primary variants, correlating to the hardware ecosystem:

| Variant | Target Hardware | Primary Use |
|---|---|---|
| `bgpx-router-firmware` | BGP-X Router v1 | End-user AIO, replaces home router |
| `bgpx-node-firmware` | BGP-X Node v1 | Community relay, domain bridge, range extension |
| `bgpx-gateway-firmware` | BGP-X Gateway v1 | Provider exit node and infrastructure |
| `bgpx-openwrt` | Third-party OpenWrt routers | Software-only package |
| `bgpx-client-firmware` | BGP-X Client Node (Tier 2) | Low-cost mesh endpoint (ESP32-S3/nRF52840) |
| `bgpx-modem-firmware` | BGP-X Adapter/Dongle (Tier 3) | USB LoRa modem |

---

## 3. `bgpx-router-firmware` (BGP-X Router v1)

**Target hardware**: BGP-X Router v1 (reference: ARM dual-core Cortex-A53 SoC, e.g., MT7981B. See `router_spec.md`)

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

---

## 4. `bgpx-node-firmware` (BGP-X Node v1)

**Target hardware**: BGP-X Node v1 (reference: low-power ARM Cortex-A53 or equivalent. See `node_spec.md`)

**Purpose**: Community contributor node firmware. Multi-radio mesh relay, domain bridge, and range extension node with solar/battery power management.

**Base OS**: OpenWrt 23.05+ (BGP-X Node v1 custom build target)

**Kernel**: Linux 5.15 LTS (stable, wide low-power ARM BSP support) or Linux 6.1 LTS

**Capabilities enabled**:
- BGP-X mesh relay (LoRa + WiFi mesh + BLE, configurable)
- BGP-X domain bridge (with optional WAN Ethernet)
- BGP-X range extension mode
- Community gateway (with WAN + exit policy)
- Solar/battery power management daemon (`bgpx-power`)
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

### 4.1 Low-Power Build Differences

Compared to router firmware:
- Reduced maximum session count (default 2,000 vs. 10,000)
- Reduced path_id table capacity (5,000 entries vs. unlimited)
- Power-aware scheduling: BGP-X daemon yields to power management events
- Radio duty cycle integrated with `bgpx-power` battery state
- Swap disabled (same as all BGP-X firmware — key material must not be in swap)
- Optional: no web UI (CLI only, saves ~4 MB RAM)

### 4.2 `bgpx-power` Daemon

The `bgpx-power` daemon monitors battery state and adjusts BGP-X operation:

| Battery Level | Action |
|---|---|
| 100-50% | Normal operation; all radios at configured TX power |
| 50-30% | Reduce LoRa TX power by 4 dBm |
| 30-15% | Disable WiFi mesh if LoRa mesh available; LoRa TX further reduced |
| 15-5% | Enter range extension mode only; minimal session origination |
| <5% | Emergency: relay-only, minimum power, disable BLE |
| 0% (threshold) | Graceful shutdown: publish `NODE_WITHDRAW`; wait for propagation; power off |

---

## 5. `bgpx-gateway-firmware` (BGP-X Gateway v1)

**Target hardware**: BGP-X Gateway v1 (reference: high-throughput ARM quad-core, e.g., MT7988A. See `gateway_spec.md`)

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

### 5.1 Provider-Optimized Configuration

Gateway firmware defaults differ from Router firmware:
- `max_sessions` = 50,000
- `SO_RCVBUF` = `SO_SNDBUF` = 64 MB (vs. 16 MB default)
- Worker threads = `CPU_CORES` (all cores, no OS reserve)
- Sender threads = 8 (vs. 4 default)
- Session idle timeout = 120 seconds (vs. 90 seconds)
- `path_id` table shards = 256 (vs. 64 default)
- ECH cache size = 10,000 entries
- DoH resolver threads = 4

---

## 6. `bgpx-openwrt` (Third-Party Routers)

**Target hardware**: Any compatible OpenWrt 23.05+ router (see `compatible_hardware.md`)

**Purpose**: Software-only package for third-party compatible routers. No hardware customization — installs as standard OpenWrt packages.

**Package set** (installed via opkg):
```
opkg install bgpx-node bgpx-cli bgpx-luci

# Optional for LoRa domain bridge:
opkg install bgpx-modem-driver    # USB LoRa serial protocol driver

# Optional for WiFi mesh (if device supports 802.11s):
opkg install wpad-mesh-openssl    # Must replace wpad-basic if installed
```

**Configuration**: User manually edits `/etc/bgpx/config.toml` or uses bgpx-luci web UI (accessible via standard LuCI at router IP).

**Limitations vs. native firmware**:
- No TPM (software key storage only — still secure via mlock, just no hardware key isolation)
- No integrated LoRa (USB LoRa adapter required for LoRa domain)
- WiFi 802.11s availability depends on device WiFi chipset
- No first-boot wizard — manual configuration required
- Power management: OpenWrt standard (no BGP-X-specific power daemon)

**Hardware key capability auto-detection**:
`bgpx-node` checks for TPM at startup:
```bash
# Pseudocode
if /dev/tpm0 exists and tpm2_startup succeeds:
    key_backend = "tpm"
else:
    key_backend = "software"
    log "No TPM detected; using software key storage with mlock protection"
```

No configuration change needed when moving from TPM-less to TPM hardware — daemon detects and adapts.

---

## 7. `bgpx-client-firmware` (BGP-X Client Node — Tier 2)

**Target hardware**: BGP-X Client Node (reference: LILYGO T3S3, T-Beam, T-Echo, Heltec V3, RAK WisBlock 4631 — ESP32-S3 or nRF52840 based development boards. See `client_node_spec.md`)

**Purpose**: Low-cost, battery-powered endpoint for connecting to BGP-X mesh networks. Does NOT route traffic for others. Designed for individual users, sensors, and portable use.

**Runtime**: FreeRTOS or bare-metal — NOT Linux.

**Capabilities**:
- Connect to BGP-X mesh island as client
- Send/receive BGP-X encrypted traffic
- Query the DHT (no storage)
- Request paths to services
- Operate on battery for extended periods

**Limitations**:
- Cannot route traffic for other nodes
- Cannot store DHT records
- Cannot participate as a relay in BGP-X paths
- Cannot run the full `bgpx-node` daemon (insufficient RAM/CPU)

**Software stack**:
```
ESP-IDF 5.0+ / Arduino ESP32 Core
├── bgpx-client (subset protocol implementation)
├── sx1262-driver (LoRa radio driver)
├── bgpx-power (battery/solar management)
└── USB CDC-ACM (serial interface)
```

---

## 8. `bgpx-modem-firmware` (BGP-X Adapter/Dongle — Tier 3)

**Target hardware**: BGP-X Adapter/Dongle (reference: ESP32-S3 or nRF52840. See `adapter_spec.md`)

**Purpose**: USB dongle acting as a transparent LoRa radio modem. Host computer runs full `bgpx-node` daemon; dongle handles LoRa transmission/reception only. No routing intelligence on dongle.

**Variants**:
- `bgpx-modem-firmware-esp32s3.bin`: For ESP32-S3 devices (Heltec WiFi LoRa 32 V3, LILYGO T3S3, T-Beam S3 Supreme)
- `bgpx-modem-firmware-nrf52840.zip`: For nRF52840 devices (RAK4631, LILYGO T-Echo)

**Interface**: USB CDC-ACM serial (115200 baud, no flow control)

```
Commands (Router → Modem):
  TX:<hex_bytes>\n           Transmit hex payload via LoRa
  CFG:SF=9,BW=125,TX=14\n   Configure radio parameters
  STAT\n                     Request statistics
  VER\n                      Request firmware version

Responses (Modem → Router):
  OK\n                       TX accepted
  TXOK:<seq>\n               TX complete
  TXERR:<seq>:<reason>\n     TX failed (DUTY_CYCLE / RADIO_ERROR)
  RX:<rssi>:<snr>:<hex>\n    Received packet
  STAT:<json>\n              Statistics JSON
  VER:BGP-X Modem v0.1.0\n  Version response
```

**Duty cycle enforcement**: Firmware maintains per-region token bucket. EU 1% duty cycle: 36 seconds TX per hour maximum. Exceeding budget returns `TXERR:<seq>:DUTY_CYCLE`.

**Flash procedure**:
```bash
# ESP32-S3:
esptool.py --chip esp32s3 --port /dev/ttyUSB0 write_flash 0x0 bgpx-modem-firmware-esp32s3.bin

# nRF52840:
adafruit-nrfutil dfu serial --package bgpx-modem-firmware-nrf52840.zip -p /dev/ttyACM0 -b 115200
```

---

## 9. Dual-Stack Operation (BGP + BGP-X)

In dual-stack mode (firmware variants 5.1 and 5.2 with WAN), the router runs standard IP networking alongside BGP-X.

### Network Architecture

```
WAN (eth0) → ISP → Internet (BGP-routed)
    ↑
    Both BGP-X overlay traffic (UDP/7474) AND standard traffic
    share this WAN connection

LAN (eth1-4, wlan0) → Devices
    ↓
    Routing Policy Engine decides per-flow:
    → bgpx0 (BGP-X overlay) for privacy-selected traffic
    → eth0 (standard routing) for bypass traffic
```

### Interface Configuration

```
eth0:   WAN interface (DHCP from ISP)
eth1-4: LAN switch
wlan0:  WiFi access point
bgpx0:  BGP-X TUN interface (MTU 1200, address 100.64.0.1/10)
```

### DNS in Dual-Stack Mode

- **BGP-X traffic**: DNS resolved via overlay DNS proxy (port 5300)
- **Standard traffic**: DNS resolved via system resolver
- `dnsmasq` routes queries to appropriate resolver based on routing policy

### SDK and Control Connectivity for LAN Devices

LAN devices can access router BGP-X daemon:
- **Control socket**: `ssh -L /tmp/bgpx-ctl.sock:/var/run/bgpx/control.sock user@router`
- **SDK socket**: configured as TCP endpoint (`sdk_api.tcp_listen = "192.168.1.1:7475"`)
- **Standard application mode**: no SDK access needed for transparent protection

---

## 10. BGP-X Only Firmware Mode

All LAN traffic forced through BGP-X overlay. No bypass path.

**What changes vs dual-stack**:
- Routing policy: all traffic → bgpx0 (no standard routing option)
- No bypass rules (except local LAN addresses and BGP-X control traffic)
- WAN still uses BGP-routed internet as transport for overlay

**What stays the same**:
- Same daemon code
- Same SDK and Control socket access
- WAN connection uses BGP as physical transport
- Same hardware

**When to use**: maximum privacy, no accidental bypass risk.

---

## 11. Mesh-Only Operation (No WAN/ISP)

Router operates without WAN/ISP. Mesh transport only.

**Configuration Example**:
```toml
[node]
roles = ["relay", "entry", "discovery"]
# No exit role — no clearnet access without gateway

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
beacon_interval_seconds = 30

[network]
# No WAN config
listen_addr_mesh = "mesh0"
listen_port = 7474

[dht]
# Mesh-only: no internet bootstrap nodes
# Bootstrap via MESH_BEACON broadcast
bootstrap_nodes = []
```

### LoRa USB Module Setup

Compatible LoRa USB modules:
- RAK2287 USB
- Dragino LPS8N USB
- Generic SX1262 USB dongle

Installation:
```bash
opkg install bgpx-mesh
bgpx-cli mesh radio --interface /dev/ttyUSB0 --frequency 868.0 --sf 7
```

---

## 12. Domain Bridge / Gateway Configuration

Router has both mesh transport and WAN connection. Bridges mesh community to clearnet.

**Configuration Example**:
```toml
[node]
roles = ["relay", "entry", "exit", "discovery"]

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]

[network]
listen_addr = "0.0.0.0"       # WAN-facing for internet overlay
listen_addr_mesh = "mesh0"    # Mesh-facing
listen_port = 7474

[exit_policy]
logging_policy = "none"
dns_mode = "doh"
ech_capable = true

[advertisement]
is_gateway = true
gateway_for_mesh = ["bgpx-community-mesh-1"]
provides_clearnet_exit = true
```

### Multi-WAN Gateway

For critical gateway availability:

```toml
[network]
wan_primary = "eth0"        # Fiber WAN
wan_backup = "wwan0"        # 4G/5G cellular backup

failover_enabled = true
failover_check_interval_seconds = 30
```

BGP-X daemon continues operation during WAN failover. Paths through gateway may have increased latency during failover; client path quality monitoring will detect and potentially trigger rebuild.

---

## 13. Routing Policy on Router Firmware

OpenWrt integration for routing policy:

### iptables Integration (TUN Mode)

```bash
# Route LAN traffic through BGP-X for IPv4
ip rule add from 192.168.1.0/24 table bgpx priority 100
ip route add default dev bgpx0 table bgpx

# Bypass for BGP-X control traffic (prevent recursion)
ip rule add to <entry_node_ip> table main priority 50

# Bypass for LAN
ip rule add to 192.168.0.0/16 table main priority 50
```

### nftables Integration

```nft
table inet bgpx_policy {
    chain prerouting {
        type filter hook prerouting priority -100;
        ip daddr 192.168.0.0/16 return  # LAN bypass
        ip daddr @bgpx_entry_nodes return  # Control bypass
        ip protocol udp udp dport 7474 return  # Overlay bypass
        counter mark set 0x100  # Mark for BGP-X routing
    }
}
```

---

## 14. LuCI Web Interface

The `bgpx-luci` package provides an OpenWrt web interface:

**Features**:
- Node status dashboard (state, sessions, bandwidth)
- Path information (pool segments, latency, quality)
- Pool management (add/view pools)
- Routing policy management (add/remove rules, test matching)
- Exit node configuration (for gateway firmware)
- Mesh status (neighbors, beacons, DHT coverage)
- Node database browser
- Reputation data viewer
- Domain bridge status (bridge pairs, latency)
- Mesh island management (gateway operators)

**Access**: `https://[router-ip]/cgi-bin/luci/admin/network/bgpx`

---

## 15. Performance Expectations

### Tier 1 Hardware (GL-MT6000, MT7986A)

| Mode | Expected Throughput |
|---|---|
| Client (4 hops) | 80-150 Mbps |
| Relay node | 300-500 Mbps |
| With PT (obfs4) | ~85% of above |
| With cover traffic | ~90% of above |

### Latency Overhead

- Processing overhead per hop: ~1-5ms on Tier 1
- Network latency dominates: 80-300ms for 4-hop geographically diverse path
- Total RTT overhead vs direct: ~85-310ms

### Mesh Transport Performance

| Transport | Throughput | Latency |
|---|---|---|
| WiFi 802.11s | 10-100+ Mbps | 1-20ms/hop |
| LoRa | 0.3-50 Kbps | 100ms-5s/hop |
| BLE | 1-2 Mbps | 10-100ms/hop |

---

## 16. Build System

BGP-X firmware is built using OpenWrt's build system with BGP-X as an external package feed.

### Repository Structure

```
firmware/
├── bgpx-router-firmware/
│   ├── target/linux/mediatek/mt7981b/   ← Board device tree
│   ├── packages/                         ← OpenWrt package feeds
│   └── Makefile
├── bgpx-node-firmware/
│   ├── target/linux/<node-soc/          ← TBD based on final SoC
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
4. Update HAL in `bgpx-transport-lora` and `bgpx-transport-wifi-mesh` if using different radio ICs
5. Add cross-compilation target to CI
6. All other code (daemon, protocol, SDK) requires no changes

---

## 17. Firmware Version Scheme

BGP-X firmware version: `FIRMWARE_MAJOR.FIRMWARE_MINOR.DAEMON_VERSION`

Example: `1.0.0.1.0` = Firmware major 1, firmware minor 0, patch 0, bgpx-node daemon version 1.0.

Firmware major version increments for hardware breaking changes (new device tree, new radio driver ABI).
Firmware minor version increments for software-only updates (new daemon version, new packages).

OTA updates are allowed within the same firmware major version. Cross-major-version updates require manual procedure (hardware-breaking change may require re-configuration).

---

## 18. Firmware Update Process

All firmware variants support secure OTA updates:

1. Update package downloaded from `updates.bgpx.network` (or local update server)
2. Package signature verified using BGP-X release key (Ed25519)
3. If signature valid: flash to inactive partition (A/B partition scheme)
4. Reboot to new partition
5. If boot fails: automatic rollback to previous partition

Update key pinned in firmware; cannot be changed without full reflash.

---

## 19. Hardware Security Features

| Feature | bgpx-router-firmware | bgpx-node-firmware | bgpx-gateway-firmware |
|---|---|---|---|
| TPM 2.0 for key storage | ✅ | Optional | ✅ |
| Secure Boot | ✅ | Optional | ✅ |
| Encrypted storage (node private key) | ✅ TPM-sealed | ✅ (TPM or software) | ✅ TPM-sealed |
| Physical reset button (clears key) | ✅ | ✅ | ✅ |
| JTAG disabled in production | ✅ | ✅ | ✅ |

---

## 20. First Boot Provisioning

```
1. Power on device
2. Device broadcasts initial MESH_BEACON with generated temporary identity
3. User connects to provisioning WiFi AP: "bgpx-setup-XXXXXX"
4. LuCI provisioning wizard runs:
   a. Set island_id
   b. Configure gateway vs. mesh-only mode
   c. If gateway: enter WAN credentials, verify internet connectivity
   d. Generate permanent Ed25519 keypair (sealed to TPM if available)
   e. Configure LoRa frequency for local regulations
5. Device reboots with permanent identity
6. If gateway: publishes advertisement to unified DHT
```

---

## 21. Factory Test Procedure

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

---

## 22. LED Status Indicators

| LED Color / Pattern | Meaning |
|---|---|
| Green solid | BGP-X active, DHT connected |
| Green slow blink | BGP-X active, mesh-only (no internet) |
| Yellow solid | Provisioning mode |
| Yellow fast blink | Firmware update in progress |
| Red solid | Error — check logs via serial console |
| Red slow blink | BGP-X daemon not running |
| Blue blink | Active LoRa transmission |
| Off | Powered off or boot |

---

## 23. Deprecated Concepts

**"BGP-X Amplifier v1"**: This concept has been retired. The BGP-X Node v1 in **Range Extension mode** provides equivalent coverage benefits while maintaining full BGP-X encryption at every hop. A pure PHY-level amplifier that relays traffic without encryption introduces a security vulnerability — traffic would be exposed at the relay's radio interface.

**For range extension use cases**: Use the BGP-X Node v1 in Range Extension mode.

**If a genuine LoRa PHY-level amplifier (bidirectional RF signal amplifier, no digital processing)** is required, standard RF amplifier modules (SX1301-based or custom) serve this purpose. This is not a BGP-X product — it is a radio hardware component outside the BGP-X ecosystem.
