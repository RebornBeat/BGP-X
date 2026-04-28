# BGP-X Firmware Specification

**Version**: 0.1.0-draft

---

## 1. Firmware Variants

| Variant | Target | Role |
|---|---|---|
| bgpx-mesh-node | BGP-X hardware (MT7981B) | Full routing with WiFi mesh and LoRa |
| bgpx-gateway-node | Gateway hardware (MT7988A) | Domain bridge node with clearnet exit |
| bgpx-amplifier | Amplifier hardware (STM32H7 + radio) | Range extension only |
| bgpx-openwrt | OpenWrt-compatible routers | Software package |

---

## 2. bgpx-mesh-node Firmware

### Hardware Base

BGP-X unified hardware platform: MT7981B ARM Cortex-A53 dual-core 1.3GHz, 512MB DDR4, 128MB NAND + 8GB eMMC, TPM 2.0.

Radios: WiFi 6 (MT7915, 802.11s mesh mode), LoRa (SX1262).

### Software Stack

```
OpenWrt 23.05 (kernel 6.1 LTS)
bgpx-node daemon (mesh firmware build)
bgpx-cli (control utility)
LuCI web interface (bgpx-luci package)
wpad-mesh-openssl (WiFi 802.11s)
```

### Configuration: Mesh Node (No Internet)

```toml
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
name = "bgpx-mesh-001"
data_dir = "/overlay/bgpx"

[[routing_domains]]
domain_type = "mesh"
island_id = "ISLAND_ID_HERE"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
wifi_mesh_id = "bgpx-ISLAND_ID_HERE"
lora_interface = "/dev/ttyS1"
lora_frequency_mhz = 868.0
lora_spreading_factor = 7

[dht]
bootstrap_nodes = []    # Mesh-only: bootstrap via MESH_BEACON

[mesh_island]
auto_publish_advertisement = false  # No internet; cannot publish to unified DHT

[extensions]
cross_domain_routing_enabled = true
domain_bridge_enabled = false  # Not a bridge; no clearnet endpoint
mesh_island_routing = true
```

### Configuration: Gateway/Bridge Node (With Internet)

```toml
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "exit", "discovery"]
name = "bgpx-gateway-001"
data_dir = "/overlay/bgpx"

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
public_addr = ""      # Auto-detected via STUN
public_port = 7474

[[routing_domains]]
domain_type = "mesh"
island_id = "ISLAND_ID_HERE"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyS1"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true

[mesh_island]
auto_publish_advertisement = true
advertisement_interval_hours = 8
island_dht_sync = true

[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.0/8"]
logging_policy = "none"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true

[extensions]
cross_domain_routing_enabled = true
domain_bridge_enabled = true
mesh_island_routing = true
```

---

## 3. bgpx-gateway-node Firmware

### Hardware Base

Upgraded gateway hardware: MT7988A ARM Cortex-A73 quad-core, 1GB RAM, 8GB eMMC, WAN ports, SFP+, mPCIe (4G/5G).

Domain bridge capable by default. Supports multiple routing domains simultaneously. Higher throughput for production gateway deployments.

---

## 4. bgpx-amplifier Firmware

### Hardware Base

STM32H7 MCU + SX1262 LoRa radio. No Linux. No routing intelligence. Battery-powered option.

### Function

Pure MESH_BEACON + MESH_FRAGMENT forwarding. No DHT. No path construction. No session establishment. Just radio relay.

Configured via USB serial during setup. Broadcasts MESH_BEACON every 30 seconds. Forwards all received LoRa frames to all mesh neighbors.

---

## 5. Firmware Update Process

All firmware variants support secure OTA updates:

1. Update package downloaded from `updates.bgpx.network` (or local update server)
2. Package signature verified using BGP-X release key (Ed25519)
3. If signature valid: flash to inactive partition (A/B partition scheme)
4. Reboot to new partition
5. If boot fails: automatic rollback to previous partition

Update key pinned in firmware; cannot be changed without full reflash.

---

## 6. Hardware Security Features

| Feature | bgpx-mesh-node | bgpx-gateway-node |
|---|---|---|
| TPM 2.0 for key storage | ✅ | ✅ |
| Secure Boot | ✅ | ✅ |
| Encrypted storage (node private key) | ✅ TPM-sealed | ✅ TPM-sealed |
| Physical reset button (clears key) | ✅ | ✅ |
| JTAG disabled in production | ✅ | ✅ |

---

## 7. First Boot Provisioning

```
1. Power on device
2. Device broadcasts initial MESH_BEACON with generated temporary identity
3. User connects to provisioning WiFi AP: "bgpx-setup-XXXXXX"
4. LuCI provisioning wizard runs:
   a. Set island_id
   b. Configure gateway vs. mesh-only mode
   c. If gateway: enter WAN credentials, verify internet connectivity
   d. Generate permanent Ed25519 keypair (sealed to TPM)
   e. Configure LoRa frequency for local regulations
5. Device reboots with permanent identity
6. If gateway: publishes advertisement to unified DHT
```

---

## 8. LED Status Indicators

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
