# BGP-X Mesh Network Deployment Guide

**Version**: 0.1.0-draft

---

## 1. Overview

This guide covers deploying a BGP-X mesh network — a collection of nodes communicating via radio transport (WiFi 802.11s, LoRa, BLE) without requiring individual ISP connections.

---

## 2. Use Cases

**Community LAN Privacy**: Apartment building or neighborhood LAN with shared overlay.

**Rural Connectivity**: Geographically dispersed nodes connected by LoRa and WiFi.

**Emergency/Disaster Communications**: Mesh network independent of infrastructure.

**Privacy at Events**: Temporary mesh for conference, protest, or gathering.

**Underserved Connectivity**: Community internet access via mesh + single shared gateway.

---

## 3. Deployment Planning

### Node Role Assignment

| Role | Configuration | Hardware |
|---|---|---|
| Mesh relay | WiFi mesh + LoRa | Indoor or outdoor node |
| Mesh gateway | WiFi mesh + LoRa + WAN | Gateway unit with SIM/fiber |
| LoRa repeater | LoRa only | Outdoor weatherproof unit |
| WiFi mesh extender | WiFi 802.11s only | Indoor unit or AP |
| Amplifier | LoRa amplifier | Mast-mounted unit |

### Minimum Viable Mesh

For a functional mesh that can construct valid paths internally:

```
3 nodes minimum (for 3-hop minimum path constraint):
  Node A: entry + relay
  Node B: relay
  Node C: exit (if clearnet gateway) or relay

+ 1 gateway (for clearnet access)
```

### Coverage Planning

**WiFi 802.11s**:
- Urban: 50-200m per hop
- Rural (open): 200-500m per hop
- With directional antennas: 1-10km per hop

**LoRa**:
- SF7 (fastest): 1-3km urban, 3-10km rural
- SF12 (slowest, longest range): 5-15km urban, 10-50km rural

Combine: WiFi mesh for high-bandwidth backbone; LoRa for long-range DHT connectivity between islands.

---

## 4. Hardware Setup

### GL.iNet GL-MT3000 (Mesh Node)

```bash
# Flash OpenWrt firmware
# Download from: https://firmware-selector.openwrt.org/?target=mediatek/filogic
# Flash via LuCI or sysupgrade

# SSH into device
ssh root@192.168.8.1

# Install BGP-X packages
opkg update
opkg install bgpx-node bgpx-cli bgpx-mesh bgpx-luci

# Configure WiFi mesh interface
cat >> /etc/config/wireless << 'EOF'
config wifi-device 'radio_mesh'
    option type 'mac80211'
    option channel '6'
    option hwmode '11g'

config wifi-iface 'mesh0'
    option device 'radio_mesh'
    option mode 'mesh'
    option mesh_id 'bgpx-community-1'
    option encryption 'sae'
    option key 'mesh-psk-change-me'
    option network 'mesh'
EOF

/etc/init.d/network restart
```

### LoRa USB Module Setup

```bash
# Install LoRa driver
opkg install kmod-usb-serial-ch341 bgpx-lora

# Identify device
ls /dev/ttyUSB*  # Usually /dev/ttyUSB0

# Test module
bgpx-cli mesh radio-test \
    --interface /dev/ttyUSB0 \
    --frequency 868.0 \
    --sf 7

# Configure BGP-X for LoRa
cat >> /etc/bgpx/node.toml << 'EOF'
[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
lora_spreading_factor = 7
lora_bandwidth_khz = 125
beacon_interval_seconds = 30
EOF
```

---

## 5. BGP-X Mesh Configuration

```toml
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
# Note: no "exit" role unless this node has WAN access

[network]
# No internet WAN — mesh only
listen_port = 7474

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
beacon_interval_seconds = 30

[dht]
# No internet bootstrap nodes — mesh bootstrap via beacons
bootstrap_nodes = []

[advertisement]
region = "SA"
country = "CO"
asn = 65001  # Private ASN for mesh network
is_gateway = false
```

---

## 6. Gateway Configuration

The gateway node bridges mesh and internet:

```toml
[node]
roles = ["relay", "entry", "exit", "discovery"]

[network]
listen_addr = "0.0.0.0"   # Internet-facing
listen_port = 7474
public_addr = "<gateway_public_ip>"

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]

[advertisement]
is_gateway = true
gateway_for_mesh = ["bgpx-community-1"]
provides_clearnet_exit = true

[exit_policy]
logging_policy = "none"
dns_mode = "doh"
ech_capable = true
```

---

## 7. First Boot and Bootstrap

```bash
# Generate node key
bgpx-cli keygen --output /etc/bgpx/node_private_key

# Start daemon
/etc/init.d/bgpx start

# Watch for beacon discovery
bgpx-cli events watch &

# Trigger immediate beacon
bgpx-cli mesh beacon

# Wait 60 seconds for routing table population
sleep 60

# Check mesh peers
bgpx-cli mesh peers
# Expected: 1+ peers discovered via beacons

# Check DHT
bgpx-cli dht stats
# Expected: routing_table_size >= 3

# Test path construction (once 3+ nodes online)
bgpx-cli paths build --segments "bgpx-default:3"
```

---

## 8. Meshtastic Hardware Integration

Use low-cost Meshtastic devices as LoRa radio modems:

### Compatible Hardware (Radio Modem Mode)

- LILYGO T-Beam Supreme (ESP32-S3 + SX1262)
- Heltec LoRa32 v3 (ESP32-S3 + SX1262)
- RAK WisBlock 4631 (nRF52840 + SX1262)

### Flashing BGP-X Modem Firmware

```bash
# Download BGP-X modem firmware for your device
# (Replaces Meshtastic firmware with BGP-X radio modem protocol)
esptool.py --port /dev/ttyUSB0 write_flash \
    0x0 bgpx-modem-t-beam-supreme-v0.1.0.bin

# Connect to router
# Modem auto-detected as /dev/ttyUSB0

# Configure
bgpx-cli mesh radio-config \
    --interface /dev/ttyUSB0 \
    --frequency 868.0 \
    --sf 7
```

---

## 9. Multi-Island Bridging

Connect geographically separated mesh islands via internet:

```
Island A (Lima) ──────── Gateway A ──────── Internet ──────── Gateway B ──── Island B (Bogotá)
  (WiFi mesh)              (WAN)                                (WAN)              (LoRa mesh)
```

### Configuration

Both gateways configured as internet-accessible overlay nodes AND mesh gateways.

Clients on Island A automatically discover Island B nodes via DHT cross-domain synchronization performed by Gateway A and Gateway B.

No special configuration needed on client nodes — the gateway handles bridging transparently.

### Pool for Bridge Path Control

Create a named pool containing the internet relay nodes between islands:

```toml
# Island A client path config
[path_constraints]
segments = [
    { pool = "lima-island-relays", hops = 2 },
    { pool = "inter-island-bridge", hops = 2 },
    { pool = "bogota-island-relays", hops = 1, exit = false },
]
```

---

## 10. Monitoring Mesh Networks

```bash
# View all mesh peers
bgpx-cli mesh peers
# Output: NodeID, Transport, Last Seen, RTT (ms), Location

# Mesh statistics
bgpx-cli mesh stats
# Output: beacons_sent, beacons_received, fragments_sent, fragments_received,
#         reassembly_successes, reassembly_failures, store_and_forward_queued

# Gateway bridge status
bgpx-cli gateway status
# Output: bridge_active, records_synced_internet_to_mesh, records_synced_mesh_to_internet

# LoRa link quality
bgpx-cli mesh lora-stats
# Output: rssi, snr, last_hop_latency_ms, duty_cycle_remaining_pct

# Test LoRa range
bgpx-cli mesh radio-test --interface /dev/ttyUSB0 --ping <remote_node_id>
```

---

## 11. Troubleshooting

**No mesh peers discovered**:
```bash
# Check interface up
ip link show mesh0

# Trigger immediate beacon
bgpx-cli mesh beacon

# Check LoRa module
bgpx-cli mesh radio-status
# Verify frequency matches other nodes
```

**Path construction fails** (mesh only):
```bash
# Check node count
bgpx-cli dht stats

# Need minimum 3 nodes for 3-hop path
# If < 3 nodes visible in DHT: deploy more nodes or check beacon propagation
```

**Gateway not reachable**:
```bash
# Verify gateway WAN
ping 8.8.8.8 -c 3

# Verify gateway BGP-X
bgpx-cli status

# Check bridge active
bgpx-cli gateway status

# If WAN down: check cellular backup
```

**LoRa duty cycle exceeded**:
```bash
bgpx-cli mesh lora-stats | grep duty_cycle
# If duty_cycle_remaining_pct = 0: must wait for cycle reset
# LoRa rate limits to 1% airtime per hour in EU
```
