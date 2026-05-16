# BGP-X Maritime Deployment

**Version**: 0.1.0-draft

---

## 1. Overview

Maritime deployment installs BGP-X hardware aboard vessels (sailboats, motorboats, commercial ships, ferries, fishing vessels) or on maritime infrastructure (buoys, offshore platforms, lighthouses, coastal stations). Maritime BGP-X nodes extend mesh coverage over water and connect maritime users to the BGP-X overlay via cellular, satellite, or shore-based WiFi WAN.

**Maritime vessels are mobile nodes**. Configuration should include `mobile = true` to indicate the node's position changes. Maritime nodes are exempt from geographic plausibility RTT scoring because vessel position and assigned IP addresses (particularly satellite) vary continuously.

**Hardware applicable to this deployment**:

| Hardware | Role | Deployment Context |
|---|---|---|
| **BGP-X Router v1** (outdoor IP67 carrier) | Full router for crew WiFi + relay + domain bridge | Vessels providing WiFi hotspot to crew/passengers; offshore passagemakers |
| **BGP-X Node v1** | Relay, domain bridge, mesh gateway | All vessel sizes; solar-powered autonomous operation; offshore platforms |
| **BGP-X Client Node** (Tier 2) | Endpoint only, no routing | Individual sensor nodes, handheld devices, low-power IoT aboard vessel |
| **BGP-X Adapter** (Tier 3) | USB LoRa modem | Laptop/desktop aboard vessel connecting to mesh via USB |

---

## 2. Maritime Environment Challenges

The maritime environment is one of the most hostile for electronics. BGP-X maritime deployments must address:

### 2.1 Saltwater Corrosion

Saltwater is extremely corrosive to electronics and metal components.

| Element | Challenge |
|---|---|
| Salt spray | Conductive residue accelerates corrosion and leakage currents |
| Humidity | Constant high humidity promotes condensation inside enclosures |
| UV exposure | Degrades plastics, cable jackets, antenna radomes |
| Temperature cycling | Day/night thermal cycling stresses seals |

**All maritime BGP-X installations MUST**:

- Use IP67 (minimum) enclosures — BGP-X Node v1 and Router v1 outdoor carrier meet this
- Use marine-grade stainless steel (316 SS) for all fasteners, brackets, mounting hardware
- Apply corrosion-inhibiting compound (Tef-Gel or equivalent) to all electrical connections
- Use tinned copper wire for all cable runs (bare copper corrodes rapidly)
- Apply self-amalgamating tape on ALL outdoor RF connectors (SMA, N-type, RP-SMA)
- Inspect all connections at minimum annually; more frequently in tropical/coastal environments

### 2.2 Corrosion Protection for Internal Components

For exposed circuit boards or connectors:

- **PCB coating**: Apply conformal coating (Electrolube HPA or similar acrylic/silicone conformal coating)
- **Preparation**: Clean PCB with IPA and flux remover before coating
- **Application**: 2× coats conformal coating minimum
- **Receptacles/connectors**: Mask before coating; apply contact lubricant after
- **Desiccant**: Include silica gel packets inside enclosure; replace annually

### 2.3 Vibration and Motion

Vessels experience constant vibration (engine, wave action) and motion (pitch, roll, yaw).

| Vessel Type | Vibration Profile |
|---|---|
| Sailboat under sail | Low-frequency flex and impact (wave slap) |
| Motorboat at speed | High-frequency engine vibration |
| Commercial vessel | Continuous low-frequency engine vibration |
| Ferry | Varies — docking shocks plus underway vibration |

**Mounting requirements**:

- Secure all mounting hardware with locking fasteners (Nyloc nuts, thread-locking compound, or safety wire)
- Shock-mount sensitive electronics using rubber grommets or commercial shock mounts
- Cable management: secure all cables every 30 cm; prevent chafing against sharp edges during vessel motion
- Use cable ties rated for UV and salt exposure (marine-grade nylon or stainless steel)

### 2.4 Salt Spray Ingress Prevention

Even IP67-rated enclosures can accumulate salt deposits on cable glands and seals over time.

- Inspect cable glands annually
- Replace O-rings as needed (every 2-3 years minimum)
- Flush exterior of enclosures with fresh water after exposure to breaking waves or heavy spray
- Do not pressure-wash enclosures — water can force past seals

---

## 3. WAN Connectivity Options for Maritime

Maritime BGP-X nodes require a clearnet connection to participate in the BGP-X overlay. The WAN connection is always **clearnet domain** (type 0x00000001), regardless of physical transport.

### 3.1 Cellular (Near Shore)

```
LTE/5G cellular modem (USB or built-in) → BGP-X clearnet domain
Coverage: 0-30 km from shore (typical LTE maritime range)
Bandwidth: 10-150 Mbps
Latency: 20-80 ms
Cost: Standard maritime data plan
```

Most cost-effective WAN for coastal cruising and harbor operation. Range degrades with distance from shore — cells are designed for land coverage, not offshore. At 20 km offshore, LTE signal may be intermittent. At 30+ km: no cellular.

**Recommended hardware**: Multi-carrier LTE/5G USB modem with external antenna connector. Mount antenna at masthead or radar arch for maximum height.

### 3.2 Satellite (Offshore and Open Ocean)

**Satellite internet is clearnet domain**. From BGP-X's perspective, satellite services (Starlink, Iridium, Inmarsat) are WAN connections to the clearnet. They provide BGP-routed IP connectivity. The physical medium (satellite radio) is invisible to the BGP-X protocol.

#### Starlink Maritime

```
Starlink flat-panel maritime terminal → Ethernet / WiFi output
Coverage: Global (except polar extremes — see Starlink coverage map)
Bandwidth: 20-200 Mbps
Latency: 25-60 ms (LEO constellation)
Cost: Maritime plan pricing (verify current)
Power: 50-75W continuous
```

Starlink Maritime is the **recommended satellite WAN** for offshore vessel BGP-X deployment:

- Self-tracking electronically (no mechanical mount needed)
- Operates underway at typical vessel speeds
- Automatic beam handoff
- Flat-panel design rated for marine environment
- Connects to BGP-X Router v1 or Node v1 via Ethernet

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
satellite_orbit = "leo"
latency_class = "satellite-leo"
```

#### Iridium Certus (True Global Including Poles)

```
Coverage: Every point on Earth including poles
Bandwidth: 88-704 Kbps (depending on service tier)
Latency: 30-80 ms LEO latency, but low bandwidth
```

For high-latitude voyaging where Starlink coverage degrades, Iridium provides reliable but low-bandwidth connectivity. Suitable for BGP-X DHT participation and light relay duty; not suitable for high-throughput relay.

#### Inmarsat BGAN / Fleet Xpress

```
Coverage: Global (except extreme polar)
Bandwidth: 50 Mbps (Fleet Xpress)
Latency: 600+ ms (GEO)
```

GEO satellite service. High latency makes real-time application challenging. Suitable for DHT, messaging, and batched data transfer. Not recommended for interactive sessions.

### 3.3 Shore WiFi (Harbor / Marina)

When anchored or at berth, vessel BGP-X hardware can use shore WiFi as WAN:

```
Shore marina WiFi → Long-range WiFi adapter → BGP-X hardware → Clearnet
```

**Recommended hardware**: High-gain directional USB WiFi adapter (Alfa AWUS036ACH or equivalent) with parabolic or panel antenna.

- Mount WiFi adapter at masthead or radar arch
- Point toward marina access point
- BGP-X daemon treats shore WiFi as clearnet WAN
- Automatic failover to cellular/satellite when leaving harbor range

---

## 4. LoRa Coverage Over Water

Water has very low radio absorption. LoRa links over water significantly exceed land-based estimates due to flat reflection surface and Fresnel zone clearance over curved ocean.

### 4.1 Estimated Range Over Water

| Spreading Factor | Calm Water Range | Rough Sea Range | Notes |
|---|---|---|---|
| SF7 | 15-30 km | 8-15 km | High throughput, shorter range |
| SF9 | 30-60 km | 20-40 km | Balanced |
| SF12 | 60-120 km | 40-80 km | Maximum range, lowest throughput |

**Antenna height matters critically**. Antenna at 10m above waterline approximately doubles range vs. deck level due to Fresnel zone clearance over the curved water surface.

### 4.2 LoRa Antenna Mounting on Vessels

**Masthead preferred** for maximum range. If masthead is not feasible: radar arch, stern rail mount, or spreaders.

**Installation requirements**:

- Use marine N-type or PL-259 bulkhead connector at deck penetration (waterproof)
- Route coax inside mast (if possible) or secure externally with UV-rated cable ties every 30cm
- Use low-loss coax (LMR-400 or equivalent) for masthead runs exceeding 10m
- Lightning arrestor at deck level — **essential** because the mast is the most likely lightning strike point
- Ground the lightning arrestor to vessel ground (not just coax shield)

**Antenna selection**:

- **Recommended**: 5-8 dBi fiberglass collinear antenna
- **Pattern**: Omnidirectional (vessel changes heading; cannot track nodes with a directional antenna)
- **Warning**: Marine VHF antennas (156-174 MHz) are NOT compatible with LoRa 868/915 MHz — do not use VHF antennas for LoRa

---

## 5. WiFi Mesh Antenna for Vessel-to-Vessel Communication

WiFi 802.11s mesh between vessels in convoy or at anchor:

| Scenario | Range | Band |
|---|---|---|
| Anchored vessels, marina | 500m - 3km | 5 GHz (less interference) or 2.4 GHz (longer range) |
| Convoy underway | 200m - 1km | 5 GHz |

**Antenna**: Omnidirectional 5 dBi fiberglass, similar mounting to LoRa antenna.

---

## 6. Satellite Antenna Mounting

**Starlink Maritime flat-panel**:

- Mount on aft deck, cabin top, or radar arch with unobstructed sky view
- Keep minimum 1m separation from radar scanner (radar interference)
- Power: dedicated cable run from Starlink terminal; do not share PoE with BGP-X hardware
- The Starlink terminal consumes 50-75W — significant for small vessels; plan power budget

---

## 7. Installation Configurations

### 7.1 Configuration 1: Coastal Cruiser (Sailboat, Motorboat)

**Use case**: Day sailing, coastal cruising, weekend trips. Near-shore and offshore within 20km of cellular coverage, with satellite backup for extended passages.

**WAN**: LTE (primary) + Starlink (failover for offshore)

**Hardware**: BGP-X Node v1 or Router v1 (outdoor IP67)

**Antenna placement**:

```
Vessel mast (10-15m above water):
  LoRa antenna (8 dBi fiberglass) — masthead
  WiFi mesh antenna (5 dBi) — spreaders or radar arch

Cabin top or stern rail:
  Starlink flat-panel
  LTE/5G antenna (high-gain)
  BGP-X Node v1 (IP67 enclosure, bolted to deck)
```

**Power**: 12V vessel electrical system → DC-DC step-down (12V to 5V) → BGP-X Node v1

**Configuration**:

```toml
[node]
mobile = true                        # Vessel moves — mobile node declaration
roles = ["relay", "entry", "discovery"]

[clearnet_transport]
wan_type = "auto"                    # Use LTE in range; Starlink offshore
wan_failover = true
latency_class = "broadband"          # When on LTE
satellite_class = "satellite-leo"    # When on Starlink

[[routing_domains]]
domain_type = "clearnet"
endpoints = []                       # Auto-detected from WAN interface

[[routing_domains]]
domain_type = "mesh"
island_id = "coastal-vessel-mesh"
transports = ["lora", "wifi-mesh"]
island_auto_join = true             # Join spontaneous vessel mesh
```

### 7.2 Configuration 2: Offshore Passagemaker (Bluewater Sailboat)

**Use case**: Ocean passages, long-distance voyaging, trans-oceanic trips. May spend weeks out of cellular range.

**WAN**: Starlink Maritime (primary) + Iridium (emergency/polar backup)

**Hardware**: BGP-X Router v1 (outdoor IP67) — provides crew WiFi hotspot in addition to relay

**Antenna placement**:

```
Mast (15-20m above water):
  LoRa antenna (8 dBi fiberglass) — masthead

Stern rail or aft deck:
  Starlink flat-panel
  BGP-X Router v1 (IP67 outdoor carrier)
    - WiFi AP for crew devices
    - WAN: Starlink Ethernet

Cabin:
  Iridium Certus terminal (if equipped)
```

**Configuration**:

```toml
[node]
mobile = true
roles = ["relay", "entry", "exit", "discovery"]

[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
latency_class = "satellite-leo"

[[routing_domains]]
domain_type = "clearnet"

[[routing_domains]]
domain_type = "mesh"
transports = ["lora"]
island_auto_join = true

[gateway]
enabled = true                      # Act as gateway for other vessels in range
```

### 7.3 Configuration 3: Commercial Vessel / Ferry

**Use case**: Ferry routes, cargo ships, passenger vessels. High-bandwidth multi-WAN bonding.

**WAN**: 4G/5G multi-SIM bonding + Starlink

**Hardware**: BGP-X Gateway v1 (rack-mount or outdoor IP65)

**Function**:

- Passenger WiFi hotspot (standard 802.11 AP; all traffic through BGP-X overlay)
- BGP-X relay node (LoRa mesh with other vessels)
- Domain bridge: clearnet (multi-WAN) ↔ LoRa mesh (fishing fleet, harbor vessels)

**Power**: Vessel 24V DC system → AC inverter → Gateway v1; or DC-DC converter if available

### 7.4 Configuration 4: Fixed Maritime Station

**Use case**: Lighthouse, offshore platform, buoy, coastal relay station. Fixed position; not mobile.

**WAN**: Satellite (Starlink or Inmarsat) or microwave link to shore

**Hardware**: BGP-X Node v1 mounted at maximum height on structure.

**Key characteristic**: Fixed position; high elevation provides extraordinary LoRa range over water (50+ km from 30m elevation over flat ocean).

**Configuration**:

```toml
[node]
mobile = false                       # Fixed installation
roles = ["relay", "discovery"]

[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"

[[routing_domains]]
domain_type = "clearnet"

[[routing_domains]]
domain_type = "mesh"
island_id = "offshore-platform-relay"
transports = ["lora"]
```

---

## 8. BGP-X Maritime Mesh Network

Multiple BGP-X-equipped vessels in the same area form a spontaneous mesh island:

```
[Vessel A: 15km offshore]
        │ LoRa (8 km range at 5m elevation)
        ▼
[Vessel B: 7km offshore] ←── LoRa ──→ [Vessel C: 5km offshore]
        │
        │ LoRa (range to shore)
        ▼
[Shore Station: lighthouse or coastal relay node]
        │ WAN (fiber or microwave)
        ▼
[Clearnet BGP-X network]
```

**Mesh island properties**:

- Vessels within LoRa range of each other discover via MESH_BEACON broadcast
- Vessels automatically form a mesh island (no configuration required beyond `island_auto_join = true`)
- Vessels within LoRa range of a shore station can access clearnet via the shore station's gateway
- Vessels beyond shore range but within LoRa range of another vessel relay through multiple vessel hops
- End-to-end encryption maintained throughout — vessel mesh is BGP-X encrypted, not just LoRa packet radio

**Spontaneous fleet mesh**:

When two or more vessels are within LoRa range:

1. Each vessel's MESH_BEACON announces its presence on LoRa
2. Each vessel discovers the other's NodeID
3. Vessels form a spontaneous mesh island automatically
4. Fleet communication (vessel-to-vessel) uses BGP-X native services — encrypted end-to-end
5. No routing through satellite required for intra-fleet communication

---

## 9. Domain Bridge Configuration for Maritime Gateways

A vessel with both satellite WAN and LoRa mesh can act as a **domain bridge node**, connecting the maritime LoRa mesh to the clearnet overlay.

```toml
[node]
mobile = true
roles = ["relay", "entry", "exit", "discovery", "domain_bridge"]

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "auto", port = 7474 }]  # Starlink WAN

[[routing_domains]]
domain_type = "mesh"
island_id = "maritime-fleet-alpha"
transports = ["lora"]

[domain_bridge]
enabled = true
bridges = [
    {
        from_domain = "clearnet",
        to_domain = "mesh:maritime-fleet-alpha",
        transport = "lora",
        available = true
    }
]
```

**Domain bridge behavior**:

- The vessel's BGP-X daemon routes traffic between clearnet (via satellite) and mesh (via LoRa)
- Other vessels within LoRa range can reach clearnet via this bridge
- The bridge node does not see the content (onion encryption) or link source to destination
- All privacy properties of BGP-X are maintained across the domain boundary

---

## 10. Maritime Regulatory Notes

**LoRa (868 MHz EU / 915 MHz US)**: ISM band — generally permitted without license for maritime use on vessels flagged in most jurisdictions. However:

- Some flag states require maritime radio operator certification
- Transmit power limits apply (check your flag state regulations)
- LoRa devices may be subject to maritime radio equipment standards (IEC 60945 for safety equipment)

**WiFi (2.4 GHz / 5 GHz)**: ISM band — generally permitted. Range over water extends further than expected; configure transmit power appropriately to avoid interference.

**Satellite terminals**: Starlink, Iridium, Inmarsat terminals are type-approved for maritime use by their manufacturers. Ensure your terminal is the maritime-rated version (motion-rated for underway operation).

**THIS GUIDE IS INFORMATIONAL ONLY**. Verify compliance with maritime communications regulations for your vessel's flag state and operating area. BGP-X provides no legal advice regarding radio operation.

---

## 11. Maintenance at Sea

Maritime environments accelerate maintenance requirements.

### Weekly

- Visual inspection of antenna connections (corrosion, looseness)
- Check enclosure for condensation or water ingress
- Verify node reporting in dashboard or via `bgpx-cli node status`
- Check LoRa RSSI from known shore station or other vessel

### Monthly

- Wipe antenna connections with corrosion inhibitor (Boeshield T-9 or equivalent)
- Check cable management for chafe or UV degradation
- Verify satellite terminal pointing (Starlink self-tracks, but check for physical obstruction)
- Test WAN failover (disconnect primary; verify backup activates)

### Annually (or upon haul-out)

- Full dismount inspection
- Re-apply conformal coating to PCBs
- Replace desiccant packets in enclosures
- Inspect all cable sheaths for UV degradation
- Replace any corroded connectors
- Verify lightning arrestor ground connection
- Check O-rings on all waterproof connectors; replace as needed

---

## 12. Power System Integration

Marine 12V/24V DC systems are well-matched to BGP-X hardware requirements.

```
Main Battery Bank (AGM / LiFePO4)
         │
         ├─── Battery monitor (Victron BMV-712 or equivalent)
         │
         ├─── Fuse (5-10A appropriate for load)
         │
         ├─── DC-DC converter (12V/24V → 5V, 3A minimum)
         │
         └─── BGP-X Node v1 / Router v1
```

**Power budget**:

| Component | Power Draw |
|---|---|
| BGP-X Node v1 (LoRa only) | 0.5-2W |
| BGP-X Node v1 (WiFi + LoRa) | 2-8W |
| BGP-X Router v1 (all radios) | 5-15W |
| Starlink terminal | 50-75W |
| Iridium terminal | 5-15W |

**Solar supplementation**: Most sailing vessels have solar and wind generation. BGP-X Node v1 adds <5W to the power budget — typically negligible for vessels with existing solar arrays. For smaller vessels without solar, a 50W panel and small LiFePO4 battery can power a LoRa-only Node v1 indefinitely in most latitudes.

---

## 13. Maritime Deployment Checklist

### Hardware Installation

- [ ] All hardware in IP67 enclosures (minimum); enclosures bolted securely with Nyloc fasteners
- [ ] All mounting hardware: marine-grade 316 stainless steel
- [ ] All electrical connections: tinned copper wire; corrosion-inhibiting compound applied
- [ ] All RF connectors: self-amalgamating tape applied
- [ ] LoRa antenna at maximum height (masthead preferred); cable secured every 30cm
- [ ] WiFi antennas installed (if applicable)
- [ ] Satellite terminal mounted with clear sky view; minimum 1m from radar scanner
- [ ] Lightning arrestor installed at deck level; grounded to vessel ground
- [ ] Power connection: fused tap from vessel 12V/24V panel
- [ ] UPS or battery buffer recommended for engine-start voltage dips

### Configuration Verification

- [ ] `mobile = true` set in config (vessel moves)
- [ ] `island_auto_join = true` configured (or specific `island_id` for fleet)
- [ ] `session_max_duration_secs` set appropriately (consider satellite latency)
- [ ] WAN type configured correctly (cellular / satellite / auto-failover)
- [ ] Satellite latency class declared (`satellite-leo` for Starlink; `satellite-geo` for Inmarsat)
- [ ] `bgpx-cli node identity` — confirm NodeID
- [ ] `bgpx-cli domains list` — confirm all expected domains active
- [ ] LoRa RSSI from shore station or test vessel verified

### Sea Trial

- [ ] LoRa range test underway: verify range at 1 km, 5 km, 10 km from other vessel or shore
- [ ] WAN failover tested: disconnect LTE; verify Starlink takes over (if equipped)
- [ ] BGP-X overlay path established: confirm multi-hop path via `bgpx-cli paths list`
- [ ] Session established and maintained for 30 minutes while vessel moving
- [ ] Session graceful close when moving out of range (not KEEPALIVE_TIMEOUT)
- [ ] NODE_WITHDRAW on vessel power-off confirmed
- [ ] Fleet mesh formation tested when multiple vessels within range
