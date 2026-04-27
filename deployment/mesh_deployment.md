# BGP-X Mesh Deployment Guide

**Version**: 0.1.0-draft

---

## 1. Mesh Island Concepts

A mesh island is a named collection of BGP-X nodes communicating via radio transport. It operates as a first-class routing domain — discoverable from the unified DHT, reachable from clearnet clients without special hardware, and independently functional when offline.

Every mesh island needs:
- An `island_id` (descriptive, globally unique string)
- At least 3 nodes for useful routing
- At least 1 gateway/domain bridge node with internet connectivity for cross-domain access

---

## 2. WiFi 802.11s Mesh Deployment

### Hardware Requirements

| Component | Recommended | Minimum |
|---|---|---|
| Router | GL.iNet GL-MT3000 or MT7981B | OpenWrt-compatible with 802.11 radio |
| RAM | 512 MB | 256 MB |
| Storage | 8 GB | 128 MB NAND |
| Radio | WiFi 6 (802.11ax) | WiFi 4 (802.11n) |

### Configuration

```bash
# Enable 802.11s on the router
uci set wireless.radio0.disabled=0
uci set wireless.mesh.network=mesh
uci set wireless.mesh.mode=mesh
uci set wireless.mesh.mesh_id=bgpx-your-island-name
uci set wireless.mesh.encryption=none
uci commit wireless
wifi reload
```

```toml
# /etc/bgpx/node.toml
[node]
roles = ["relay", "entry", "discovery"]

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"
```

---

## 3. LoRa Mesh Deployment

### Hardware

**Recommended**: GL.iNet GL-MT3000 + Heltec LoRa32 v3 (USB or SPI) or RAK WisBlock 4631.

**Frequency**: 868 MHz (EU), 915 MHz (US/AU/NZ), 923 MHz (AS2), 470 MHz (CN).

**EU Duty Cycle**: maximum 1% → ~36 seconds transmit per hour at SF7.

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["lora"]
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0
lora_spreading_factor = 7     # 7=fast/short range; 12=slow/long range
lora_bandwidth_khz = 125
lora_coding_rate = 5
lora_tx_power_dbm = 14        # Respect regulatory limits
```

---

## 4. Domain Bridge Node Deployment

A domain bridge node connects two routing domains. It is the enabling component for cross-domain routing.

### Minimal Gateway Node (Clearnet + WiFi Mesh)

```toml
[node]
roles = ["relay", "entry", "discovery"]

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
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"

[domain_bridge]
enabled = true

[mesh_island]
auto_publish_advertisement = true
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
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true
```

---

## 5. Multiple Bridge Nodes Per Island (IMPORTANT)

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
```

---

## 6. Clearnet User Accessing Mesh Service

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

## 7. Island Naming Best Practices

Island IDs should be:
- Descriptive and geographically specific (reduces collision probability)
- Lowercase with hyphens
- Short enough to be human-readable

Good: `lima-san-isidro-north-2026`, `berlin-kreuzberg-mesh`, `nairobi-kibera-1`

Bad: `community-1`, `my-island`, `test`

---

## 8. Monitoring Mesh Island Health

```bash
# Island status
bgpx-cli domains island-status --island-id your-island-name

# Bridge pair health
bgpx-cli domains bridges --from clearnet --to "mesh:your-island-name"

# Test cross-domain connectivity
bgpx-cli domains test --from clearnet --to "mesh:your-island-name"

# Watch for bridge events
bgpx-cli events watch --filter domain_bridge,mesh_island
```
