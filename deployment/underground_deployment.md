# Underground Deployent

**Version**: 0.1.0-draft

---

## 1. Overview

Underground deployment places BGP-X nodes in tunnels, subway systems, underground facilities, mines, and semi-subterranean structures (bunkers, basements, underground parking). Radio propagation in underground environments differs fundamentally from open-air deployment — LoRa and WiFi signals attenuate rapidly around tunnel curves and through dense rock, requiring careful repeater placement.

**Hardware applicable to this deployment**:
- **BGP-X Router v1** (Outdoor IP67 carrier): Full routing capability; gateway node for mesh-to-clearnet bridging at portal; solar/mast-mountable.
- **BGP-X Node v1**: Primary hardware for underground relay nodes; low power, compact IP67 form factor; PoE or battery powered.
- **BGP-X Client Node** (Tier 2): Low-power endpoint for miners, field workers, or sensors; battery-powered handhelds.

Underground BGP-X nodes provide privacy infrastructure for workers, public transit users, and secure facilities where surface RF signals cannot reach.

---

## 2. RF Propagation Underground

Radio propagation in underground environments is fundamentally different from surface conditions due to waveguide effects, multipath interference, and absorption by surrounding material.

### 2.1 Propagation Modes

**Tunnel Waveguide Effect**: Long tunnels act as waveguides for radio waves. Modes that align with tunnel geometry propagate significantly further than free space propagation.

**Multi-path**: Underground environments create severe multi-path reflections from walls, rails, and equipment. WiFi 802.11s with MIMO helps mitigate; LoRa's spreading factors provide robustness.

**Absorption**: Rock, soil, and concrete absorb RF energy. Higher frequencies attenuate faster. LoRa (868 MHz / 915 MHz) penetrates better than WiFi 2.4/5 GHz through solid obstacles.

### 2.2 Range Expectations by Environment

| Environment | WiFi 802.11s Range | LoRa Range | Notes |
|---|---|---|---|
| Open tunnel (concrete) | 100-400m | 300-800m | Tunnel acts as waveguide |
| Mine gallery (rock) | 20-100m | 50-500m | Irregular surfaces, high absorption |
| Basement (residential) | 10-50m (floor penetration) | 30-200m | Signal must penetrate concrete floors |
| Transit tunnel (steel rails) | 15-60m | 50-300m | Metal infrastructure reflects RF |
| Subway platform | 30-100m | 100-300m | Open space, good propagation |

### 2.3 Attenuation Factors

| Obstruction | WiFi 2.4 GHz | WiFi 5 GHz | LoRa 868/915 MHz |
|---|---|---|---|
| Concrete wall | 10-15 dB | 15-25 dB | 5-10 dB |
| Rock wall | 15-30 dB | 20-40 dB | 10-25 dB |
| 90° bend | 10-20 dB | 15-25 dB | 15-25 dB |
| Train car (metal) | 20-40 dB | 30-50 dB | 15-30 dB |
| Water/soil fill | 20-40 dB/m | 30-60 dB/m | 10-20 dB/m |

LoRa's lower frequency provides better penetration through dense material, making it preferred for deep underground or high-attenuation environments.

---

## 3. Node Placement Strategies

### 3.1 Leaky Feeder (Radiating Cable) Infrastructure

Traditional approach for long tunnels: coaxial radiating cable run the length of the tunnel with periodic antenna couplers.

```
Node A (portal) ── Radiating coax ── Node B (midpoint) ── Radiating coax ── Node C (end)
(WAN connected)      (100-200m coverage)                         (end of tunnel)
```

**BGP-X adaptation**: Nodes connect to the radiating cable via N-type connectors at intervals. Coverage is continuous along the cable length. Use BGP-X Node v1 units at each connection point.

**Advantage**: Continuous coverage without placing nodes every 100-200m. Lower node count.

**Disadvantage**: Requires infrastructure installation. Higher upfront cost. Not suitable for temporary deployments.

### 3.2 Node Chain (WiFi Relay)

For WiFi 802.11s mesh in tunnels without leaky feeder infrastructure:

```
Node A ──[WiFi mesh]──── Node B ──[WiFi mesh]──── Node C ──[WiFi mesh]──── Node D
(portal)    100-200m          100-200m              100-200m               (deep)
```

Each node provides coverage for ~100-200m in each direction along the tunnel. Nodes relay traffic along the chain. Use BGP-X Node v1 for all relay nodes; BGP-X Router v1 at the portal for WAN connection.

### 3.3 LoRa for Long-Range Underground

LoRa propagates further underground than WiFi, especially in open tunnels. Use for:
- DHT synchronization
- Low-bandwidth messaging
- Sensor telemetry
- Emergency alert channels

```
Node A (portal) ──[LoRa]──── 500m ──── Node B (midpoint) ──[LoRa]──── 500m ──── Node C (deep)
(WAN connected)                           (relay only)                     (end of tunnel)
         ↕
   Workers with LoRa
   handheld devices
```

Workers carry Meshtastic-compatible handhelds or BGP-X Client Nodes; BGP-X relay nodes send/receive LoRa traffic.

### 3.4 Node Spacing Guidance

| Environment | WiFi Relay Spacing | LoRa Relay Spacing |
|---|---|---|
| Straight tunnel (≥3m diameter) | 200m | 500m |
| Curved tunnel | 50-100m (node at each bend) | 300m |
| Irregular mine gallery | 50-100m | 100-300m |
| Subway platform | 30-50m | N/A (use WiFi) |

Place nodes at every significant bend or junction.

---

## 4. Deployment Scenarios

### 4.1 Subway / Metro System

BGP-X nodes deployed in subway tunnels and stations create a privacy-protected mesh for transit system communications and public passenger access.

```
[Surface gateway: BGP-X Router v1 at station entrance]
  WAN: fiber/cellular clearnet
  Mesh: WiFi 802.11s downward into station
         │
         ▼
[Station level: BGP-X Node v1 at platform]
  Relay node: no WAN
  Mesh: WiFi along tunnel + WiFi to surface
         │ WiFi down tunnel
         ▼
[Tunnel relay node 1: BGP-X Node v1, 200m in]
         │ WiFi
         ▼
[Tunnel relay node 2: BGP-X Node v1, 400m in]
         │
... (relay every 150-250m until next station)
         │
         ▼
[Next station: BGP-X Node v1]
  Connected to next station's gateway
```

**Power**: Underground nodes use PoE from station electrical infrastructure. No solar possible underground. BGP-X Node v1 PoE 802.3af (15W) input is sufficient.

**Passenger access**: Passengers on platforms connect to the station node's WiFi LAN. Their traffic routes encrypted through the tunnel relay chain to the surface gateway, then into the BGP-X overlay.

**Tunnel node considerations**:
- Must withstand vibration from train passage
- Enclosure IP67 required (moisture, dust from trains)
- Maintenance access during off-hours only

### 4.2 Mine Deployment

Mines require BGP-X for private communications between miners, equipment monitoring, and surface connectivity.

```
[Surface: BGP-X Router v1]
  WAN: satellite (Starlink) or cellular or fiber
  Mesh: LoRa + WiFi
         │ LoRa + WiFi down main shaft
         ▼
[Shaft relay nodes: BGP-X Node v1, every 300m]
         │
         ▼
[Level tunnels: BGP-X Node v1 at each tunnel junction]
         │ WiFi along tunnel level
         ▼
[Working face relay nodes: BGP-X Node v1, battery powered]
```

**Power in mines**:
- Main tunnels: electrical power via mine electrical system (PoE)
- Working face nodes: battery-powered (BGP-X Node v1 with LiFePO4 battery). Replace or recharge during shift changes.

**Hazardous environments (CRITICAL)**:

Class I Division 1 or Zone 0 (explosive atmosphere) mines require **intrinsically safe (IS) certified electronics**.

- BGP-X hardware in standard IP67 enclosures is **NOT rated IS/ATEX**.
- For hazardous mine atmospheres, BGP-X hardware must be placed in:
  - Intrinsically safe barriers
  - Purged enclosures (positive pressure with inert gas)
  - Remote antenna placements (electronics in safe area, antenna in hazardous zone)
- **ATEX-certified enclosures**: Required for Zone 1/2 (gas) or Zone 21/22 (dust).
- **Certifications to verify**: ATEX (EU), IECEx (international), MSHA (US mines), CSA/UL (North American).
- **Operator responsibility**: Selecting and maintaining appropriate certification for the specific hazardous area classification.

**Non-hazardous mine areas** (ventilated zones, control rooms): Standard IP67 enclosures suitable.

### 4.3 Underground Bunker / Shelter

BGP-X deployment in underground shelters or bunkers for emergency communications.

```
[Outside: LoRa antenna on surface (short cable run to indoor BGP-X unit)]
         │ LoRa antenna cable (LMR-400)
         ▼
[Bunker entrance: BGP-X Node v1]
  WAN: surface antenna via cable (satellite or LoRa to surface gateway)
  Mesh: WiFi 802.11s inside bunker
         │ WiFi throughout bunker
         ▼
[Bunker interior nodes: additional BGP-X Router v1 or Node v1 units]
```

**Penetrating the bunker wall**:
- Antenna cable (LMR-400) through the wall via a coaxial bulkhead connector.
- Proper weatherproofing/sealing on both sides.
- Do not break the RF path — use low-loss cable and minimize bends through the wall penetration.

**Power**: Bunker typically has backup generator or UPS. Ensure BGP-X nodes connected to protected power circuits.

### 4.4 Basement / Parking Structure

For multi-floor underground parking or basement deployments:

```
[Ground floor: BGP-X Router v1 (WAN via building fiber)]
         │ WiFi mesh downward
         ▼
[Basement level 1: BGP-X Node v1]
         │ WiFi mesh downward
         ▼
[Basement level 2: BGP-X Node v1]
         │ WiFi mesh downward
         ▼
[Basement level 3: BGP-X Node v1]
```

**Between-floor propagation**:
- Concrete floor slabs: 15-30 dB attenuation at 5 GHz, 10-20 dB at 2.4 GHz.
- Use 2.4 GHz for inter-floor links; 5 GHz for same-floor high-bandwidth links.
- LoRa (if using) penetrates multiple floors better than WiFi.

---

## 5. Hardware for Underground

### 5.1 Node Selection

| Environment | Recommended Hardware | Enclosure |
|---|---|---|
| Subway tunnel | BGP-X Node v1 | IP67 |
| Mine (non-hazardous) | BGP-X Node v1 | IP67 |
| Mine (hazardous) | BGP-X Node v1 inside IS-rated enclosure | ATEX/IECEx rated |
| Bunker | BGP-X Router v1 (portal) + Node v1 (interior) | IP67 |
| Basement/parking | BGP-X Router v1 (surface) + Node v1 (underground) | IP20 or IP67 |

### 5.2 LoRa Handhelds for Workers

Workers in underground environments can carry BGP-X Client Nodes (Tier 2) or Meshtastic-compatible handhelds:

| Device | Role | Battery | Certification |
|---|---|---|---|
| LILYGO T-Echo | LoRa client for individual worker | Battery (days) | Not IS-rated |
| Custom IS-rated handheld | Hazardous mine environment | Battery | IS/ATEX required |

**Non-hazardous areas**: Standard Meshtastic handhelds (LILYGO T-Beam, T-Echo, Heltec V3) suitable.

**Hazardous areas**: Must use IS-certified LoRa handhelds (mining-specific hardware required).

### 5.3 Sensors and IoT

Low-power underground sensors can use BGP-X Client Nodes for LoRa telemetry:

- Air quality sensors (methane, CO2)
- Structural integrity sensors (vibration, displacement)
- Personnel tracking (wearable LoRa beacons)
- Equipment monitoring (temperature, runtime)

---

## 6. Antenna Selection for Underground

### 6.1 Directional Panel Antennas (Tunnels)

For straight tunnel sections: directional panel antennas pointing along the tunnel axis concentrate energy in the direction of travel.

| Antenna Type | Gain | Beam Width | Use Case |
|---|---|---|---|
| 7 dBi patch panel | 60° | Short relay, slight bend tolerance | Station-to-first relay |
| 10 dBi flat panel | 35° | Straight tunnel relay | Long straight sections |
| 14 dBi yagi | 25° | Maximum range in dead-straight tunnels | Mine main shaft |

**Note on LoRa directional antennas underground**: SX1262 supports directional antennas. Configure the BGP-X LoRa transport to match the antenna gain in software (`lora_antenna_gain_dbi` config parameter) for correct RSSI-based power management.

### 6.2 Omnidirectional Antennas (Junctions, Stations)

At tunnel junctions, cross-passages, and station platforms: omnidirectional antennas serve peers in multiple directions:

- 3-5 dBi ceiling-mount omnidirectional for WiFi in large open areas
- Rubber duck omnidirectional for LoRa in junction areas

---

## 7. Power Infrastructure Underground

### 7.1 PoE Distribution

Underground BGP-X nodes powered via PoE from building or mine electrical infrastructure:

```
Surface or main electrical room:
  PoE injectors / managed PoE switch (e.g., Ubiquiti USW-Pro-48-PoE)
         │
         │ Cat6/Cat6A cable (PoE up to 100m per run per standard)
         │ For longer runs: use PoE extenders or fiber + remote PoE injector
         │
         ▼
BGP-X Node v1 (802.3af or 802.3at PoE input)
```

**PoE run limits**: Standard PoE is specified for 100m Cat6 cable runs. Beyond 100m:
- Use PoE extenders (inject PoE partway along long cable runs)
- Use fiber + media converter + local PoE injector at node location
- For mine shafts (>100m deep): fiber backbone with local PoE injectors at each level

### 7.2 Battery Backup

For underground nodes in locations where power interruption is a risk (mine power outages, emergency scenarios):

```
PoE input → UPS (sealed lead-acid or LiFePO4) → BGP-X Node v1

Battery capacity:
  BGP-X Node v1 draws ~3W at PoE
  7Ah sealed lead-acid at 12V = 84 Wh = 84 Wh ÷ 3W = 28 hours backup
  Sufficient for overnight operation through a power outage
```

**Mine emergency power**: In mines, BGP-X nodes should be connected to the mine's emergency power circuit (if available) or have local battery backup sufficient for emergency communication duration.

---

## 8. BGP-X Configuration for Underground

Underground chains may have higher relay latency and intermittent connectivity for LoRa links.

```toml
[node]
roles = ["relay", "discovery"]

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]
# WiFi for local coverage; LoRa for longer tunnel reach

beacon_interval_seconds = 30

[sessions]
# Underground chains may have high relay latency
session_idle_timeout_seconds = 120  # Longer timeout for deep underground

[store_and_forward]
enabled = true   # Important for intermittent LoRa links in complex tunnels
buffer_size_kb = 1024
packet_ttl_seconds = 3600

[advertisement]
link_quality_profiles.wifi_mesh.latency_class = 1  # Medium latency expected
link_quality_profiles.lora.latency_class = 2        # High latency expected
```

---

## 9. Cross-Domain Routing Underground

Underground BGP-X deployments typically form a **mesh island** that connects to the surface via a gateway node at the surface entry point.

```
Surface gateway: BGP-X Router v1
  WAN: fiber/cellular/satellite (clearnet domain)
  Bridge: clearnet ↔ underground mesh island
         │
         │ WiFi 802.11s or LoRa relay chain underground
         │ (all encrypted; onion routing preserved underground)
         ▼
Underground relay nodes: BGP-X Node v1 (relay only)
         │
         ▼
Underground users / devices (on LAN of any underground BGP-X Router v1)
```

**Underground users' traffic**:
- Travels encrypted via BGP-X overlay through underground relay chain
- Exits at surface gateway
- Routes through clearnet BGP-X relay network to destination
- Surface gateway sees only adjacent relay IP addresses — not underground user identities

**Clearnet users accessing underground services**:
- Path construction discovers surface gateway as domain bridge
- Traffic enters underground mesh island via surface gateway
- Underground service remains accessible from clearnet via standard cross-domain path

---

## 10. Safety Considerations

### 10.1 Emergency Communications Priority

Underground BGP-X deployments in mines and public infrastructure should configure emergency channel priority:

```toml
[emergency]
enabled = true
emergency_channel_id = "emergency-band"

# Emergency streams bypass normal routing constraints
# Emergency stream priority = 0 (highest)
# Emergency streams not subject to congestion control throttling
emergency_stream_always_forward = true
```

Emergency communications configured with stream priority 0 are forwarded before all other traffic even under congestion conditions.

### 10.2 Failure Planning

Underground relay nodes are difficult to physically access for maintenance. Plan for:

- **PoE-controlled remote reboot**: managed PoE switch with per-port power control
- **Remote access via BGP-X control socket**: SSH tunnel via overlay if necessary
- **Spare nodes staged at surface**: same configuration; hot-swap capable
- **NODE_WITHDRAW published automatically** on power loss (graceful vs. ungraceful shutdown)
- **Automatic DHT re-join** on power restore (no operator intervention needed)

---

## 11. Deployment Checklist

### Pre-Installation
- [ ] RF propagation survey: walk tunnel with handheld LoRa/WiFi test device; identify relay positions
- [ ] Power infrastructure survey: identify available PoE drop points or electrical outlets
- [ ] Identify all bends, junctions, and level changes requiring relay nodes
- [ ] Identify any explosive atmosphere zones (mine); plan IS-rated enclosures or remote antenna placement if required
- [ ] Determine if leaky feeder infrastructure is already present (reuse if possible)
- [ ] Surface gateway location selected; WAN available at surface

### Installation
- [ ] Surface gateway installed and online; verified connected to clearnet BGP-X network
- [ ] Underground relay nodes installed in sequence (start at surface; verify each before installing next)
- [ ] Each node verified reachable from surface before installing next underground node
- [ ] PoE cable runs secured; no pinch points; cable protected in conduit where mechanical damage possible
- [ ] All RF connectors protected (self-amalgamating tape underground — moisture is still a concern)
- [ ] Node enclosures closed and IP67 seals verified
- [ ] For hazardous areas: IS-rated enclosures or barriers installed; certification verified

### Commissioning
- [ ] Full relay chain test: from deepest node to surface, via `bgpx-cli node ping` chain
- [ ] Cross-domain path test: from surface clearnet client, through gateway, to deepest underground node
- [ ] Emergency priority test: emergency stream established; verifies priority handling
- [ ] PoE remote reboot test: verify each node can be power-cycled via managed switch
- [ ] Node recovers automatically after reboot: `bgpx-cli node show` from above ground shows node online within 60 seconds of reboot
- [ ] Battery backup test (if applicable): simulate power interruption; verify node continues operation on backup power
