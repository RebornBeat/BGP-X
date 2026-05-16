# BGP-X Mast and Tower Deployment

**Version**: 0.1.0-draft

---

## 1. Overview

Mast and tower deployments elevate BGP-X hardware to maximize LoRa and WiFi mesh coverage radius. A BGP-X Router v1 or BGP-X Node v1 at 30 meters elevation achieves 10-30 km LoRa line-of-sight range on clear terrain — sufficient to cover an entire district or small town with a single node.

This document covers fixed elevated deployments: rooftop mounts, purpose-built masts, existing communication towers, and building facade installations.

**Hardware applicable to this deployment**:
- **BGP-X Router v1** (outdoor IP67 carrier): for users who want one device that is both their home router and an elevated relay. WAN runs up the mast via PoE cable; LAN devices connect via WiFi inside the building.
- **BGP-X Node v1**: for community contributors deploying a dedicated elevated relay or gateway node. No home router function; pure relay, range extension, or bridge operation. Solar-powered option eliminates need for PoE cable run.

Elevated relay nodes extend mesh coverage dramatically. A node at 30m height covers 50-100× the area of a ground-level node.

Typical applications:
- Rooftop relay connecting neighborhood WiFi mesh islands
- LoRa mast covering rural settlement cluster
- Tower relay bridging two valleys
- Building rooftop serving as WiFi 802.11s backbone

---

## 2. Coverage vs. Height

### 2.1 LoRa Range by Elevation (SX1262, +14 dBm TX, SF9 BW125, 2 dBi antennas both sides)

| Elevation | Urban Range | Suburban Range | Rural / Clear Terrain |
|---|---|---|---|
| Ground (2m) | 0.5-1.5 km | 1-3 km | 2-5 km |
| Rooftop (10m) | 1-3 km | 3-8 km | 5-15 km |
| Mast (20m) | 2-5 km | 5-15 km | 8-25 km |
| Mast (30m) | 3-8 km | 8-20 km | 10-30 km |
| Tower (45m) | 5-12 km | 12-30 km | 15-45 km |
| Tower (60m+) | 8-20 km | 20-50 km | 20-60 km |

### 2.2 WiFi 802.11s Range by Elevation (MT7915, 5 GHz, directional antenna)

| Elevation | Effective Range |
|---|---|
| Ground (2m) | 50-200m |
| Rooftop (10m) | 200m-1km (LOS), 100-400m (obstructed) |
| Mast (30m) | 1-5 km (LOS with directional antenna) |

WiFi 802.11s range at elevation is primarily useful for connecting rooftop-to-rooftop in dense urban environments or for high-bandwidth backhaul between mast nodes.

### 2.3 Height Rules of Thumb

| Height | WiFi 802.11s Range (clear) | LoRa SF9 Range (clear) |
|---|---|---|
| Ground level (2m) | 100-300m | 2-5km |
| Rooftop (10-15m) | 500m-2km | 5-15km |
| Mast (30m) | 2-8km | 10-30km |
| Tower (60m+) | 5-20km | 20-60km |

---

## 3. Site Selection for Maximum Coverage

### 3.1 Line of Sight Assessment

Radio line of sight is THE most important factor in mesh range. A $25 device on a mast will outperform a $150 device on a desk.

```
Scenario A: Node on a desk inside a house
  Height: 1m
  Obstructions: Walls, furniture, neighbors' houses
  Range: 500m - 2km

Scenario B: Node on a roof (external antenna)
  Height: 8m
  Obstructions: None (antenna is above roofline)
  Range: 5km - 15km

Scenario C: Node on a mast/tower
  Height: 30m
  Obstructions: None
  Range: 20km - 50km

Scenario D: Node on a mountaintop
  Height: 500m AGL
  Range: 100km+
```

**The antenna height matters more than the hardware.**

### 3.2 Fresnel Zone Clearance

For reliable WiFi links, the first Fresnel zone must be clear of obstacles.

```python
# Fresnel zone clearance required for obstacle-free WiFi link
def fresnel_first_zone(distance_m, frequency_ghz):
    lambda_m = 0.3 / frequency_ghz
    return 8.656 * (lambda_m * distance_m) ** 0.5  # meters radius at midpoint

# Example: 2km link at 5.8 GHz
fresnel_first_zone(2000, 5.8)  # = 5.4m radius clearance needed at midpoint
# Antenna height must clear any obstacle + 5.4m for reliable link
```

### 3.3 Antenna Quality and Gain

| Antenna Type | Gain | Range Impact | Notes |
|--------------|------|--------------|-------|
| PCB antenna (on Node board) | -2 to 0 dBi | Poor | Convenience only |
| Small "candy bar" antenna | 2-3 dBi | Okay | Included with most devices |
| Fiberglass omni (30cm) | 5-8 dBi | Good | Standard outdoor |
| Fiberglass omni (1m) | 8-10 dBi | Excellent | Serious infrastructure |
| Directional Yagi | 10-15 dBi | Maximum | Point-to-point links only |

**The BGP-X Router v1 and Node v1 mandate external SMA connectors**. This allows any antenna. Use the best antenna practical for the deployment.

### 3.4 TX Power

| Device | LoRa Chip | Max TX Power | Notes |
|--------|-----------|--------------|-------|
| BGP-X Router v1 | SX1262 | 22 dBm (158mW) | Same as Node v1 |
| BGP-X Node v1 | SX1262 | 22 dBm (158mW) | Same as Router v1 |
| Meshtastic devices | SX1276/SX1262 | 17-22 dBm | Varies |
| Regulatory limit (US 915MHz) | — | 30 dBm (1W) | With spreading |
| Regulatory limit (EU 868MHz) | — | 14-27 dBm | Duty cycle limited |

**TX power is the same across BGP-X hardware.** Regulatory limits matter more than hardware differences.

---

## 4. Rooftop Mount (Most Common)

### 4.1 Site Requirements

- Roof access permission from building owner (written recommended)
- Structural assessment for wind load (consult engineer for masts >2m above roofline)
- Lightning protection if in high-lightning area or structure >30m above surrounding terrain
- Cable routing path from roof to indoor equipment location (BGP-X Router v1 only; Node v1 may be entirely rooftop-mounted)

### 4.2 Mount Types

**Non-penetrating roof mount**: weighted base with vertical mast. No holes in roof. Suitable for flat roofs. Maximum mast height: 3-4m above mount base (wind load limits). Ballast: 50-100 kg concrete or sandbags.

**J-mount (wall or parapet)**: L-shaped bracket bolted to parapet wall or building facade. Simple; 1-2m mast extension above parapet. Very common for residential rooftop deployments.

**Chimney mount**: stainless steel straps around chimney. No drilling. 1-2m mast above chimney top. Electrically isolated from chimney.

**Through-roof penetration mast**: base flange through roof membrane; requires professional waterproofing; allows taller masts (3-6m above roofline); most wind-resistant.

### 4.3 BGP-X Router v1 — Rooftop with Indoor LAN

```
Rooftop:
  [Outdoor IP67 carrier] ← LoRa antenna (SMA)
                         ← WiFi antennas (RP-SMA × 4)
  PoE: 802.3at, 30W
         │
         │ Cat6 cable (conduit recommended; weatherproof cable entry)
         │ (max 100m PoE cable run)
         │
Indoor:
  [PoE injector] ← [AC power]
         │
         └── [BGP-X Router v1 as WAN] ← [ISP modem/ONT]
         OR
         └── [Network switch] → [LAN devices]

Config:
  - WAN: ISP connection via indoor unit or PoE-injector-side network
  - Mesh: outdoor antenna → mesh island participation
  - LAN: indoor WiFi (standard 802.11 client AP) + wired LAN
```

Note: BGP-X Router v1 in outdoor IP67 carrier is a single unit — the same device that is the router is also the elevated relay. The PoE cable carries both power and data between the outdoor unit and the ISP connection point indoors. This is the standard WISP (wireless ISP) architecture adapted for BGP-X.

### 4.4 BGP-X Node v1 — Rooftop Dedicated Relay

```
Rooftop:
  [BGP-X Node v1, IP67] ← LoRa antenna
                         ← WiFi mesh antenna
  PoE: 802.3af, 15W (or solar — see solar deployment doc)
         │
         │ Cat6 cable
         │
Indoor:
  [PoE injector] ← [AC power]
  (optional: Ethernet tap for WAN — if configuring as gateway)

Config (relay-only):
  - role = "relay"
  - transports = ["lora", "wifi-mesh"]
  - No WAN required

Config (gateway):
  - role = "relay,domain_bridge"
  - WAN: Ethernet from PoE injector side to router/ISP modem
  - bridge_capable = true
```

---

## 5. Purpose-Built Mast Deployment

For maximum coverage, a purpose-built mast raises the BGP-X Node v1 or Router v1 to 10-30m elevation. This is the primary deployment method for community mesh islands covering districts or towns.

### 5.1 Mast Types

**Telescoping push-up mast** (10-18m): aluminum sections that extend and lock. Available from ham radio suppliers. No excavation needed; base plate bolted to ground. Requires guying (3-4 guy wires) at top and mid-sections. Good for temporary or semi-permanent deployment.

**Lattice tower** (up to 60m+): bolted steel sections. Requires concrete foundation. Engineering permit required in most jurisdictions. Maximum strength; survives wind; permanent installation. Used for high-priority community gateway nodes.

**Monopole/fiberglass mast** (6-18m): single fiberglass or steel pole set in concrete. Clean aesthetics; smaller footprint than lattice. Common for WISP deployments; directly reusable for BGP-X Node v1.

**Existing infrastructure**: communication towers, water towers, tall buildings, existing WISP masts — all suitable attachment points. Requires permission from owner; often negotiated as community partnership.

### 5.2 Structural Requirements

| Mast Height | Wind Load Design | Foundation Requirement |
|---|---|---|
| <6m (self-supporting) | 120 km/h sustained | Surface mount; weight plate |
| 6-12m | 120 km/h sustained + 1.5× safety | Concrete footing 0.5m³ |
| 12-20m | Engineering assessment required | Concrete footing 1-2m³; guying |
| >20m | Full structural engineering required | Engineered foundation; full guying |

BGP-X Node v1 weight: ~1.5 kg including enclosure and antennas. Wind area: ~0.015 m². At 30m height in 120 km/h wind: ~15 N force. Negligible for mast structural calculations; antenna wind area typically dominates.

### 5.3 LoRa Antenna Selection for Mast Deployment

| Antenna | Gain | Pattern | Use Case |
|---|---|---|---|
| 2 dBi rubber duck | Omnidirectional | 360° horizontal | Short-range; default included |
| 5 dBi fiberglass whip | Omnidirectional | 360°, slightly compressed | Standard mast deployment |
| 8 dBi fiberglass colinear | Omnidirectional | 360°, compressed vertical | Maximum omni range |
| 9 dBi Yagi | Directional | ~60° beam | Point-to-point island link |
| 12 dBi Yagi | Directional | ~40° beam | Long-distance directional link |

**Recommendation for community mast**: 8 dBi omnidirectional fiberglass antenna. Doubles effective range over 2 dBi rubber duck. At 30m elevation: 20-40 km coverage radius on clear terrain.

### 5.4 Cable Selection

| Cable | Loss @ 868 MHz (per 10m) | Max Run | Use Case |
|---|---|---|---|
| RG-58 | ~3.5 dB | 10m | Short runs only |
| LMR-200 | ~1.8 dB | 20m | Standard rooftop runs |
| LMR-400 | ~0.9 dB | 50m | Mast runs up to 30m |
| LMR-600 | ~0.6 dB | 80m+ | Tall tower deployments |
| Hardline 7/8" | ~0.3 dB | 200m+ | Very tall towers |

**Rule of thumb**: every 3 dB cable loss = halved effective range. Use the lowest-loss cable practical for the installation. Spend money on cable rather than antenna gain — cable loss is always harmful; antenna gain may cause regulatory issues.

**Connectors**: all connections at antenna N-type (weatherproof). SMA at BGP-X Node v1 end. Use N-to-SMA adapter at the enclosure cable entry gland. Weatherproof all outdoor connections with self-amalgamating tape.

---

## 6. PoE Power Delivery

### 6.1 Standard PoE Architecture

```
Building                        Mast (up to 100m)
  │                                    │
  ├─ PoE Switch/Injector ──CAT6────── PoE Splitter
  │  (802.3af, 15W)                    │
  │                                    ├─ Router/node (12V or 5V)
  │                                    └─ LoRa module (5V)
```

**CAT6 run length**: up to 100m (802.3at standard). Beyond 100m: use fiber with media converter.

**Fiber option** (beyond 100m):
```
Building: PoE → Media converter → SFP fiber → run → Media converter → PoE out → Node
```

### 6.2 PoE Power Degradation over Long Runs

PoE power degrades over long cable runs due to cable resistance. BGP-X Node v1 maximum power: ~12W.

| Cable Run | Voltage Drop (Cat6 24 AWG) | Minimum PoE Standard |
|---|---|---|
| 0-30m | Negligible | 802.3af (15W) |
| 30-60m | ~2V drop | 802.3at (30W) to compensate |
| 60-100m | ~4V drop | 802.3at required |
| >100m | Exceeds spec | Use 802.3bt (60W) or separate power run |

For mast runs >50m: use 802.3at PoE injector even though Node v1 only draws 12W. The higher voltage headroom compensates for cable resistance.

**Alternative for tall masts**: run 24V DC power cable separately from data Cat6. Use DC barrel connector at Node v1 (if carrier supports wide-input DC). Avoids PoE voltage drop issue entirely.

---

## 7. Lightning Protection

Any elevated installation requires lightning protection consideration.

### 7.1 Risk Assessment

- Urban area, building height <30m above surroundings: low risk; standard grounding sufficient
- Rural, exposed location, >30m above surroundings: significant risk; lightning rod and grounding required
- Tower >30m: always requires proper lightning protection system (LPS)

### 7.2 Grounding for BGP-X Node v1 on Mast

```
Lightning rod (if required by height/exposure)
      │ #6 AWG copper conductor
      ▼
Ground rod (1.8m copper-clad steel; 3m from mast base)
      │
      │ Bond to mast structure
      │ Bond to all antenna coax shields (via lightning arrestor)
      │
Ethernet cable: use outdoor-rated shielded Cat6 (STP/FTP)
Shield grounded at base only (one end); prevents ground loop
      │
Building entry: install Ethernet surge protector at building entry point
```

**Ethernet surge protector**: essential at the building entry point for any mast deployment. Protects indoor equipment from induced surges on the cable. ~$20-50; install between outdoor cable end and any indoor equipment.

### 7.3 Lightning Arrestors

For elevated installations:
- Lightning arrestor (N-type) on each antenna cable, rated for 1 kA
- Earthing cable from arrestor to earthing rod
- PoE port: surge suppressor at building entry

---

## 8. Network Configuration for Mast Nodes

Mast nodes typically serve as relay-only or dual-role (relay + LoRa amplifier):

```toml
[node]
roles = ["relay", "discovery"]
# Consider adding "entry" if this node should accept direct client connections

[mesh]
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"

# Use directional antenna sector for coverage if rooftop placement
wifi_mesh_sector_coverage_degrees = 120  # If directional antenna

lora_interface = "/dev/ttyUSB0"
lora_spreading_factor = 9  # Longer range vs SF7; 15-20km possible
lora_frequency_mhz = 868.0

[advertisement]
bandwidth_mbps = 50   # WiFi mesh backbone
is_gateway = false
```

### 8.1 802.11s Mesh Configuration for Long-Range

For directional WiFi links between masts:

```
Node A (mast 1, 5 GHz directional 16 dBi)
                    ←─────── 2km link ────────→
                                                Node B (mast 2, 5 GHz directional 16 dBi)
```

```bash
# Configure directional link
iw dev mesh0 mesh join bgpx-backbone \
    --freq 5180 \
    --beacon-interval 100 \
    --dot11MeshHWMPRannInterval 5000  # Reduce routing announcements for point-to-point
```

---

## 9. Multi-Node Mast Topology

For community mesh islands, deploy multiple elevated BGP-X Node v1 units to achieve full area coverage with redundancy.

### 9.1 Star Topology (Single Central Mast)

```
          [BGP-X Node v1 @ 30m, central mast]
          LoRa: 20 km radius coverage
         /                                    \
[Node A]                                    [Node B]
[ground level]                          [rooftop 10m]
      |                                       |
[Local devices]                       [Local devices]
```

Simple. Single point of failure. Best for small communities.

### 9.2 Mesh Topology (Multiple Elevated Nodes)

```
[Node v1 @ 30m, North] ←─ LoRa ─→ [Node v1 @ 25m, Central]
         |                                    |
     LoRa (15km)                         LoRa (12km)
         |                                    |
[Node v1 @ 20m, West]  ←─ LoRa ─→ [Node v1 @ 30m, South]
```

Each node covers its area; all nodes can route to each other. Multiple paths between any two island members. Resilient to single node failure.

**Recommended for community islands**: minimum 3 elevated nodes, each providing independent coverage of 33% of the island area, with overlapping coverage in center.

---

## 10. Range Extension Mode (Replacing Amplifier Concept)

**Note on Broadcast Amplifier (Deprecated Concept)**:

The original BGP-X specification included a "BGP-X Amplifier v1" as a separate product category — a minimal LoRa repeater with no BGP-X routing intelligence.

**This concept has been retired.**

The BGP-X Node v1 in Range Extension mode provides equivalent coverage benefits while maintaining full BGP-X encryption at every hop. A pure PHY-level amplifier that relays traffic without encryption introduces a security vulnerability — traffic would be exposed at the relay's radio interface.

### 10.1 Range Extension Configuration

For pure coverage extension without routing for others, configure BGP-X Node v1 in relay mode:

```toml
[node]
role = "relay"

[performance]
relay_only_mode = true
max_sessions = 500
path_construction = false

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island"
transports = ["lora"]
```

**Solar-powered range extension**:
- BGP-X Node v1 in IP67 enclosure
- 5W solar panel + 3000mAh LiPo
- 8 dBi fiberglass LoRa antenna
- Total cost: ~$80-100 (vs $60 for deprecated amplifier concept)

The Node v1 in Range Extension mode:
- IS a FULL BGP-X routing node (runs complete bgpx-node daemon)
- Prioritizes forwarding traffic over originating sessions
- Maintains full BGP-X onion encryption at every hop
- Provides LoRa coverage extension with all privacy properties intact

---

## 11. Regulatory Considerations

### 11.1 Mast Height Permits

Most jurisdictions have height limits below which no permit is required:

| Jurisdiction | Typical Permit-Free Height |
|---|---|
| United States (residential) | <6m (20 ft) above roofline for ham/receive antennas; varies for transmit |
| European Union (varies by country) | 2-5m above roofline for small installations |
| United Kingdom | 4m in designated areas; 15m in rural zones |

BGP-X operators should verify local planning and building code requirements before erecting masts.

### 11.2 Radio Frequency Licensing

LoRa radio operates in ISM bands (868 MHz EU, 915 MHz NA) which are license-free with power limits:
- EU: maximum +14 dBm EIRP (25 mW) in many sub-bands; duty cycle 1%
- NA (FCC Part 15.247): maximum +30 dBm EIRP; no duty cycle limit
- EIRP = TX power + antenna gain - cable loss

**Example calculations**:

With 8 dBi antenna, +14 dBm TX power, 1 dB cable loss: EIRP = 14 + 8 - 1 = **21 dBm EIRP**. Within EU limits.

With 8 dBi antenna, +20 dBm TX, 1 dB cable: EIRP = 27 dBm. Exceeds EU limit. Reduce TX power to 7 dBm for 14 dBm EIRP compliance.

BGP-X firmware enforces TX power limits per configured region SKU. Always verify EIRP is within local regulatory limits.

---

## 12. Maintenance

### 12.1 Quarterly Inspection

- Tighten cable glands and mounting hardware
- Check antenna connections for corrosion
- Inspect earthing connections
- Check dessicant (replace if saturated)
- Verify node reporting in monitoring

### 12.2 Annual Maintenance

- Full dismount inspection of enclosure seals
- Lightning arrestor test
- Antenna pattern check (use phone as signal probe)
- Update BGP-X firmware if new stable release

---

## 13. Deployment Checklist

### 13.1 Pre-Installation

- [ ] Site survey: measure elevation, identify obstructions, estimate LoRa coverage radius
- [ ] Obtain building owner permission (written)
- [ ] Verify local regulations for mast height and radio frequency
- [ ] Calculate EIRP and verify regulatory compliance
- [ ] Order hardware: BGP-X Router v1 (outdoor carrier) or BGP-X Node v1, antennas, cable, connectors, mast hardware
- [ ] Plan cable route from rooftop/mast to indoor equipment or PoE injector
- [ ] Plan lightning protection if required

### 13.2 Installation

- [ ] Mount mast or rooftop bracket; verify level; torque fasteners to spec
- [ ] Mount BGP-X hardware in outdoor IP67 enclosure; verify IP67 sealing
- [ ] Install LoRa antenna (SMA); self-amalgamating tape on connector
- [ ] Install WiFi antennas (RP-SMA × 4); tape connectors
- [ ] Run antenna feedline (LMR-400 or LMR-200); secure with UV-rated cable ties every 30cm
- [ ] Install Ethernet cable; use cable gland at enclosure entry; IP67 cable gland
- [ ] Install surge protector at building entry point
- [ ] Connect PoE injector; verify LED indicates powered device
- [ ] BGP-X hardware powers on; confirm via LED indicators (or SSH from indoor network)

### 13.3 Commissioning

- [ ] `bgpx-cli node identity` — confirm NodeID
- [ ] `bgpx-cli domains list` — confirm clearnet and mesh domains active
- [ ] `bgpx-cli node stats` — confirm sessions accumulating after 10-15 minutes
- [ ] Check LoRa RSSI from nearest known LoRa peer: `bgpx-cli node ping <peer_id> --domain mesh:<island_id>`
- [ ] If gateway: `bgpx-cli domains bridges --health` — confirm bridge pair online
- [ ] If gateway: `bgpx-cli islands show --self --dht-freshness` — confirm island published to DHT
- [ ] Document GPS coordinates, installation date, hardware serial, NodeID for records

---

## 14. Satellite WAN Integration

BGP-X Router v1 and BGP-X Node v1 (with WAN) support **satellite WAN connections** via USB satellite modems:

- **Starlink Gen 3**: USB port on terminal; presents as USB Ethernet adapter; BGP-X daemon detects via USB vendor ID; treats as clearnet domain with LEO latency class (20-40ms)
- **Iridium Certus 100/350**: Serial modem interface; BGP-X satellite transport driver handles AT commands; GEO latency class (600ms+)
- **Inmarsat BGAN/FBB**: Similar serial/IP interface; GEO latency class

**Important**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) provide **internet connectivity over BGP**. From BGP-X's perspective, they are clearnet transports — the same clearnet domain as fiber or cellular, just with higher latency. They are not a separate mesh routing domain. See `/docs/satellite_architecture.md` for the full architecture.

---

## 15. Physical Installation Best Practices

### 15.1 Pole Mounting

```
Standard installation:
  - 40mm diameter pole (galvanized steel)
  - Node enclosure: stainless steel hose clamps (2×)
  - Antenna: separate pole extension bracket (+30cm above node)
  - Earthing: stainless wire to earthing stake
  - Cable management: UV-resistant zip ties; drip loops at entry
  - Sealant: silicone sealant on all cable entry points
```

### 15.2 Weatherproofing

All cable penetrations MUST be sealed:
- Silicon-filled cable gland for each connection
- Drip loop (cable goes down before entering enclosure)
- Dessicant sachet inside sealed enclosure
- UV-resistant external components

### 15.3 Antenna Placement

For optimal performance:
- Separate LoRa and WiFi antennas by at least 30cm vertically
- Mount antennas above the node enclosure (not beside)
- Ensure antennas are above roofline or obstructions
- Use appropriate antenna gain for coverage area (higher gain = narrower vertical pattern)

---

## 16. Summary

Elevated BGP-X deployments extend mesh coverage by 10-100× compared to ground-level installations. The key success factors are:

1. **Height**: Antenna elevation is the single most important factor
2. **Line of sight**: Clear Fresnel zone for reliable links
3. **Cable quality**: Use low-loss coax (LMR-400 or better) for long runs
4. **Weatherproofing**: Proper IP67 sealing and cable glands are essential
5. **Lightning protection**: Required for installations >30m above surroundings
6. **Regulatory compliance**: Verify height permits and EIRP limits before installation

The BGP-X Router v1 and Node v1 are designed for outdoor deployment with IP67 carriers, PoE power delivery, and professional-grade antenna connectors. Use the right hardware for the right deployment scenario:

- **Router v1 outdoor carrier**: End user who wants AIO home router + elevated mesh relay
- **Node v1**: Community contributor deploying dedicated elevated relay or gateway
