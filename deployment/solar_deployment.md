# BGP-X Solar Deployment

**Version**: 0.1.0-draft

---

## 1. Overview

Solar deployment enables BGP-X nodes in locations without grid power: remote terrain, agricultural land, mountain passes, maritime buoys, roadside locations, disaster response scenarios, and community mesh infrastructure. Solar-powered BGP-X nodes are first-class participants in the network — they relay traffic, participate in the DHT, and can serve as domain bridges or mesh island gateways.

**Hardware applicable to this deployment**:
- **BGP-X Node v1** (primary): solar MPPT and LiPo/LiFePO4 battery integrated; designed for 24/7 off-grid operation
- **BGP-X Router v1** (outdoor carrier): with external solar MPPT charge controller and battery; higher power requirement (5-15W)
- **BGP-X Client Node** (Tier 2): low-power LoRa endpoint for solar/sensor deployment
- **Third-party hardware**: GL.iNet GL-MT3000, Raspberry Pi, and other OpenWrt-compatible hardware with solar systems

**This guide covers**:
- Power system design for each hardware tier
- Battery chemistry selection for different climates
- Solar panel selection and mounting
- Firmware power management features
- Complete deployment configurations
- Installation checklist

---

## 2. Power Budget Fundamentals

### 2.1 Power Budget Calculation

The power budget determines solar array size and battery capacity:

```
Daily energy required = active_power × active_hours + idle_power × idle_hours

Example calculation for BGP-X Node v1 (LoRa-only):
  Idle power: 0.5W average
  Daily energy: 0.5W × 24h = 12 Wh

Battery capacity (3 days autonomy, 50% depth of discharge):
  = 12 Wh × 3 / 0.5 = 72 Wh → use 4000 mAh LiPo @ 3.7V = 14.8 Wh (tight)
  → Recommended: 8000 mAh LiPo @ 3.7V = 29.6 Wh (safe margin)

Solar panel (accounting for 4-5 peak sun hours/day, 80% efficiency):
  = 12 Wh / (4 × 0.8) = 3.75W → minimum 4W panel
  → Recommended: 6W panel for cloudy day margin
```

### 2.2 Power Requirements by Hardware

**BGP-X Node v1 Power Consumption**:

| Mode | Consumption | Notes |
|---|---|---|
| LoRa-only relay (idle) | 0.5W | Minimum viable configuration |
| LoRa-only relay (TX active) | 1-1.5W | During packet transmission |
| WiFi mesh + LoRa (idle) | 2W | Standard community relay |
| WiFi mesh + LoRa (active) | 3-5W | During heavy traffic |
| All radios active (WiFi + LoRa + BLE) | 3W idle, 8W peak | Gateway mode |
| All radios + satellite WAN | 8-85W | Starlink dominates power budget |

**BGP-X Router v1 Power Consumption**:

| Mode | Consumption | Notes |
|---|---|---|
| Relay + LAN routing | 5W idle | Standard dual-stack |
| Full active with all radios | 10-15W | Heavy traffic |
| Router + satellite WAN | 55-90W | Starlink terminal adds 50-75W |

**BGP-X Client Node (Tier 2) Power Consumption**:

| Mode | Consumption | Notes |
|---|---|---|
| LoRa receive (idle) | 0.1W | Deep sleep between beacons |
| LoRa transmit | 0.5W | During TX burst |
| WiFi connected | 0.3W | Additional WiFi radio |
| Active all radios | 0.8W peak | Maximum |

### 2.3 Conservative Sizing Guidelines

Always oversize solar and battery by 50-100% for reliability:

| Component | Minimum | Recommended |
|---|---|---|
| Solar panel | 1.5× calculated wattage | 2× calculated wattage |
| Battery capacity | 2 days autonomy | 3-5 days autonomy |
| Charge controller | 1.5× max panel current | 2× max panel current |
| Solar cable | 4mm² | 6mm² |

---

## 3. Power System Design

### 3.1 BGP-X Node v1 Power System

The BGP-X Node v1 integrates a CN3791-based MPPT solar charge controller directly on the PCB. Solar input connects directly to the Node v1 — no external charge controller required.

```
Solar Panel (6V, 3-10W)
      │ (positive and negative)
      ▼
BGP-X Node v1 SOLAR input (reverse polarity protected)
      │ internal MPPT circuit
      ▼
LiPo / LiFePO4 battery (3.7V or 3.2V nominal)
      │ integrated battery management
      ▼
5V boost converter → SoC, radios, peripherals
```

**Minimum solar panel**: 6V, 3W. Sufficient for 24/7 LoRa-only relay with ≥4 hours daily sun.

**Recommended solar panel**: 6V, 6W (or 2× 3W panels in parallel). Provides margin for cloudy days and operation with WiFi + LoRa active.

**Battery capacity by operation mode**:

| Operation Mode | Idle Power | Overnight Energy (14h) | Minimum Battery | Recommended Battery |
|---|---|---|---|---|
| LoRa-only relay | 0.5W | 7 Wh | 2000 mAh LiPo (7.4 Wh) | 4000 mAh LiPo (14.8 Wh) |
| WiFi mesh + LoRa | 2W | 28 Wh | 8000 mAh LiPo (29.6 Wh) | 12000 mAh LiPo (44.4 Wh) |
| All radios active | 3W | 42 Wh | 12000 mAh LiPo (44.4 Wh) | 20000 mAh LiPo (74 Wh) |
| All radios + satellite WAN | 8W+ | 112+ Wh | External battery required | 40Ah LiFePO4 @ 12V (128 Wh) |

**Satellite WAN note**: Starlink Gen 3 consumes 50-75W — dramatically higher than all other components combined. Solar arrays and batteries must be sized accordingly. See Section 7 for complete Starlink configuration.

### 3.2 BGP-X Router v1 Solar Power System

The Router v1 requires an external solar charge controller since it lacks integrated MPPT:

```
Solar Panel (12V or 24V, 20-50W)
      │
      ▼
External MPPT charge controller (e.g., Victron 75/10 or Epever TRACER)
      │
      ▼
12V / 24V lead-acid or LiFePO4 battery
      │
      ▼
12V DC output → BGP-X Router v1 outdoor carrier (12V DC input)
           OR
           → 802.3at PoE injector → BGP-X Router v1
```

**Minimum solar array for Router v1 (all radios, 15W peak)**:
- Solar: 30W panel (2× safety margin over peak consumption)
- Battery: 40Ah 12V lead-acid or 20Ah LiFePO4 (night + 2-day cloudy reserve)

**Recommended solar array for Router v1**:
- Solar: 50W panel
- Battery: 60Ah 12V lead-acid or 40Ah LiFePO4

### 3.3 Third-Party Hardware Solar Systems

For GL.iNet GL-MT3000, Raspberry Pi, and similar hardware:

```
Solar Panel (12V, 20-40W)
      │
      ▼
MPPT Charge Controller (Victron 75/15 or similar)
      │
      ├──── 12V LiFePO4 battery pack (20-50Ah)
      │
      └──── 12V output → DC-DC converter → 5V 3A → Device USB-C
```

**GL.iNet GL-MT3000 power budget**:

| Mode | Consumption |
|---|---|
| Active (WiFi mesh + LoRa via USB) | ~5W |
| Active (WiFi only) | ~4W |
| Active (LoRa only) | ~1W |
| Idle (mesh) | ~2W |
| Deep sleep (if supported) | ~0.1W |

**Solar sizing for GL-MT3000 with LoRa**:
- Daily energy: 5W × 10h + 2W × 14h = 78 Wh/day
- Battery (3 days autonomy, 50% DoD): 78 × 3 / 0.5 = 468 Wh → 40Ah @ 12V
- Solar panel (4 peak sun hours, 80% efficiency): 78 / (4 × 0.8) = 24.4W → use 30W

---

## 4. Battery Chemistry Selection

### 4.1 LiPo (Lithium Polymer) — Default for Node v1

| Parameter | Value |
|---|---|
| Nominal voltage | 3.7V |
| Energy density | High (~200 Wh/kg) |
| Temperature range | -20°C to +60°C (capacity reduced below 0°C) |
| Cycle life | 300-500 cycles to 80% capacity |
| Cost | Low |
| Safety | Requires BMS; puncture risk; no venting |

LiPo is the default for BGP-X Node v1 in temperate climates. Avoid in cold climates (below -10°C) — capacity drops significantly and charging LiPo below 0°C damages the cell.

**Critical**: LiPo MUST NOT be charged below 0°C. The BGP-X Node v1 firmware includes temperature monitoring and will disable charging when battery temperature is below 0°C.

### 4.2 LiFePO4 (Lithium Iron Phosphate) — Recommended for Harsh Environments

| Parameter | Value |
|---|---|
| Nominal voltage | 3.2V (cell), 12V or 24V (pack) |
| Energy density | Moderate (~120 Wh/kg) |
| Temperature range | -30°C to +70°C (minimal capacity loss at cold) |
| Cycle life | 2000-4000 cycles to 80% capacity |
| Cost | Higher |
| Safety | Very safe; no thermal runaway |

**LiFePO4 strongly recommended for**:
- Cold climate deployments (mountain, polar, high-altitude)
- Long deployment cycles where battery access is difficult
- Higher duty-cycle operation (near-continuous radio TX)
- Any deployment expected to last more than 2-3 years without battery replacement
- Maritime and industrial environments

### 4.3 Temperature and Battery Performance

| Temperature | LiPo Remaining Capacity | LiFePO4 Remaining Capacity |
|---|---|---|
| +25°C | 100% | 100% |
| 0°C | ~80% | ~95% |
| -10°C | ~65% | ~90% |
| -20°C | ~40% | ~80% |
| -30°C | ~20% (do not charge) | ~70% |
| -40°C | ~10% (do not charge) | ~60% |

**For deployments where winter temperatures reach -20°C or below**: LiFePO4 is REQUIRED. LiPo at -20°C provides insufficient capacity for overnight operation and MUST NOT be charged.

### 4.4 Cold Weather Deployment Strategy

**LoRa radios**: operate to -40°C without issue.

**WiFi chips**: typically rated to -20°C (check manufacturer specs).

**LiFePO4 batteries**: discharge to -20°C; charge only above 0°C (unless self-heating cells).

**For extreme cold climates**:
- Insulated enclosure with thermal mass
- Battery inside insulated enclosure (not outside exposed to air)
- Low-temperature charging cutoff in charge controller
- Consider self-heating LiFePO4 cells for Arctic deployment
- Position solar panel to maximize winter sun capture (steeper tilt angle)

---

## 5. Solar Panel Selection and Mounting

### 5.1 Panel Types

**Monocrystalline silicon** (recommended):
- Highest efficiency (17-22%)
- Best performance in low light
- Best W/m² ratio
- Standard for BGP-X deployments

**Polycrystalline silicon**:
- Slightly lower efficiency (15-18%)
- Lower cost
- Acceptable if panel weight and size are not constrained

**Flexible / amorphous**:
- Lower efficiency (8-12%)
- Bendable, lightweight
- Use only when panel rigidity is impossible (curved surfaces, fabric mounting)
- Significant efficiency penalty

### 5.2 Panel Sizing by Configuration

**BGP-X Node v1 — LoRa-only relay**:
- Consumption: ~0.5W average
- Daily energy: ~12 Wh
- Panel: 3W at 6V (produces ~15 Wh on 5-hour peak sun day)
- Battery: 4000 mAh LiPo (14.8 Wh, ~1.5 days reserve)
- Cost-effective: smallest viable solar node

**BGP-X Node v1 — WiFi + LoRa community relay**:
- Consumption: ~2W average
- Daily energy: ~48 Wh
- Panel: 10W at 6V (produces ~50 Wh on 5-hour day)
- Battery: 12000 mAh LiPo (44.4 Wh, ~1 day reserve) — add second panel for margin

**BGP-X Node v1 — All radios, gateway mode**:
- Consumption: ~3-8W average
- Daily energy: ~72-192 Wh
- Panel: 20W at 12V (produces ~100 Wh on 5-hour day)
- Battery: 40Ah LiFePO4 at 12V (~128 Wh, ~1-2 days reserve)

**BGP-X Router v1 — Full operation**:
- Consumption: 10-15W average
- Daily energy: 240-360 Wh
- Panel: 40-50W at 12V
- Battery: 60Ah LiFePO4 at 12V (~256 Wh, ~1 day reserve)

**BGP-X Node v1 + Starlink satellite WAN**:
- Starlink Gen 3: 50-75W
- Total system: 55-85W average
- Daily energy: 1320-2040 Wh
- Panel: 300W minimum (ideally 400W+)
- Battery: 100Ah LiFePO4 at 24V (~2560 Wh, ~1.5 days reserve)

**Starlink with solar is the most demanding configuration**. A detailed power audit is mandatory before deployment. Starlink dish must be positioned for clear sky view, which may conflict with optimal solar panel angle — separate mounting structures are typically required.

### 5.3 Panel Orientation and Tilt

**Fixed tilt optimization**: angle from horizontal = local latitude. Maximizes annual energy yield.

| Region | Latitude | Optimal Fixed Tilt |
|---|---|---|
| Equatorial (0-15°) | 0-15° | 10-15° (mostly for rain runoff) |
| Tropical (15-30°) | 15-30° | 15-30° |
| Mediterranean (30-45°) | 30-45° | 30-45° |
| Temperate (45-55°) | 45-55° | 45-55° |
| Northern Europe / Canada (55-65°) | 55-65° | 55-65° |

**Winter-optimized tilt** (where winter energy is critical):
- Add 10-15° to the optimal annual tilt
- Captures more low-angle winter sun at the cost of some summer production
- Recommended for high-latitude deployments with significant winter load

**Azimuth**:
- Face true south (northern hemisphere)
- Face true north (southern hemisphere)
- Deviations up to 30° from optimal azimuth reduce output by <10% — acceptable
- Greater deviations cause significant energy loss

### 5.4 Shading Analysis

Before installation, assess shading at **winter sun angle** (lowest elevation):

1. Determine winter solstice sun angle: `90° - latitude - 23.5°`
2. Check for obstructions (trees, buildings, mountains) at this angle
3. Even partial shading dramatically reduces panel output
4. Consider pole mounting to clear local obstructions

---

## 6. Enclosure and Weatherproofing

### 6.1 BGP-X Node v1 Enclosure

Factory-sealed IP67 enclosure. Solar and battery connections:

**Option A: Separate battery enclosure**
- IP67 battery box mounted adjacent to Node v1
- LiFePO4 cells inside; BMS PCB inside
- 2× cable glands: solar in, DC out to Node v1
- Advantage: battery replaceable without opening Node v1

**Option B: Battery inside Node v1 enclosure**
- Flat LiPo integrated in Node v1 internal space
- Solar panel connects via waterproof cable gland
- Advantage: fewer boxes; smaller installation footprint

**Recommendation**: For deployments longer than 2 years or where battery replacement is possible, use Option A with LiFePO4.

### 6.2 BGP-X Router v1 Outdoor Enclosure

The outdoor IP67 carrier provides weatherproof housing for the PCB. Solar system components are external:

- Solar panel on separate mounting frame
- Battery in separate weatherproof box
- Charge controller in battery box or separate enclosure
- All cables enter Router enclosure via waterproof cable glands

### 6.3 Cable Weatherproofing

All outdoor cables MUST be:
- UV-stabilized jacket (not standard indoor cable)
- Rated for outdoor direct burial or aerial installation
- Terminated with waterproof connections (M12 connectors or self-amalgamating tape on SMA/RP-SMA)
- Solar panel cable: USE double-insulated (standard for solar PV)

### 6.4 Antenna Weatherproofing

- LoRa antenna: outdoor-rated omni with waterproof N-type or SMA connector
- WiFi antenna: outdoor-rated, properly sealed connections
- Apply self-amalgamating tape over all exposed connectors
- Ensure drip loops on all cables entering enclosures

---

## 7. bgpx-power Daemon — Solar-Aware Operation

The BGP-X Node v1 and Router v1 firmware includes the `bgpx-power` daemon which monitors battery state and adjusts BGP-X operation to preserve battery during low-solar-input conditions.

### 7.1 Battery Level Thresholds and Actions

| Battery Level | System Action | BGP-X Impact |
|---|---|---|
| 100-50% | Normal operation | All configured radios at full TX power |
| 50-30% | Reduce TX power | LoRa TX power reduced by 4 dBm; WiFi TX reduced by 2 dBm |
| 30-15% | Disable WiFi mesh | WiFi mesh disabled; LoRa-only relay continues |
| 15-5% | Range extension mode | Session origination disabled; relay-only; minimum TX power |
| <5% (critical) | Graceful shutdown | NODE_WITHDRAW published; 60s propagation delay; shutdown |

**Graceful shutdown at critical battery**: the bgpx-power daemon signals the bgpx-node daemon to publish a signed NODE_WITHDRAW to the DHT before shutdown. This notifies the network that the node is going offline intentionally. Other nodes remove it from routing tables immediately rather than waiting for KEEPALIVE timeout (90 seconds). This is particularly important for island gateway nodes — graceful withdrawal triggers faster path rebuild for island members.

### 7.2 Solar Recharge Recovery

When battery recovers after a low-battery event:

```
Battery > 20%: re-enable WiFi mesh (if configured)
Battery > 35%: restore normal TX power
Battery > 50%: resume normal operation including session origination
Battery = 100%: no action; normal operation continues
```

Recovery is automatic. No operator intervention required.

### 7.3 Power Management Configuration

```toml
[power_management]
enabled = true
battery_monitor = "i2c"                    # or "adc"
battery_chemistry = "lifepo4"              # or "lipo"
low_temperature_charge_cutoff_c = 0       # Disable charging below this temp

# Per-threshold actions
battery_reduce_tx_at_pct = 50
lora_tx_power_reduced_dbm = 10            # from configured maximum
wifi_disable_at_pct = 30
relay_only_mode_at_pct = 15
emergency_shutdown_at_pct = 5
graceful_shutdown_wait_secs = 60           # Time for NODE_WITHDRAW propagation
```

### 7.4 Dynamic Power Mode (Advanced Configuration)

For maximum battery life, the daemon can dynamically adjust behavior based on time of day and solar forecast (if cellular connectivity available):

```toml
[power_management.dynamic]
enabled = false                            # Requires internet connectivity
forecast_source = "open-meteo"            # Solar forecast API
conservative_mode = true                  # Assume lower production than forecast
reserve_multiplier = 1.5                  # Scale energy requirements by this factor
```

When dynamic mode is enabled:
- Reduce beacon frequency during predicted low-production periods
- Pre-emptively disable WiFi during extended cloudy periods
- Delay non-critical DHT operations until battery recovers

---

## 8. BGP-X Configuration for Solar Nodes

### 8.1 Power-Conscious Configuration

```toml
[node]
roles = ["relay", "discovery"]
# Avoid "entry" role if possible (entry requires more uptime)

[mesh]
enabled = true
transports = ["lora"]                     # LoRa only for minimal power
lora_spreading_factor = 9                  # Longer range compensates for no WiFi
beacon_interval_seconds = 60              # Reduced from 30s to save power

[extensions]
cover_traffic_enabled = false            # Disable cover traffic to save power
path_quality_reporting_enabled = true     # Lightweight, keep enabled

[store_and_forward]
enabled = true                            # Handle duty cycle limits
buffer_size_kb = 512
packet_ttl_seconds = 3600
```

### 8.2 Configuration for Gateway Mode (with WAN)

```toml
[node]
roles = ["relay", "discovery", "entry", "exit"]

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[[routing_domains]]
domain_type = "mesh"
island_id = "community-mesh"
transports = ["lora", "wifi-mesh"]

[bridge]
enabled = true
from_domain = "clearnet"
to_domain = "mesh:community-mesh"
```

### 8.3 Configuration for Satellite WAN

```toml
[[routing_domains]]
domain_type = "clearnet"
wan_type = "satellite"
satellite_provider = "starlink"
latency_class = "satellite-leo"

[exit]
enabled = true
pool = "satellite-aware-exits"
```

---

## 9. Typical Solar Deployment Configurations

### Configuration 1: Minimal LoRa Relay (Rural Coverage)

**Purpose**: Extend LoRa mesh coverage between two communities. No WAN needed — relay only.

```
Hardware:
- BGP-X Node v1 (LoRa-only mode)
- 3W 6V monocrystalline solar panel
- 4000 mAh LiFePO4 battery (cold-rated)
- IP67 junction box for battery

Mounting:
- 4m fiberglass pole in rural field
- Solar panel on mounting bracket, tilted at local latitude
- Node v1 below solar panel (shade from panel; cooler)
- Omni LoRa antenna at top of pole

Power budget:
- Consumption: 0.5W × 24h = 12 Wh/day
- Production: 3W panel × 5h sun × 0.8 = 12 Wh/day
- Surplus: ~0 Wh/day (tight)
- Battery: 14.8 Wh = ~1.5 days reserve

Recommendation: Use 6W panel for margin.

Config:
  [[routing_domains]]
  domain_type = "mesh"
  island_id = "rural-relay-chain"
  transports = ["lora"]

  [node]
  role = "relay"
  
  [performance]
  relay_only_mode = true
```

### Configuration 2: WiFi + LoRa Community Relay

**Purpose**: Standard community mesh relay with both WiFi and LoRa.

```
Hardware:
- BGP-X Node v1
- 10W 6V solar panel (or 2× 5W in parallel)
- 12000 mAh LiFePO4 battery
- Weatherproof battery box

Mounting:
- 6m mast
- Solar panel at top with optimized tilt
- Node v1 below panel
- WiFi antennas + LoRa antenna on mast

Power budget:
- Consumption: 2W × 24h = 48 Wh/day
- Production: 10W × 5h × 0.8 = 40 Wh/day
- Deficit: -8 Wh/day (insufficient)
- Battery: 44.4 Wh = ~1 day reserve

Recommendation: Use 15W panel for reliable operation.

Config:
  [[routing_domains]]
  domain_type = "mesh"
  island_id = "community-mesh"
  transports = ["lora", "wifi-mesh"]
```

### Configuration 3: Mesh Island Gateway (Off-Grid with Starlink)

**Purpose**: Remote community mesh island gateway with satellite WAN.

```
Hardware:
- BGP-X Router v1 (outdoor IP67 carrier) OR BGP-X Node v1 with WAN option
- Starlink Gen 3 terminal
- 300W 24V solar array (3× 100W panels)
- 200Ah 24V LiFePO4 battery bank (4800 Wh; ~2.5 days reserve at 80W)
- 60A MPPT solar charge controller (Victron SmartSolar 100/50)
- 24V to 12V DC-DC converter for BGP-X hardware

Mounting:
- Solar panels: ground-mounted frame at optimal tilt
- Starlink dish: clear sky view; separate pole
- BGP-X hardware: weatherproof enclosure near base
- LoRa antenna: 6m mast adjacent to enclosure
- WiFi antennas: on mast above LoRa

Power budget:
- BGP-X hardware: 8W × 24h = 192 Wh/day
- Starlink (average): 60W × 24h = 1440 Wh/day
- Total: 1632 Wh/day
- Production: 300W × 6h = 1800 Wh/day
- Surplus: 168 Wh/day (minimal margin)
- Battery: 4800 Wh = ~3 days reserve

WARNING: This configuration requires reliable sun. Backup generator or cellular WAN recommended for critical deployments.

Config:
  [[routing_domains]]
  domain_type = "clearnet"
  [clearnet_transport]
  wan_type = "satellite"
  satellite_provider = "starlink"
  latency_class = "satellite-leo"

  [[routing_domains]]
  domain_type = "mesh"
  island_id = "remote-community"
  transports = ["lora", "wifi-mesh"]
```

### Configuration 4: Tier 2 Client Node (Sensor Deployment)

**Purpose**: Low-power mesh endpoint for sensor data or personal use.

```
Hardware:
- LILYGO T3S3 or T-Echo
- 2W flexible solar panel
- 2000 mAh LiPo internal battery

Mounting:
- Solar panel attached to device case
- Device in weatherproof box
- LoRa antenna external

Power budget:
- Consumption: 0.3W × 24h = 7.2 Wh/day
- Production: 2W × 4h = 8 Wh/day
- Surplus: 0.8 Wh/day
- Battery: 7.4 Wh = ~1 day reserve

Config:
  Minimal BGP-X client firmware
  LoRa receive/transmit only
  No routing for others
```

---

## 10. Remote Monitoring

### 10.1 Monitoring via Mesh Network

Solar nodes with only LoRa connectivity have limited monitoring bandwidth:

```bash
# Low-bandwidth status check via LoRa
bgpx-cli mesh status-broadcast

# Monitor via gateway node (aggregates mesh node status)
bgpx-cli gateway mesh-status
```

### 10.2 Out-of-Band Monitoring

**Recommended for critical infrastructure**: Include a low-power cellular module (CAT-M1 or NB-IoT) for out-of-band monitoring, even when the node has no mesh connectivity.

This enables:
- Alerts when battery drops below threshold
- Remote power management commands
- Firmware updates without site visit

### 10.3 Monitoring Metrics

The bgpx-node daemon exposes these solar-specific metrics:

```
bgpx_power_battery_level_pct         - Current battery level
bgpx_power_battery_voltage_v          - Battery voltage
bgpx_power_battery_current_ma         - Battery current (positive=charging)
bgpx_power_solar_voltage_v            - Solar panel voltage
bgpx_power_solar_current_ma           - Solar panel current
bgpx_power_solar_power_w              - Instantaneous solar power
bgpx_power_mode                       - Current power mode (normal/reduced/relay_only)
bgpx_power_shutdown_events_total      - Count of graceful shutdowns
```

---

## 11. Installation Checklist

### 11.1 Pre-Installation

- [ ] Power budget calculated: consumption vs. production vs. reserve
- [ ] Battery chemistry selected for climate (LiFePO4 for <0°C)
- [ ] Solar panel tilt and azimuth calculated for site latitude
- [ ] Site survey: shading analysis at winter sun angle
- [ ] Hardware ordered including all cables, connectors, mounting
- [ ] Regulatory check: permits for installation location
- [ ] Enclosure selected for environment (IP rating, ATEX if required)

### 11.2 Installation

- [ ] Mounting structure installed and verified secure
- [ ] Solar panel mounted and oriented correctly
- [ ] Panel connections waterproofed (M12 connectors or tape)
- [ ] Battery installed in weatherproof enclosure; BMS connected
- [ ] Charge controller wired: panel → MPPT → battery → load
- [ ] BGP-X hardware connected to load output
- [ ] Power-on test: verify device boots
- [ ] All cable connections waterproofed
- [ ] Cable entry sealed with glands
- [ ] Drip loops on all outdoor cables
- [ ] Antennas installed; connectors sealed
- [ ] Solar panel tested in sunlight: verify charging current

### 11.3 Commissioning

- [ ] Battery voltage under load verified
- [ ] Solar charging current verified when panel illuminated
- [ ] BGP-X daemon started: `systemctl status bgpx-node`
- [ ] Power management daemon active: `bgpx-cli node stats | grep battery`
- [ ] Battery level reported correctly
- [ ] Test threshold actions by applying load
- [ ] Graceful shutdown verified at critical threshold
- [ ] NODE_WITHDRAW published before shutdown verified
- [ ] Recovery behavior verified: node re-joins DHT after power restoration
- [ ] bgpx-power temperature cutoff tested (if applicable)

### 11.4 Post-Deployment Monitoring

- [ ] Check at 24 hours: battery level, charging behavior
- [ ] Check at 7 days: battery level trends, solar production
- [ ] Check at 30 days: system stability
- [ ] Adjust panel tilt if production insufficient
- [ ] Review logs for unexpected shutdowns
- [ ] Verify NODE_WITHDRAW not published unexpectedly

---

## 12. Troubleshooting

### 12.1 Insufficient Power

**Symptoms**: Frequent low-battery warnings, unexpected shutdowns

**Diagnosis**:
- Check solar panel orientation and tilt
- Check for new shading (tree growth, new construction)
- Verify panel output with multimeter
- Check battery capacity with load test
- Verify charge controller settings

**Solutions**:
- Increase panel size or add second panel
- Relocate panel to unshaded location
- Replace degraded battery
- Adjust power mode configuration to reduce consumption

### 12.2 Battery Not Charging

**Symptoms**: Battery level declining despite sunlight

**Diagnosis**:
- Check solar panel connections
- Verify charge controller is operating
- Check temperature (below 0°C with LiPo)
- Check for blown fuse in charge circuit

**Solutions**:
- Repair connections
- Replace charge controller
- For cold weather: LiFePO4 required for reliable charging

### 12.3 Node Offline After Recovery

**Symptoms**: Node does not rejoin network after power restoration

**Diagnosis**:
- NODE_WITHDRAW may still be cached in DHT
- DHT bootstrap may have failed

**Solutions**:
- Verify NODE_WITHDRAW has expired (>48 hours)
- Force DHT bootstrap: `bgpx-cli dht bootstrap --force`
- Verify internet connectivity for gateway nodes

---

## 13. Maintenance Schedule

| Interval | Task |
|---|---|
| Monthly | Visual inspection; check for damage, loose connections |
| Quarterly | Clean solar panel surface; verify mounting hardware |
| Annually | Battery capacity test; connector corrosion check |
| 2-3 years (LiPo) | Battery replacement |
| 5-7 years (LiFePO4) | Battery replacement |
| As needed | Firmware updates (via cellular or site visit) |

---

## 14. Regulatory Considerations

### 14.1 Solar Installation

- Solar panel mounting may require permits in some jurisdictions
- Ground-mounted arrays over certain sizes may require planning permission
- Roof-mounted systems may have structural requirements

### 14.2 Radio Operation

- LoRa operation requires compliance with regional regulations (ISM band)
- WiFi 802.11s operates on unlicensed bands
- Outdoor antennas may have height restrictions

### 14.3 Environmental

- Battery disposal must comply with hazardous waste regulations
- Lead-acid batteries have stricter disposal requirements than LiFePO4
- Check local regulations for solar panel disposal

---

## 15. Summary

Solar deployment enables BGP-X in locations without grid power:

| Scenario | Hardware | Panel | Battery | Reserve |
|---|---|---|---|---|
| LoRa-only relay | Node v1 | 3-6W | 4000-8000 mAh LiPo | 1.5-3 days |
| WiFi + LoRa relay | Node v1 | 10-15W | 12000 mAh LiPo | 1-2 days |
| Island gateway | Node v1 or Router v1 | 20-30W | 40Ah LiFePO4 @ 12V | 1-2 days |
| Gateway + Starlink | Router v1 | 300W | 200Ah LiFePO4 @ 24V | 2-3 days |
| Client endpoint | Tier 2 | 2W | 2000 mAh LiPo | 1 day |

**Key principles**:
- Always oversize solar and battery by 50-100%
- Use LiFePO4 for cold climates and long deployments
- Configure power management thresholds appropriate to hardware
- Monitor remotely when possible
- Plan for graceful shutdown to preserve network stability

**Next steps**:
- Review `mesh_deployment.md` for community mesh architecture
- Review `mast_tower_deployment.md` for elevated installations
- Review `gateway_spec.md` for provider infrastructure requirements
