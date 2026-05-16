# BGP-X Aerial Deployment

**Version**: 0.1.0-draft

---

## 1. Overview

Aerial platforms elevate BGP-X hardware to altitudes unreachable by ground-based masts, providing temporary, emergency, or semi-permanent coverage over wide areas. A node at 150m altitude can achieve coverage equivalent to 20+ ground nodes.

**Deployment Categories**:
1.  **Tethered Platforms**: Aerostats, kites, and tethered drones. Semi-permanent, stable, suitable for events or disaster response.
2.  **Free-Flying Platforms**: Multirotor UAVs, fixed-wing UAVs, and high-altitude balloons. Short-duration or specialized research deployments.

This guide covers platform selection, payload design, regulatory compliance, and operational procedures for both categories.

---

## 2. Hardware Selection

Aerial deployment hardware selection is critical due to weight, power, and environmental constraints.

| Platform Type | Recommended Hardware | Why |
|---|---|---|
| **Tethered Aerostat** | **BGP-X Node v1** | High lift capacity supports full Node weight; PoE/battery options. |
| **Tethered Kite** | **BGP-X Client Node / Stripped Node v1** | Limited payload capacity; weight critical. |
| **Tethered Drone** | **BGP-X Node v1** | Professional platforms can carry 2-5kg; unlimited power via tether. |
| **Multirotor UAV** | **BGP-X Client Node** | Strict payload limits (<500g); short duration. |
| **Fixed-Wing UAV** | **BGP-X Client Node** | Moderate payload (500g-1kg); longer duration. |
| **High-Altitude Balloon** | **BGP-X Client Node** | Minimal payload weight; extreme altitude.

**Note on BGP-X Router v1**: At ~2kg, the Router v1 is generally too heavy for most aerial platforms. Use Node v1 (AIO outdoor node) or Client Node (low-cost endpoint) instead.

---

## 3. Platform Types

### 3.1 Tethered Aerostat (Weather Balloon / Helium Balloon)

A helium-filled balloon tethered to a ground station, suspending a BGP-X Node v1 payload.

**Advantages**:
- Altitude: 50-300m AGL (above ground level)
- Duration: Hours to days (battery) or indefinite (tether power)
- Coverage: 40-60 km radius at 150m altitude (LoRa, clear terrain)
- Stable platform for directional antennas
- Less regulated than free-flying aircraft

**Balloon Sizing**:
- Total payload: Node v1 (~1.5 kg with battery & enclosure) + Tether (~0.5 kg/100m) = ~3-5 kg
- Required lift: 2× payload = 6-10 kg net lift
- Balloon diameter: ~2.5-3 meters for 10 kg lift
- Helium volume: ~8 m³

**Power Options**:
- **Battery-only**: 2× 3S 5000 mAh LiPo (~110 Wh) provides 38+ hours at 2W (Node v1 typical draw).
- **Tether Power**: 2× 16 AWG copper conductors woven into tether (adds ~100g/10m). 13V supply at ground delivers 12.6V at 100m altitude.

**Weather Limits**: Wind >20 km/h makes control difficult; >40 km/h is unsafe.

---

### 3.2 Tethered Kite

A kite-suspended platform. Requires wind; no helium cost.

**Advantages**:
- Free operation (no helium)
- Silent and discreet
- Deployable with minimal equipment

**Limitations**:
- Requires minimum 15 km/h wind (3-4 Beaufort)
- Less stable than aerostats
- Payload limit: ~500g for standard kites; ~800-1000g for large delta kites (3m²+)

**Hardware**: Use BGP-X Client Node (LILYGO T3S3) or stripped-down Node v1 (remove WiFi module, minimal enclosure). Target total payload weight <800g.

---

### 3.3 Tethered Drone Platform

A multirotor or fixed-wing drone connected by a power/data tether to a ground station.

**Advantages**:
- Rapid deployment (15-30 minutes)
- High wind tolerance (50+ km/h)
- Unlimited power via tether
- Mobile ground station (vehicle-mounted)

**Limitations**:
- High equipment cost ($5,000-20,000 for system)
- Noise (multirotors)

**Hardware**: BGP-X Node v1 fits comfortably on professional tethered drones with 2-5kg payload capacity.

---

### 3.4 Multirotor UAV (Drone)

Free-flying multirotor for temporary coverage.

| Capability | Specification |
|---|---|
| Altitude | 50-400m AGL (regulatory dependent) |
| Duration | 15-45 minutes (battery dependent) |
| Coverage | 10-50 km radius (LoRa at altitude) |

**Payload Design (5kg-class hexacopter)**:
Target payload budget: <1 kg. Use Client Node or custom lightweight payload.

| Component | Weight |
|---|---|
| Compute (Pi Zero 2W / ESP32-S3) | 12-25g |
| LoRa Module (SX1262) | 8g |
| Antenna (LoRa flexible) | 30g |
| Battery (2S 2200 mAh LiPo) | 130g |
| Minimal Enclosure | 80g |
| Cabling/Mounting | 25g |
| **Total** | **~350g** |

---

### 3.5 Fixed-Wing UAV

For longer-duration survey or emergency coverage.

| Capability | Specification |
|---|---|
| Altitude | 100-2000m AGL |
| Duration | 1-4 hours |
| Coverage | 50-200 km radius |

**Payload**: Similar to multirotor (~350-800g). Use Client Node hardware.

---

### 3.6 High-Altitude Balloon (HAB)

For extreme-range research or special operations.

| Capability | Specification |
|---|---|
| Altitude | 20-35 km (stratosphere) |
| Duration | 6-24 hours |
| Coverage | 300-800+ km (LoRa SF12) |

**Payload**: Lightweight Client Node. GPS-triggered shutdown on descent. Data logger for post-flight analysis.

---

## 4. Payload Design

### 4.1 BGP-X Node v1 Payload (For Aerostats, Tethered Drones)

| Component | Weight |
|---|---|
| BGP-X Node v1 PCB + Heatsink | ~400g |
| IP67 Enclosure | ~600g |
| LoRa Antenna (Fiberglass) | 50g |
| WiFi Antennas (×2, optional) | 100g |
| Battery (12h operation) | ~300g |
| Mounting Frame | ~200g |
| **Total** | **~1,650g** |

**Power**:
- Battery-only (111 Wh): 55+ hours at 2W draw.
- Tether power (24V DC via tether): Unlimited duration.

---

### 4.2 Lightweight Client Node Payload (For UAVs, Kites)

| Component | Weight |
|---|---|
| LILYGO T3S3 (ESP32-S3 + SX1262) | 25g |
| Flexible LoRa Antenna | 30g |
| Minimal 3D-Printed Case | 40g |
| 18650 Battery (3400 mAh) | 50g |
| Mounting Hardware | 15g |
| **Total** | **~160g** |

---

## 5. BGP-X Node Configuration

For aerial relay nodes, configure for maximum coverage and minimal overhead:

```toml
[node]
roles = ["relay", "discovery"]
# No exit role — aerial nodes are relay only

[mesh]
enabled = true
transports = ["lora"]
lora_spreading_factor = 7  # SF7 for low latency — altitude compensates for range
beacon_interval_seconds = 15  # Frequent beacons due to changing peers

[performance]
relay_only_mode = true  # Maximize forwarding; minimize overhead
max_sessions = 500      # Reduced for aerial context

[advertisement]
# Aerial node notes for geo plausibility scorer
link_quality_profiles.lora.latency_class = 0  # Low latency expected
bandwidth_mbps = 54

[geo_plausibility]
# Aerial nodes may move; disable geo checks or set wide tolerance
exempt = true
# Or set jurisdiction if fixed-position aerostat:
# country = "US"
# region = "NA"
```

**Disaster Response Configuration**:

```toml
[node]
role = "relay"

[[routing_domains]]
domain_type = "mesh"
island_id = "emergency-response-2026"
transports = ["lora", "wifi-mesh"]

[power_management]
enabled = true
emergency_shutdown_at_pct = 3   # Extend operation
graceful_shutdown_wait_secs = 30
```

---

## 6. Deployment Procedure — Tethered Aerostat

### 6.1 Equipment Checklist

- [ ] Helium cylinder (regulator, fill nozzle)
- [ ] Balloon (appropriate size for payload)
- [ ] Tether line: ≥2mm Dyneema or Kevlar
- [ ] Ground anchor: stake or vehicle tow hook
- [ ] BGP-X Node v1 (configured, charged)
- [ ] Payload attachment: 3× independent straps
- [ ] Wind anemometer
- [ ] Radio communications for ground team
- [ ] Warning light (red LED on tether per aviation regulations)

### 6.2 Inflation and Launch

```
1. Check wind: < 20 km/h sustained, < 30 km/h gusts
2. Inflate balloon; verify lift ≥2× payload weight
3. Attach tether; inspect knots (bowline)
4. Attach payload 2m below balloon; three-point attachment
5. Verify Node v1 powered and communicating (WiFi or LoRa)
6. Walk balloon to clear area; release slowly
7. Pay out tether to target altitude; secure to ground anchor
8. Verify coverage from ground reference node
```

### 6.3 Monitoring

Every 30 minutes:
- Check wind speed (abort if >25 km/h)
- Verify tether tension steady
- Check altitude stable (helium loss)
- Verify Node v1 reachable

### 6.4 Recovery

```
1. Haul in tether; secure balloon before detaching payload
2. Power off Node v1
3. Vent or preserve helium
4. Inspect tether and attachments
5. Review BGP-X relay statistics
```

---

## 7. Regulatory Requirements

### 7.1 United States (FAA)

| Platform | Regulation |
|---|---|
| **UAV (Multirotor/Fixed-Wing)** | Part 107 registration (>250g). Max altitude 400 ft (120m) AGL without waiver. BVLOS requires waiver. |
| **Tethered Aerostat** | No FAA authorization required if <500 ft (150m) AGL and >3nm from airport. Requires orange/white flag on tether (day) and red light (night). |
| **Free Balloon** | FAA notification required. See 14 CFR 101. |

**LAANC**: Use for authorization near controlled airspace.

### 7.2 European Union (EASA)

| Platform | Regulation |
|---|---|
| **UAV** | EASA regulations (A1/A2/A3 categories). Registration required. |
| **Tethered Aerostat** | Generally permitted up to 150m AGL with notification to national aviation authority. |

### 7.3 General Recommendation

Always notify local authorities (airport control, emergency services) before deployment, even where not legally required. Mark tether with warning tape or orange flags every 10m.

---

## 8. Disaster Response Rapid Deployment

BGP-X aerial deployment is critical for disaster response when terrestrial infrastructure is damaged.

### 8.1 Rapid Deployment Kit

- BGP-X Node v1 (pre-configured)
- 3m helium balloon (packed)
- 2× 3.5L helium cylinders (produces ~7m³)
- 150m Dyneema tether
- 12Ah LiPo battery pack (pre-charged)
- Ground anchor stake
- Payload attachment hardware

**Deployment Time**: 15-20 minutes from vehicle to airborne.

**Coverage**: 40+ km radius at 150m AGL. Single aerial Node v1 covers a city-scale area.

---

## 9. Pre-Deployment Checklist

### Equipment
- [ ] Balloon/kite/drone inspected
- [ ] Tether integrity verified
- [ ] BGP-X Node powered, configured
- [ ] Battery charged (or tether power tested)
- [ ] Antenna connections secure

### Site
- [ ] Wind forecast checked (<20 km/h for aerostats)
- [ ] Airspace verified (no TFR, airport proximity)
- [ ] Ground anchor position identified
- [ ] Launch area clear of obstacles

### Operations
- [ ] Communications check (ground team)
- [ ] Payload attachment tested
- [ ] Warning lights/markers ready
- [ ] Authorities notified (if required)
- [ ] Recovery team briefed

---

## 10. Additional Resources

- **Hardware Specifications**: `/hardware/hardware_spec.md`
- **Node v1 Details**: `/hardware/node_spec.md`
- **Client Node Details**: `/hardware/client_node_spec.md`
- **Satellite Integration**: `/docs/satellite_architecture.md`
