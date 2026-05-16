# Vehicle Deployment 

**Version**: 0.1.0-draft

---

## 1. Overview

Vehicle-mounted BGP-X nodes serve as mobile relay infrastructure, extending mesh coverage and synchronizing DHT records as the vehicle travels. A bus, truck, emergency vehicle, or even a private car carrying a BGP-X Router v1 or Node v1 creates a moving relay that:

- Dynamically joins mesh islands along its route
- Extends LoRa coverage to areas without permanent relay nodes
- Synchronizes DHT records between mesh island clusters
- Bridges isolated mesh islands when passing through their coverage areas
- Provides mobile access points for BGP-X users aboard the vehicle

**Hardware Applicable to This Deployment**:

| Hardware | Role in Vehicle | Notes |
|---|---|---|
| **BGP-X Router v1** | Primary vehicle node; mobile gateway; passenger hotspot | Outdoor IP67 carrier for exterior mounting; PoE powered |
| **BGP-X Node v1** | Compact relay; domain bridge; mobile relay | Lower power; solar/battery compatible |
| **BGP-X Client Node (Tier 2)** | Personal mesh endpoint (bicycle, motorcycle) | Battery powered; low cost; endpoint only |
| **BGP-X Adapter (Tier 3)** | USB LoRa modem for laptop/desktop | Turns any computer into mesh client |

---

## 2. Use Cases

### 2.1 Public Transit (Bus, Train, Ferry)

A bus route passing through rural areas creates a periodic relay that:

- Extends LoRa coverage to areas without permanent relay nodes
- Synchronizes DHT records between mesh island clusters that cannot otherwise communicate
- Provides mobile access point for BGP-X users aboard the vehicle
- Bridges isolated mesh islands when the bus passes through each island's coverage area

**Configuration**: BGP-X Router v1 (if passenger WiFi is desired) or BGP-X Node v1 (relay only). LoRa antenna on roof; WiFi antennas on roof or integrated in interior. Powered from vehicle 12V/24V electrical system.

### 2.2 Emergency and Response Vehicles

Emergency vehicles (ambulances, fire trucks, police) carrying BGP-X hardware expand mesh coverage in disaster or emergency areas where infrastructure may be damaged.

**Configuration**: BGP-X Node v1 in relay mode or BGP-X Router v1 for crew communications. Rapid deployment: vehicle provides power; no configuration changes needed (pre-configured before deployment).

### 2.3 Commercial Trucks and Delivery Vehicles

Long-distance commercial vehicles traveling between cities create a moving relay backbone connecting rural mesh islands along routes. A single truck route covering 500 km creates a dynamic relay path that periodically connects all mesh islands along the route.

### 2.4 Convoy Communications

Multiple vehicles form a moving mesh network:

```
Vehicle A ──[WiFi mesh]──── Vehicle B ──[WiFi mesh]──── Vehicle C
(Lead, has cellular)         (Relay)                     (Tail)
```

**Topology**: WiFi 802.11s with 5 GHz for inter-vehicle high bandwidth; LoRa for long-range fallback when vehicles separate.

**Gateway vehicle**: Lead vehicle with cellular data acts as internet gateway for the convoy mesh.

### 2.5 Private Vehicles

Individual users with BGP-X Router v1 installed become mobile relays whenever driving. No special configuration needed — the router's outdoor IP67 carrier mounts under the vehicle, on the roof rack, or in the trunk with external antennas.

### 2.6 Survey / Journalism

Secure communications while mobile in the field. Vehicle acts as a secure relay for field researchers, journalists, or NGO staff operating in areas with unreliable or hostile infrastructure.

---

## 3. Hardware Selection and Mounting

### 3.1 BGP-X Router v1 / Node v1 Mounting

**Permanent installation**:

- Node or Router secured in trunk, under-seat compartment, or electronics bay
- Antennas mounted on vehicle roof using NMO mount or L-bracket
- Cable routed through existing grommets or new weatherproof grommets
- Power connected to vehicle electrical system

**Temporary installation** (rental/borrowed vehicles):

- Node or Router in protective case on seat or trunk
- Magnetic mount antennas on roof
- Power via 12V accessory outlet (cigarette lighter adapter)
- Window pass-through for antenna cables with foam weather seal

### 3.2 Antenna Mounting Options

**Permanent Roof Mount (Recommended for owned vehicles)**:

- **NMO mount**: 3/4" hole through roof; standard mobile radio installation
- **LoRa antenna**: 5/8" wave whip (5-6 dBi) on NMO mount
- **WiFi antennas**: Dual NMO mounts for MIMO, or single dual-band panel antenna
- **Cable routing**: Through existing cable grommets (often near roof pillars)
- **Sealing**: Waterproof boot or silicone sealant around penetration

**Temporary Magnetic Mount (Rental / Temporary)**:

- Magnetic base LoRa antenna (NMO-to-SMA magnetic base) on steel roof
- Magnetic base WiFi antennas
- Cable enters via window edge (foam seal) or door gap
- **Safety note**: At highway speeds (100+ km/h), verify antenna stability. Some magnetic bases are insufficient for crosswinds — use safety tether to roof rack or window frame.

**Trailer-Mounted Antenna (Heavy Trucks)**:

- Antenna mounted on trailer roof (maximum height for range)
- BGP-X hardware in cab powered from cab electrical system
- Long cable run: use LMR-400 to minimize loss over 10-15m

**Optimal Placement**:

- WiFi antenna: roof center for omnidirectional coverage
- LoRa antenna: roof or trunk lid; keep away from other electronics

---

## 4. Vehicle Power Systems

### 4.1 12V Vehicle Electrical System (Cars, Vans, Light Trucks)

```
12V vehicle battery / alternator
      │
      ▼
Fused tap: 10A blade fuse inline
      │
      ├── BGP-X Node v1: 12V DC barrel input (via voltage regulator: 12V → 5V, 3A)
      │   Power consumption: ~3W = 250 mA at 12V
      │
      └── BGP-X Router v1 via PoE injector (802.3af): 12V input → PoE output → Router
          Power consumption: ~15W = 1.25 A at 12V
```

**Power consumption**:

| Device | Power Draw at 12V | Load on Vehicle |
|---|---|---|
| BGP-X Node v1 | 250-350 mA | Negligible |
| BGP-X Router v1 | 1.0-1.5 A | Negligible |

**Recommended connection**: After ignition switch fuse line so device powers off with vehicle (prevents battery drain when parked). Alternative: battery isolator for 24/7 operation.

### 4.2 24V Vehicle Electrical System (Heavy Trucks, Buses)

```
24V vehicle system
      │
      ▼
24V to 12V step-down converter (or 24V to 5V for Node v1 directly)
      │
      ▼
BGP-X hardware
```

24V systems require a step-down converter (DC-DC buck). Recommended: 5A rating for headroom.

### 4.3 Shore Power (Ferry, Train, Parked Vehicles)

Vehicles with shore power access (ferries at dock, trains at station, parked trucks with APU):

```
Shore power (AC 110V or 230V)
      │
      ▼
AC-to-DC adapter or PoE injector with AC input
      │
      ▼
BGP-X hardware
```

Standard PoE injectors with AC input work directly.

### 4.4 Vehicle-Integrated UPS

For continuous operation through engine starts (which can cause 12V dips):

```
12V vehicle → UPS battery (small sealed lead-acid or LiFePO4) → BGP-X hardware
```

A 7Ah SLA battery provides ~20 minutes of BGP-X operation during engine start or brief power interruptions. Recommended for emergency vehicles.

---

## 5. Vehicle-Specific Configuration

### 5.1 Mobile Relay (No WAN — Mesh Only)

Vehicle participates in mesh islands along its route. Dynamically joins whichever islands are in LoRa range.

```toml
[node]
role = "relay"
mobile = true              # CRITICAL: declares this is a mobile node

[[routing_domains]]
domain_type = "mesh"
# No island_id specified: join any island whose MESH_BEACON is received
island_auto_join = true
transports = ["lora", "wifi-mesh"]

[performance]
session_max_duration_secs = 300    # 5-minute sessions; prevents stale paths
keepalive_interval_seconds = 10     # More frequent for mobile churn

[advertisement]
# Geographic plausibility: mobile nodes have highly variable RTT
# Should be disabled to prevent false reputation penalties

[reputation]
geo_plausibility_enabled = false
```

**Mobile Node Advertisement Effects**:

When `mobile = true` is declared:

- KEEPALIVE_TIMEOUT does not generate a reputation penalty
- Path construction algorithms prefer stable fixed nodes for long-lived paths
- This node is still used as a relay when it is the best option available
- Advertisement geographic location is not used for geo plausibility scoring

### 5.2 Mobile Gateway (With Cellular/Satellite WAN)

Vehicle has cellular data (LTE/5G) or satellite WAN. Functions as moving internet gateway for mesh islands it passes through.

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true
mobile = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[clearnet_transport]
wan_type = "cellular"          # or "satellite" for Starlink on vehicle
interface = "wwan0"            # LTE modem interface name (varies)

[[routing_domains]]
domain_type = "mesh"
island_auto_join = true
transports = ["lora", "wifi-mesh"]
```

### 5.3 Passenger Hotspot (BGP-X Router v1 in Vehicle)

BGP-X Router v1 provides WiFi hotspot for passengers while serving as a relay:

```toml
# Standard BGP-X Router v1 config
# WAN: vehicle cellular/satellite
# LAN: passenger WiFi (standard 802.11 AP on 2.4/5 GHz)
# Mesh: 802.11s on second radio channel (or LoRa)

[node]
role = "relay,domain_bridge"
mobile = true
```

Passengers connect to vehicle WiFi (standard connection, no BGP-X software required). All traffic is protected by router's BGP-X overlay.

### 5.4 Convoy Configuration (Gateway Vehicle)

Lead vehicle in a convoy acts as gateway for other vehicles:

```toml
[node]
role = "relay,entry,exit,discovery"
mobile = true

[advertisement]
is_gateway = true
gateway_for_mesh = ["convoy-mesh-alpha"]

[exit_policy]
logging_policy = "none"
dns_mode = "doh"
```

---

## 6. Motion Considerations

### 6.1 LoRa in Motion

LoRa mesh operation while the vehicle is moving:

- **Doppler shift**: At vehicle speeds (< 200 km/h), Doppler shift on 868/915 MHz LoRa is negligible (< 1 ppm). No impact on demodulation.
- **RSSI variation**: Signal strength fluctuates rapidly as vehicle moves relative to ground nodes. BGP-X's path selection is robust — when RSSI drops below usable threshold, path is rebuilt or session terminates gracefully.
- **Session duration**: Sessions with ground nodes should be kept short (`session_max_duration_secs = 300`) since vehicle will move out of range. Long-lived sessions that time out due to motion generate reputation events (KEEPALIVE_TIMEOUT); this is expected and not penalized for mobile nodes.

### 6.2 WiFi Mesh in Motion (Convoy)

- **Inter-vehicle links**: Stable at highway speeds within line-of-sight (< 100m between vehicles)
- **Topology changes**: Frequent as vehicles join/leave convoy. BGP-X's 802.11s mesh layer handles peer churn automatically.
- **Latency**: Inter-vehicle WiFi mesh latency is low (< 10ms), supporting higher-throughput traffic between convoy vehicles.

---

## 7. Specialized Vehicle Deployments

### 7.1 Motorcycle

- **Power**: 12V bike battery (sufficient capacity)
- **Hardware**: BGP-X Node v1 or Client Node in tank bag or tail pack
- **Antenna**: Small magnetic mount or bar-end mount
- **LoRa**: SX1262 USB module in weatherproof pouch

### 7.2 Bicycle

- **Power**: USB powerbank (20,000 mAh)
- **Hardware**: BGP-X Client Node (Tier 2) — GL.iNet Mango Mini or LILYGO T3S3
- **Antenna**: Handlebar mount for WiFi; LoRa module in frame bag
- **Estimated runtime**: 8-12 hours per 20,000 mAh bank

**Configuration for low-power Client Node**:

```toml
[node]
role = "client"              # Endpoint only
mobile = true

[mesh]
enabled = true
transports = ["lora"]

[power]
power_save_mode = true
beacon_interval_seconds = 60 # Less frequent beacons for battery life
```

---

## 8. Weather Protection

For nodes exposed to weather in vehicle/outdoor positions:

- **Waterproof bag or pouch**: For nodes not rated IP67 (most Tier 2/3 devices)
- **Cable entry**: Use grommet with silicone sealant
- **Condensation**: Desiccant sachet inside pouch
- **Temperature**: Most devices rated 0-40°C operating; use insulated pouch below 0°C

---

## 9. Satellite WAN on Vehicles

BGP-X Router v1 and Node v1 support **satellite WAN connections** via USB satellite modems (Starlink, Iridium, Inmarsat).

**Starlink Gen 3 on Vehicle**:

- USB port on terminal presents as USB Ethernet adapter
- BGP-X daemon detects via USB vendor ID
- Treated as clearnet domain with LEO latency class (20-40ms)

**Configuration**:

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_class = "leo"       # or "geo" for Inmarsat/iridium

[link_quality_profiles]
satellite = { latency_class = "satellite-leo", bandwidth_class = "high" }
```

**Satellite is clearnet**: From BGP-X's perspective, satellite internet is clearnet domain (0x00000001), NOT a separate routing domain.

---

## 10. Vehicle Deployment Checklist

### Installation

- [ ] Antenna mounted (roof or temporary magnetic); cables routed indoors
- [ ] BGP-X hardware mounted securely (no rattling; cannot become projectile)
- [ ] Power connection: fused tap from vehicle electrical system
- [ ] Power consumption verified: <5% of vehicle alternator capacity
- [ ] Cables secured along vehicle interior; no pinch points
- [ ] BGP-X hardware powered on; daemon starts correctly
- [ ] `bgpx-cli node identity` — confirm NodeID
- [ ] `bgpx-cli domains list` — confirm mesh domain active

### Configuration Verification

- [ ] `mobile = true` declared in node config
- [ ] `island_auto_join = true` configured (or specific island_id)
- [ ] `session_max_duration_secs` set appropriately for vehicle speed
- [ ] `geo_plausibility_enabled = false` for mobile nodes
- [ ] LoRa RSSI from roadside test node: acceptable when parked; fades as vehicle moves (expected)

### Operational Testing

- [ ] Drive route segment; verify sessions established with ≥3 mesh nodes
- [ ] Verify sessions terminate gracefully (not KEEPALIVE_TIMEOUT) when vehicle moves out of range
- [ ] For passenger hotspot: verify passenger WiFi works; verify overlay active
- [ ] For convoy: verify inter-vehicle WiFi mesh connectivity while moving
- [ ] For mobile gateway: verify clearnet access works through vehicle WAN

---

**End of Document**
