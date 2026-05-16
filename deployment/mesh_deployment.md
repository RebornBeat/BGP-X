# BGP-X Mesh Network Deployment Guide

**Version**: 0.1.0-draft

---

## 1. Mesh Island Concepts

A mesh island is a named collection of BGP-X nodes communicating via radio transport (WiFi 802.11s, LoRa, BLE, Ethernet P2P). It operates as a **first-class routing domain** — discoverable from the unified DHT, reachable from clearnet clients without special hardware, and independently functional when offline.

Every mesh island needs:
- An `island_id` (descriptive, globally unique string)
- At least 3 nodes for useful routing (to satisfy minimum path constraints)
- At least 1 gateway/domain bridge node with internet connectivity for cross-domain access (optional for isolated operation)

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
| Mesh relay | WiFi mesh + LoRa | BGP-X Node v1 (outdoor) or Router v1 |
| Mesh gateway / Domain bridge | WiFi mesh + LoRa + WAN | BGP-X Node v1 with WAN or Router v1 |
| LoRa range extender | LoRa only, Range Extension mode | BGP-X Node v1 (outdoor, solar) |
| WiFi mesh extender | WiFi 802.11s only | BGP-X Node v1 or Router v1 |
| Client endpoint | LoRa or WiFi mesh, no routing for others | BGP-X Client Node (Tier 2) or Adapter (Tier 3) |

### Minimum Viable Mesh

For a functional mesh that can construct valid paths internally:

```
3 nodes minimum (for 3-hop minimum path constraint):
  Node A: entry + relay
  Node B: relay
  Node C: exit (if clearnet gateway) or relay

+ 1 gateway / domain bridge node (for clearnet access)
```

### Coverage Planning

**WiFi 802.11s**:
- Urban: 50-200m per hop
- Rural (open): 200-500m per hop
- With directional antennas: 1-10km per hop

**LoRa**:
- SF7 (fastest): 1-3km urban, 3-10km rural
- SF12 (slowest, longest range): 5-15km urban, 10-50km rural

**Combine**: WiFi mesh for high-bandwidth backbone; LoRa for long-range DHT connectivity between islands.

---

## 4. Hardware Setup

Refer to `/hardware/hardware_spec.md` and `/hardware/node_spec.md` for full specifications.

### BGP-X Node v1 / Router v1 (Reference Implementation)

**Example: GL.iNet GL-MT3000 (Mesh Node)**

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
uci set wireless.radio0.disabled=0
uci set wireless.mesh.network=mesh
uci set wireless.mesh.mode=mesh
uci set wireless.mesh.mesh_id=bgpx-your-island-name
uci set wireless.mesh.encryption=none
uci commit wireless
wifi reload
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
```

---

## 5. BGP-X Mesh Configuration

Mesh nodes require configuration of the `routing_domains` array with `domain_type = "mesh"`.

### Mesh-Only Node (No Internet)

```toml
# /etc/bgpx/node.toml
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
# Note: no "exit" role unless this node has WAN access

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
lora_spreading_factor = 7
lora_bandwidth_khz = 125
lora_tx_power_dbm = 14        # Respect regulatory limits
```

### Gateway / Domain Bridge Node (Clearnet + Mesh)

The gateway node bridges mesh and internet:

```toml
[node]
roles = ["relay", "entry", "exit", "discovery"]

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
public_addr = "YOUR_PUBLIC_IP"
public_port = 7474

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true

[advertisement]
is_gateway = true
gateway_for_mesh = ["your-island-name"]
provides_clearnet_exit = true

[exit_policy]
logging_policy = "none"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

### Gateway with LoRa Fallback

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
```

### Mesh-to-Mesh Bridge (No Clearnet)

Connects two separate mesh islands (e.g., via long-range LoRa link) without traversing the public internet.

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "island-a"
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"

[[routing_domains]]
domain_type = "mesh"
island_id = "island-b"
transports = ["lora"]
lora_interface = "/dev/ttyUSB1"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true
```

---

## 6. Domain Bridge Node Deployment

A domain bridge node connects two routing domains. It is the enabling component for cross-domain routing.

### Minimal Bridge Node Requirements

- Both a clearnet network interface (WAN)
- And at least one mesh radio interface (WiFi mesh or LoRa)

The **BGP-X Node v1** with WAN Ethernet and a LoRa/WiFi radio is the minimum viable domain bridge hardware.

---

## 7. CRITICAL: Multiple Bridge Nodes Per Island

A mesh island with only ONE bridge node has a single point of failure. If that bridge goes offline, the island loses all cross-domain connectivity.

**Minimum recommended**: 2 independent bridge nodes per island, operated by different operators in different physical locations.

```
Island lima-district-1
  ├── Bridge Node A: clearnet via fiber, operator X, north Lima
  └── Bridge Node B: clearnet via 4G, operator Y, south Lima
```

BGP-X path construction automatically detects and avoids the offline bridge node. No manual reconfiguration required.

Verify resilience:
```bash
bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1"
# Should show 2+ bridge nodes

bgpx-cli domains test --from clearnet --to "mesh:lima-district-1"
# Should succeed even if one bridge is offline
```

---

## 8. Clearnet User Accessing Mesh Service

When a clearnet user needs to reach a service in your mesh island:

**What they need**: BGP-X daemon installed on their device (no mesh hardware, no BGP-X router needed).

**What happens**:
1. Their daemon queries unified DHT for bridge nodes to your island
2. Constructs path: clearnet relays → your bridge node → mesh relays → service
3. Your bridge node handles all radio transmission on their behalf

**What you need to ensure**:
- Your bridge node has a publicly reachable clearnet UDP endpoint (7474/UDP)
- Your bridge node is publishing DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE to DHT
- Your island has sufficient relay nodes for path construction

```bash
# Verify island is discoverable from clearnet
bgpx-cli domains island-status --island-id your-island-name
# online: true
# dht_coverage: "full"
```

---

## 9. Meshtastic / Client Node Integration

Use low-cost devices as LoRa radio modems or client endpoints.

### BGP-X Client Node (Tier 2)

Reference hardware: LILYGO T3S3, T-Beam Supreme, Heltec LoRa32 v3, RAK WisBlock 4631.

These devices run **BGP-X Client Firmware** (subset implementation, not the full daemon). They can connect to the mesh as endpoints but do not route for others.

### BGP-X Adapter/Dongle (Tier 3)

Reference hardware: ESP32-S3 + SX1262.

Acts as a USB LoRa modem for existing computers. The host runs the full `bgpx-node` daemon; the adapter handles LoRa transmission.

### Flashing BGP-X Modem Firmware

```bash
# Download BGP-X modem firmware for your device
esptool.py --port /dev/ttyUSB0 write_flash \
    0x0 bgpx-modem-t-beam-supreme-v0.1.0.bin

# Configure on host daemon
bgpx-cli mesh radio-config \
    --interface /dev/ttyUSB0 \
    --frequency 868.0 \
    --sf 7
```

---

## 10. Multi-Island Bridging

Connect geographically separated mesh islands via internet:

```
Island A (Lima) ──────── Gateway A ──────── Internet ──────── Gateway B ──── Island B (Bogotá)
  (WiFi mesh)              (WAN)                                (WAN)              (LoRa mesh)
```

Both gateways are configured as internet-accessible overlay nodes AND mesh gateways.

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

## 11. Island Naming Best Practices

Island IDs should be:
- Descriptive and geographically specific (reduces collision probability)
- Lowercase with hyphens
- Short enough to be human-readable

**Good**: `lima-san-isidro-north-2026`, `berlin-kreuzberg-mesh`, `nairobi-kibera-1`

**Bad**: `community-1`, `my-island`, `test`

---

## 12. First Boot and Bootstrap

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

## 13. Monitoring Mesh Networks

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

# Island status
bgpx-cli domains island-status --island-id your-island-name

# Bridge pair health
bgpx-cli domains bridges --from clearnet --to "mesh:your-island-name"

# Test cross-domain connectivity
bgpx-cli domains test --from clearnet --to "mesh:your-island-name"

# Watch for bridge events
bgpx-cli events watch --filter domain_bridge,mesh_island
```

---

## 14. Troubleshooting

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

**Domain bridge unavailable**:
```bash
# Check bridge records in DHT
bgpx-cli domains bridges --from clearnet --to "mesh:your-island-name"

# Verify bridge node is online
bgpx-cli nodes ping <bridge_node_id>

# Check bridge node advertisement
bgpx-cli nodes show <bridge_node_id> --advertisement
# Verify bridge_capable = true and bridges array populated
```
