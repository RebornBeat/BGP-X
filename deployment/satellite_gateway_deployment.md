# Satellite Deployment

**Version**: 0.1.0-draft
**Status**: Finalized Specification

---

## 1. Overview

BGP-X satellite gateway deployment uses commercial satellite internet services (Starlink, Iridium, Inmarsat, etc.) as the WAN (clearnet) connection for BGP-X Router v1 or Node v1 units. This enables BGP-X coverage in locations without fiber or cellular connectivity: remote rural areas, islands, polar regions, deserts, maritime environments, and disaster-response zones.

**Critical understanding**: Satellite internet services are **clearnet** in BGP-X — domain type 0x00000001, the same as fiber or cellular. They provide BGP-routed IP connectivity. BGP-X overlay runs on top of this connectivity identically to any other WAN. A BGP-X unit with Starlink WAN is a clearnet relay node with satellite-class latency — not a "satellite domain" node.

**Domain type 0x00000005 (bgpx-satellite)** is RESERVED for a future BGP-X-native satellite network where satellites would be BGP-X relay nodes with inter-satellite links carrying BGP-X protocol directly. This is NOT currently active. Any node advertising domain type 0x00000005 in current deployments MUST be rejected as unverifiable.

For the full architectural explanation of satellite integration, see `docs/satellite_architecture.md`.

---

## 2. Deployment Scenarios

### Scenario 1: Remote Mesh Island Gateway (Starlink WAN)

```
Remote location (no fiber/cellular):
[BGP-X Router v1 or Node v1]
  ├── LoRa mesh: local mesh island nodes
  ├── WiFi 802.11s: local mesh nodes within range
  └── Starlink USB → Starlink terminal → satellite → internet → BGP-X network

Result: Remote mesh island connected to clearnet BGP-X overlay via satellite
```

The BGP-X device is a domain bridge: clearnet (via Starlink, which is clearnet) ↔ mesh (via LoRa/WiFi). Clearnet clients worldwide can reach the remote island's BGP-X services. Island members can reach clearnet via the satellite-clearnet exit path.

### Scenario 2: Maritime Vessel Relay

```
Vessel at sea:
[BGP-X Router v1] → [Starlink terminal] → satellite → internet
  ├── WiFi mesh: crew devices and other BGP-X nodes on vessel
  └── LoRa: shore-range LoRa relay (within ~20km of coast)

Result: Moving BGP-X relay node with satellite clearnet WAN; participates in global overlay while at sea
```

### Scenario 3: Polar / Extreme Remote (Iridium Certus)

```
Antarctic research station:
[BGP-X Node v1] → [Iridium Certus 100] → LEO satellites → gateway → internet
  └── LoRa: local station mesh island

Result: BGP-X overlay connectivity at global extremes; ~88-704 Kbps clearnet bandwidth
```

Note: Iridium Certus bandwidth (88-704 Kbps) limits BGP-X relay throughput. The node can still participate in DHT, publish advertisements, and forward BGP-X sessions — but session count will be low (5-50 depending on traffic volume).

### Scenario 4: Disaster Response Temporary Gateway

```
Disaster area (terrestrial infrastructure down):
[BGP-X Router v1] → [Starlink] → internet → BGP-X global overlay
  ├── WiFi 802.11s: responder devices, pop-up access points
  └── LoRa: wide-area LoRa mesh for coordination across km-scale areas

Result: BGP-X overlay restored in disaster area within minutes of Starlink terminal deployment
```

---

## 3. Supported Satellite Services

| Service | Latency | Download | Upload | Mobility | Notes |
|---|---|---|---|---|---|
| Starlink (Standard) | 20-40ms | 50-200 Mbps | 10-20 Mbps | Stationary | Best performance |
| Starlink Maritime | 20-40ms | 50-250 Mbps | 10-20 Mbps | Maritime | IP67 hardware |
| Starlink RV | 20-40ms | 25-100 Mbps | 2-10 Mbps | Mobile | Best for vehicle |
| Starlink Portability | 20-60ms | 5-100 Mbps | 2-20 Mbps | Mobile (limitations) | Area-specific |
| Iridium Certus | 300-500ms | 22 Mbps | 8.5 Mbps | Global | Extreme locations |
| Inmarsat Fleet | 400-600ms | 4-10 Mbps | 1-4 Mbps | Maritime | Legacy maritime |
| Viasat | 600ms | 12-100 Mbps | 3-10 Mbps | Stationary | GEO; high latency |
| HughesNet | 600ms | 15-25 Mbps | 3 Mbps | Stationary | GEO; FUP limits |
| OneWeb | 30-50ms | 50-200 Mbps | 10-20 Mbps | Stationary | LEO |
| Kuiper | 20-40ms | 50-200 Mbps | 10-20 Mbps | Stationary | LEO |

---

## 4. Hardware Configuration

### 4.1 BGP-X Router v1 with Starlink Gen 3

**Hardware needed**:
- BGP-X Router v1 (any carrier; outdoor IP67 if weatherproofing needed)
- Starlink Gen 3 terminal (circular dish; USB-C port on terminal)
- USB-A to USB-C cable (or USB-C hub)
- LoRa and/or WiFi antennas for mesh domain

**Connection**:
```
Starlink terminal USB-C → BGP-X Router v1 USB-A port
```

The BGP-X daemon detects the Starlink USB device (vendor ID `0x48bf`) at startup and loads the satellite transport driver automatically. No manual IP configuration needed — Starlink terminal operates as USB Ethernet adapter (DHCP).

**config.toml**:
```toml
[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
satellite_orbit = "leo"
latency_class = "satellite-leo"

[[routing_domains]]
domain_type = "mesh"
island_id = "your-remote-island"
transports = ["lora", "wifi-mesh"]
```

### 4.2 BGP-X Node v1 with Starlink (Off-Grid)

**Hardware needed**:
- BGP-X Node v1 (outdoor IP67; solar-powered)
- Starlink Gen 3 terminal (standalone power or shared solar array)
- Solar array: minimum 200W panel + 100Ah LiFePO4 battery for Node v1 + Starlink combined

**Power budget**:
- Starlink Gen 3: 50-75W operating (significant!)
- BGP-X Node v1: 2-8W depending on radio config
- Total: 55-85W sustained
- Requires: 200W+ solar panel; 100Ah+ LiFePO4 battery for overnight operation

This is the most demanding off-grid BGP-X deployment. Careful power budgeting is essential.

### 4.3 BGP-X Node v1 with Iridium Certus (Extreme Remote)

**Hardware needed**:
- BGP-X Node v1
- Iridium Certus 100 or Certus 350 modem (serial interface)
- USB-to-serial adapter (if using USB port on Node v1)
- Directional antenna for Iridium (LHCP circular polarization; clear sky view required)

**Bandwidth planning for Iridium Certus**:
- Certus 100: 88 Kbps up / 704 Kbps down
- BGP-X DHT participation: ~5-10 Kbps sustained
- BGP-X relay: 1-5 sessions practical at Certus 100 bandwidth
- Suitable for: DHT sync, node advertisement, light relay duty; NOT suitable for high-traffic relay

**config.toml for Iridium**:
```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "iridium-certus"
satellite_orbit = "leo"
latency_class = "satellite-leo"

# Bandwidth limit to avoid saturating Iridium link
[performance]
max_bandwidth_kbps = 50    # Leave headroom; Iridium is metered
```

### 4.4 BGP-X Router v1 with Inmarsat BGAN/FBB (Maritime)

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "inmarsat"
satellite_orbit = "geo"
latency_class = "satellite-geo"

# GEO satellite: 600ms+ RTT; adjust congestion baselines
# Path quality reports via this clearnet will show high RTT — expected, not congestion
```

---

## 5. Software Configuration

### 5.1 General Satellite Gateway Configuration

```toml
[node]
roles = ["relay", "entry", "exit", "discovery"]

[network]
listen_addr = "0.0.0.0"
listen_port = 7474
# Starlink uses CGNAT — configure STUN to discover public IP
use_stun = true
stun_server = "stun.bgpx.network:3478"

[sessions]
# Satellite has higher latency; extend timeouts
session_idle_timeout_seconds = 300     # 5 minutes (Starlink: 20-40ms RTT; GEO: 600ms+)
keepalive_interval_seconds = 60        # Less frequent to conserve bandwidth

[advertisement]
is_gateway = true
link_quality_profiles.satellite.latency_class = 2   # LEO (Starlink)
# link_quality_profiles.satellite.latency_class = 3 # GEO (Viasat, HughesNet)
bandwidth_mbps = 50   # Conservative for shared satellite

[exit_policy]
logging_policy = "none"
dns_mode = "doh"
ech_capable = true

[reputation]
geo_plausibility_enabled = false  # Satellite location changes; disable
```

### 5.2 Starlink CGNAT Handling

Starlink uses CGNAT (Carrier-Grade NAT) — no direct public IPv4. Solutions:

**Option A: STUN (simplest)**
```toml
[network]
use_stun = true
stun_server = "stun.bgpx.network:3478"
```

BGP-X daemon uses STUN to discover the public IP/port NAT maps to. Advertises this in DHT. Works for most CGNAT configurations.

**Option B: Relay via known public node**
```toml
[network]
relay_via = "public-node-nodeid@public-ip:7474"
```

Traffic punches through CGNAT via relay. Adds hop but guarantees reachability.

**Option C: IPv6 (Starlink provides native IPv6)**
```toml
[network]
prefer_ipv6 = true
listen_addr_v6 = "::"
public_addr_v6 = ""  # Auto-detected from Starlink IPv6 prefix
```

Starlink provides /56 IPv6 prefix. BGP-X on IPv6 avoids CGNAT entirely.

### 5.3 Bandwidth Conservation

Satellite bandwidth costs money or has data caps. Configure BGP-X conservatively:

```toml
[bandwidth]
total_bandwidth_mbps = 20   # Reserve 30 Mbps for non-BGP-X satellite use

[extensions]
cover_traffic_enabled = false   # Disable on metered connections

[sessions]
max_sessions = 200   # Limit sessions to conserve bandwidth
```

**Cover traffic on satellite paths**: Cover traffic minimum rate for satellite-clearnet segments uses `MIN_COVER_RATE_BY_DOMAIN["clearnet"] = 10 Kbps`. For metered Iridium connections, operators can reduce this:

```toml
[cover_traffic]
enabled = true
target_rate_kbps = 5           # Lower than default for metered satellite
min_rate_satellite_kbps = 2    # Absolute minimum for satellite-clearnet
```

### 5.4 Failover Configuration

For critical deployments: primary terrestrial + satellite failover

```toml
[network]
wan_primary = "eth0"        # Terrestrial (fiber/cellular)
wan_backup = "eth1"         # Satellite (Starlink)
failover_enabled = true
failover_check_interval_seconds = 60
failover_rtt_threshold_ms = 500  # Switch to backup if primary RTT > 500ms
```

On failover to satellite:
- BGP-X daemon continues operation
- Paths rebuild (satellite endpoint advertised in DHT)
- Latency increases (20-40ms Starlink; 600ms GEO)
- Clients with path quality monitoring detect latency change and may rebuild paths

---

## 6. Installation

### 6.1 Starlink Dish Installation

```
1. Check sky obstruction (Starlink app: "Check for Obstructions")
2. Mount dish on pole mount or roof mount
   - Elevation: minimum 1m clearance in 100° arc
   - Away from metal roofing/gutters (reduce reflections)
3. Connect power cable (supplied, 60W)
4. Connect Ethernet from Starlink router to BGP-X router WAN
5. Configure BGP-X router with DHCP on WAN (Starlink DHCP)
6. Verify connectivity: bgpx-cli dht stats
```

### 6.2 Cold Climate Considerations

Starlink dish has built-in heating — rated to -30°C. BGP-X router may need insulated enclosure below -10°C. Keep router above 0°C for LiFePO4 battery charging if solar-supplemented.

### 6.3 Installation Checklist — Remote Satellite Gateway

- [ ] BGP-X Router v1 or Node v1 configured with correct island_id and satellite WAN type
- [ ] Satellite terminal powered and online; BGP-X device detects satellite interface (`bgpx-cli domains list` shows clearnet domain)
- [ ] Satellite latency class configured correctly in config.toml
- [ ] Mesh radios active: `bgpx-cli domains list` shows mesh domain
- [ ] Bridge pairs visible: `bgpx-cli domains bridges --health` shows clearnet↔mesh pair
- [ ] MESH_ISLAND_ADVERTISE published to unified DHT: `bgpx-cli islands show --self --dht-freshness`
- [ ] DOMAIN_ADVERTISE published: `bgpx-cli domains bridges --self --dht-freshness`
- [ ] Cross-domain connectivity test from clearnet client passes: `bgpx-cli islands connectivity-test <island_id> --from-clearnet`
- [ ] For Starlink: dish obstruction check; signal > -100 dBm
- [ ] For Iridium: directional antenna aimed; link established
- [ ] Power budget verified: solar panel, battery, Node v1, and satellite terminal within array capacity
- [ ] Second bridge node configured (avoid single-bridge failure) — especially important for remote islands where physical access for repair is difficult

---

## 7. Operational Considerations

### 7.1 Geographic Plausibility — Satellite Exempt

All BGP-X nodes with `latency_class = satellite-*` are **exempt from geographic plausibility RTT scoring**. The geo plausibility scorer returns `0.5` (neutral) for these nodes.

This means:
- Satellite clearnet nodes will NOT be penalized for RTT that appears inconsistent with their claimed region
- A Starlink terminal in an Amazon jungle with a ground station IP in Miami will not appear geo-implausible
- Satellite nodes are otherwise scored normally for all other reputation events

No configuration needed — the daemon applies satellite exemption automatically based on `latency_class`.

### 7.2 Satellite-Aware BGP-X Path Construction

When a satellite-clearnet node is selected as a relay in a BGP-X path, the path construction algorithm uses satellite-calibrated latency thresholds:

```
Path quality score for satellite hop:
- latency_score: uses satellite-leo/geo baseline (400ms or 2000ms) vs. standard 200ms baseline
- MILD congestion trigger: measured_rtt > 2.0 × satellite_baseline
- SEVERE congestion trigger: measured_rtt > 4.0 × satellite_baseline
```

This prevents false congestion detection on high-latency satellite links.

---

## 8. Satellite Gateway as Mesh Island Bridge

The most valuable satellite deployment: a remote mesh island with a BGP-X Node v1 or Router v1 using satellite WAN as its bridge to the clearnet BGP-X network.

**Privacy properties** (same as any domain bridge):
- Island members who access clearnet via this bridge: exit node sees only last clearnet relay's address — not satellite terminal's IP, not island member's identity
- Clearnet clients reaching island services via this bridge: the bridge sees only the last clearnet relay's address
- The satellite provider (Starlink) sees encrypted BGP-X UDP traffic from the terminal — cannot read content or identify routing

**Operational considerations for satellite bridge**:
- Publish `bridge_latency_ms` accurately: satellite LEO path adds ~40ms to bridge latency; satellite GEO adds ~600ms
- Set appropriate `available = true/false` based on satellite link health
- Monitor for satellite outages (Starlink: weather, dish obstruction; Iridium: generally more reliable)
- For Starlink: dish orientation matters; re-aim if RSSI drops

---

## 9. Performance Expectations

### BGP-X over Starlink (LEO Satellite)

| Metric | Expected |
|---|---|
| Path construction (4 hops via satellite) | 500-2000ms |
| Relay throughput (through satellite gateway) | 20-50 Mbps |
| Added latency per hop (satellite WAN) | 20-40ms (Starlink) |
| Session reliability | High (Starlink stable; some dropouts) |

### BGP-X over GEO Satellite

| Metric | Expected |
|---|---|
| Path construction (4 hops via satellite) | 2000-5000ms |
| Relay throughput | 2-8 Mbps |
| Added latency per hop | 600ms+ |
| Session reliability | Medium (weather-sensitive) |

### Comparative Latency by WAN Type

| Clearnet WAN Type | Typical BGP-X Path RTT (4 hops) | Interactive Use | Web Browsing |
|---|---|---|---|
| Fiber | 100-300ms | Excellent | Excellent |
| Cellular (4G/5G) | 150-400ms | Good | Good |
| Starlink (LEO) | 200-500ms | Good | Good |
| Iridium Certus (LEO) | 300-600ms | Acceptable | Acceptable |
| Inmarsat GEO | 1500-3000ms | Degraded | Slow but functional |
| HughesNet/Viasat GEO | 1500-3000ms | Degraded | Slow but functional |

BGP-X Browser handles high-latency paths (including GEO satellite paths) with the same latency-tolerant rendering used for LoRa paths — batched resources, progressive rendering, latency indicator. GEO satellite paths are usable for .bgpx browsing; response times are measured in seconds rather than milliseconds.

**Recommendation**: For BGP-X gateway use, Starlink strongly preferred over GEO satellites. GEO latency makes path construction slow and keepalive management difficult.
