# BGP-X Satellite Architecture

**Version**: 0.1.0-draft

---

## 1. Executive Summary

| Satellite Type | BGP-X Domain | Notes |
|---|---|---|
| Starlink, Iridium, Inmarsat, HughesNet, Viasat | **Clearnet (0x00000001)** | Internet access over satellite; BGP-routed; BGP-X overlay runs on top |
| Future BGP-X Satellite Network | **bgpx-satellite (0x00000005)** | Reserved for future BGP-X-native satellite infrastructure; NOT ACTIVE |

**Commercial satellite internet services are clearnet.** They provide IP connectivity via BGP-routed ground station infrastructure. BGP-X treats them identically to fiber or cellular — same clearnet domain type (0x00000001), different latency class annotation. Domain type 0x00000005 is reserved for a future BGP-X-native satellite network and should NOT be used in current deployments.

---

## 2. How Commercial Satellite Internet Works

### 2.1 The Path of a Starlink Packet

```
User device (on ground)
    │
    ↓ 802.11 WiFi to Starlink terminal (router)
Starlink terminal
    │
    ↓ Ku-band radio signal uplink
Starlink LEO satellite (altitude: ~550 km)
    │
    ↓ (Option A: Inter-satellite laser link) → other Starlink satellites
    ↓ (Option B: direct downlink to ground station)
Starlink ground station / PoP
    │
    ↓ Fiber to internet exchange
BGP-routed internet
    │
    ↓ BGP routing to destination
Destination server
```

Every Starlink packet traverses BGP-routed internet infrastructure. From the moment a packet leaves the Starlink ground station, it travels the same BGP-routed internet as any fiber user's traffic. Starlink provides an alternative **first-mile** to the internet; the rest of the internet is identical.

The same applies to Iridium, Inmarsat, HughesNet, Viasat, and all current commercial satellite internet services. They differ in:
- **Orbit type** (LEO, MEO, GEO) → different latency characteristics
- **Bandwidth** (50-200 Mbps for Starlink down to 512 Kbps for some GEO services)
- **Coverage** (polar, global, regional)

**Critical fact**: None of these services provide direct terminal-to-terminal communication that bypasses the internet. All inter-user communication on these networks goes through BGP-routed ground infrastructure.

---

## 3. Satellite as Clearnet in BGP-X

### 3.1 Configuration

A BGP-X Router v1 or BGP-X Node v1 with Starlink WAN is configured with a clearnet domain exactly as with fiber:

```toml
[[routing_domains]]
domain_type = "clearnet"
endpoints = [
  { protocol = "udp", address = "0.0.0.0", port = 7474 }
]

# Satellite latency class declaration (informational — affects geo plausibility + congestion thresholds)
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
satellite_orbit = "leo"          # leo, meo, or geo
latency_class = "satellite-leo"  # 20-40ms RTT to internet nodes
```

The BGP-X daemon treats this node as a clearnet relay node. Its WAN IP is a routable internet IP (provided by Starlink). Other BGP-X nodes on the internet can reach it via UDP/7474 as with any clearnet relay.

### 3.2 Latency Classes

| Provider | Orbit | Typical RTT to Ground Station | BGP-X Latency Class |
|---|---|---|---|
| Starlink Gen 2/3 | LEO (550 km) | 20-40 ms | satellite-leo |
| Amazon Kuiper (planned) | LEO (630 km) | 25-45 ms | satellite-leo |
| OneWeb | LEO (1,200 km) | 30-60 ms | satellite-leo |
| O3b mPOWER | MEO (8,000 km) | 100-150 ms | satellite-meo |
| Inmarsat GEO | GEO (35,786 km) | 550-650 ms | satellite-geo |
| Viasat / HughesNet | GEO (35,786 km) | 550-650 ms | satellite-geo |
| Iridium NEXT (data) | LEO (780 km) | 30-60 ms | satellite-leo |

### 3.3 Geographic Plausibility

**All satellite clearnet nodes are exempt from geographic plausibility RTT scoring.**

Reason: satellite nodes may have IP addresses assigned from a ground station in a different region than the terminal's physical location. A terminal in the Amazon basin may have an IP assigned from a ground station in Miami. Geo plausibility scoring based on RTT to IP region would be incorrect for such nodes.

The geo plausibility module returns `0.5` (neutral) for any node with `latency_class = satellite-*`. This never penalizes or rewards — the score has no effect.

**Important context**: Geographic plausibility scoring is an **OPTIONAL feature for ALL BGP-X nodes**, not just satellite nodes.

- Nodes are NOT required to declare a jurisdiction in their advertisement
- If a node does NOT declare jurisdiction: geo plausibility scoring does NOT apply
- If a node DOES declare jurisdiction: geo plausibility scoring applies, measuring RTT plausibility against the claimed region
- Satellite nodes: even if jurisdiction is declared, geo plausibility scoring is EXEMPT (returns neutral 0.5 score)

Declaring jurisdiction is an opt-in privacy/convenience tradeoff. The network functions identically with or without it.

---

## 4. Can Satellite Networks Form a BGP-X Mesh?

### 4.1 With Current Commercial Satellites

**No.** Current commercial satellite services do not support direct terminal-to-terminal communication without traversing internet infrastructure.

When two Starlink users communicate:
1. User A → uplink to Starlink satellite A
2. Starlink satellite A → (if available: ISL to satellite B) or downlink to ground station
3. Ground station → internet → ground station
4. Starlink satellite B → downlink to User B

Even with Starlink's inter-satellite laser links (ISL), traffic still routes through ground station infrastructure for terminal-to-terminal communication. The ISLs are internal Starlink routing — not accessible to end users for direct communication.

A BGP-X mesh domain requires that nodes communicate directly with each other at the BGP-X protocol level, with BGP-X's own routing and onion encryption. Commercial satellite networks route all traffic through their own infrastructure and BGP — there is no mechanism to bypass this for mesh communication.

**Two Starlink terminals communicating via BGP-X overlay are both clearnet nodes.** They communicate via the clearnet domain (their traffic goes up to satellite, down to ground station, through internet, back to satellite, down to other terminal). BGP-X overlay runs on top of this — providing privacy — but this is not a mesh domain. It is two clearnet nodes with high-latency clearnet connections.

### 4.2 With Satellite Constellations Offering Raw Inter-Satellite Links

If a satellite provider were to offer APIs for direct inter-satellite packet routing bypassing ground stations — and if such packets could carry arbitrary protocol data including BGP-X protocol — then a satellite mesh domain would technically be possible. No current provider offers this.

---

## 5. Future: BGP-X Satellite Network

BGP-X reserves domain type `0x00000005` for a future **BGP-X-native satellite network**.

### 5.1 Vision

A BGP-X Satellite Network consists of:

**BGP-X Satellite Nodes**: satellites in LEO orbit running BGP-X relay node software. Each satellite has:
- BGP-X relay daemon (stripped-down for embedded space compute)
- BGP-X protocol radio (custom Ka/Ku/V-band radio for ground-to-satellite communication)
- Inter-satellite BGP-X links: optical or radio links carrying BGP-X protocol between satellites
- Solar power (in orbit)
- The satellite IS a BGP-X relay node — it participates in the unified DHT, forwards encrypted onion layers, maintains sessions

**BGP-X Satellite Terminal** (ground-side):
- User or community device with directional satellite radio
- BGP-X satellite transport driver (handles satellite radio protocol)
- Connects to BGP-X Satellite Node above
- Acts as a domain bridge between satellite-mesh domain and clearnet or terrestrial mesh

**BGP-X Satellite Domain**:
```
Domain type: 0x00000005
Instance hash: BLAKE3(orbit_id)[0:4]
Example orbit_id: "bgpx-leo-1"
Domain ID: 0x00000005-BLAKE3("bgpx-leo-1")[0:4]
```

**Cross-domain paths involving BGP-X satellites**:
```
Clearnet client → clearnet relays → domain bridge → [satellite mesh hops] → satellite exit → clearnet destination
```

A BGP-X satellite hop provides complete geographic separation — a traffic correlation adversary observing ground-level internet cannot see the satellite portion of the path. The satellite relays are in international airspace with no single jurisdictional control.

### 5.2 Why This Matters

**Today**: BGP-X paths stay on earth. Adversaries can observe traffic at internet exchange points, submarine cable taps, and ground-based network infrastructure.

**With BGP-X Satellites**: a path can route through satellite relays in orbit, completely bypassing earth-based network observation infrastructure for that portion of the path. This provides a qualitatively different level of traffic correlation resistance.

### 5.3 Technical Requirements for Future BGP-X Satellites

- Custom compact satellite platform: BGP-X relay daemon requires ~128 MB RAM and 500 MHz ARM — achievable with rad-tolerant embedded ARM systems
- Custom satellite radio: BGP-X satellite transport protocol (not current commercial protocols)
- Inter-satellite links: optical (preferred) or RF; carrying BGP-X protocol directly
- Onboard key management: HSM in space; signed node advertisements updated via ground control
- Launch: constellation of ≥30 satellites for global coverage with reasonable hop gaps
- Ground terminals: software-defined radio terminals with BGP-X satellite transport driver

### 5.4 Current Status

BGP-X Satellite is a long-term vision. Domain type `0x00000005` is reserved in the protocol. Current implementations should:
- Not use domain type `0x00000005` for any current deployment (reserved)
- Treat any node advertising domain type `0x00000005` as invalid (unverifiable without BGP-X satellite infrastructure)
- Leave protocol extension hooks in place for satellite transport driver registration

When BGP-X Satellite infrastructure launches, domain type `0x00000005` will be activated with a published domain specification update.

---

## 6. Satellite and Mesh Islands — Clearnet Bridge

The most practical satellite+mesh integration today: a BGP-X Node v1 in a remote location uses satellite WAN (Starlink) as its clearnet domain and simultaneously participates in a local LoRa/WiFi mesh island.

```
Remote mesh island (e.g., Amazon basin)
         │
   [BGP-X Node v1]
   LoRa mesh radio: connected to local mesh island
   Starlink WAN: internet via satellite
         │
    [Starlink terminal]
         │
    [LEO satellite uplink]
         │
    [BGP-X clearnet relay nodes, globally]
         │
    [Rest of BGP-X network]
```

In this configuration:
- The Node v1 is a domain bridge node: bridges `clearnet` (via Starlink) to `mesh:<island_id>` (via LoRa)
- Clearnet clients worldwide can reach the mesh island's services via this bridge
- Mesh island members can access clearnet through this bridge's exit (if exit enabled)
- The Starlink link has ~40ms additional latency vs. fiber — acceptable for BGP-X overlay use
- The Node v1 advertises `wan_type = satellite` in its clearnet domain endpoint metadata — path construction uses satellite-calibrated latency thresholds

This is the correct and complete satellite integration for today: satellite as a high-latency clearnet WAN for remote domain bridge nodes, not as a mesh domain.

---

## 7. Satellite Deployment Considerations

### 7.1 Starlink (Recommended for Remote BGP-X Deployment)

- **Gen 3 terminal**: USB-C port provides USB Ethernet; BGP-X daemon detects `0x48bf:0x0800` (vendor ID for Starlink USB adapter) and loads satellite transport driver automatically
- **Latency**: 25-40ms to Starlink PoP; realistic BGP-X path RTT via Starlink: 100-200ms total for 4 hops
- **Throughput**: 50-200 Mbps down, 5-20 Mbps up; more than sufficient for BGP-X relay duties
- **Power**: Starlink Gen 3 dish: 50-75W peak; significant for solar deployments (require 200W+ panel for sustained operation)
- **Obstruction**: requires clear sky view; not suitable for dense forest or below-cliff deployments
- **Cost**: hardware ~$300-500; service ~$100-150/month depending on region and plan

### 7.2 Iridium Certus (Truly Global, Including Poles)

- **Coverage**: every point on Earth including poles
- **Latency**: LEO orbit but ground-station routing: 60-120ms effective
- **Bandwidth**: 88 Kbps to 704 Kbps (Certus 100 to Certus 700)
- **BGP-X use**: limited throughput; suitable for BGP-X control plane (DHT, advertisements) and low-bandwidth user traffic; not suitable for high-throughput relay
- **Power**: low (modem: 5-10W)
- **Use case**: polar expeditions, maritime in extreme conditions, disaster response where Starlink is unavailable

### 7.3 Inmarsat / BGAN

- **Coverage**: near-global (latitude-limited GEO)
- **Latency**: 600ms+ (GEO)
- **Bandwidth**: 32-650 Kbps
- **BGP-X use**: BGP-X overlay functions but high GEO latency means all paths through this node add 600ms+; acceptable for some use cases, not for interactive traffic
- **Use case**: maritime, aviation, extreme remote where Starlink unavailable

### 7.4 LoRa + Satellite Combination (Recommended for Off-Grid Islands)

The recommended architecture for a truly remote mesh island with satellite backhaul:

- BGP-X Node v1 as satellite WAN + LoRa bridge node
- Solar powered (200W+ panel for Starlink; 6W panel for Iridium)
- BGP-X Router v1 units as island relay nodes (PoE from solar array)
- One bridge node handles all inter-island and clearnet traffic via satellite
- All intra-island traffic (LoRa/WiFi) operates fully offline without satellite dependency

---

## 8. HTTP Protocol for .bgpx Services Over Satellite

BGP-X native services (.bgpx addresses) use **HTTP/2 over BGP-X streams**.

This choice is particularly important for satellite-connected paths:

- **HTTP/2 multiplexing** allows fetching multiple resources over a single BGP-X stream
- **Each round-trip on satellite** adds 20-600ms depending on orbit
- **HTTP/2 eliminates redundant round-trips** for multi-resource pages and APIs
- **HTTP/3 is NOT used** within the BGP-X overlay because:
  - BGP-X already provides reliable ordered delivery at the session layer
  - HTTP/3's QUIC would add redundant reliability and congestion control layers

**HTTP/3 at exit nodes**: When a BGP-X exit node connects to a clearnet HTTP/3 server, it uses HTTP/3 over TLS over the exit's clearnet connection. This is standard HTTP/3, not over BGP-X streams. The exit negotiates HTTP/3 with the destination normally.

**For LoRa + satellite deployments** (remote mesh islands with satellite backhaul), HTTP/2's multiplexing is especially valuable — multiple resources can be fetched over a single BGP-X stream, reducing round-trips on both the high-latency satellite segment and the high-latency LoRa segment.

---

## 9. Protocol Impact Summary

Satellite clearnet nodes participate in BGP-X protocol identically to fiber/cellular clearnet nodes:

| Property | Satellite Clearnet Node | Fiber/Cellular Clearnet Node |
|---|---|---|
| Domain type | 0x00000001 (clearnet) | 0x00000001 (clearnet) |
| Node advertisement format | Identical | Identical |
| Handshake protocol | Identical (HANDSHAKE_INIT/RESP/DONE) | Identical |
| Onion encryption | Identical (ChaCha20-Poly1305) | Identical |
| Unified DHT participation | Identical | Identical |
| Pool participation | Identical | Identical |
| Reputation events | Identical (geo plausibility exempt) | Identical (geo plausibility applies if jurisdiction declared) |
| PATH_QUALITY_REPORT | Identical (domain_id = clearnet) | Identical |
| Latency thresholds | Satellite-calibrated | Standard internet thresholds |

### 9.1 Unified DHT Participation

Satellite clearnet nodes participate in the **same unified DHT** as all other BGP-X nodes:

- **DHT queries and storage**: identical to any clearnet node
- **No separate "satellite DHT"**: one unified DHT spanning all routing domains
- **Mesh-only nodes** behind a satellite-connected domain bridge access the unified DHT via the bridge node's cache

### 9.2 N-Hop Unlimited

Satellite paths have **no protocol-level maximum hop count**. Paths traversing satellite-connected clearnet nodes are subject to the same N-hop unlimited policy as all BGP-X paths:

- No global enforcement maximum
- Configurable defaults exist (recommended: 4 hops for single-domain clearnet; 6-8 for cross-domain)
- Paths may traverse satellite nodes any number of times in any combination
- Long paths with satellite segments are valid — the only practical limit is latency budget

---

## 10. Satellite Terminal Hardware Integration

### 10.1 Starlink Gen 3 Integration

**USB Detection**: The Starlink Gen 3 terminal provides a USB-C port that presents as a USB Ethernet adapter. BGP-X daemon detects this via USB vendor ID (`0x48bf:0x0800`) and loads the satellite transport driver.

**Configuration**: The terminal appears as a standard network interface (e.g., `eth1`). BGP-X treats it as the WAN interface for clearnet domain endpoints. No special satellite configuration is required beyond the `wan_type = satellite` metadata.

**Latency Class Auto-Detection**: The BGP-X daemon can detect Starlink latency characteristics and automatically set `latency_class = satellite-leo` based on observed RTT patterns.

### 10.2 Iridium Certus Integration

**Interface**: Serial modem interface (USB-Serial or RS-232). BGP-X satellite transport driver handles AT commands for connection establishment.

**Configuration**:
```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "iridium"
satellite_orbit = "leo"
latency_class = "satellite-leo"
serial_device = "/dev/ttyUSB0"
baud_rate = 115200
```

**Bandwidth Constraints**: Iridium Certus bandwidth (88 Kbps to 704 Kbps) limits throughput. BGP-X daemon adapts path quality expectations and congestion thresholds for low-bandwidth satellite connections.

### 10.3 Inmarsat BGAN/FBB Integration

**Interface**: Similar serial/IP interface to Iridium.

**Configuration**:
```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "inmarsat"
satellite_orbit = "geo"
latency_class = "satellite-geo"
```

**GEO Latency Implications**: 600ms+ RTT means all interactive protocols experience noticeable delay. BGP-X overlay encryption adds minimal overhead. Suitable for non-interactive traffic and control plane operations.

---

## 11. Power and Environmental Considerations

### 11.1 Starlink Power Requirements

| Component | Power Draw |
|---|---|
| Starlink Gen 3 dish (active) | 50-75W |
| Starlink Gen 3 dish (idle/heater) | 20-30W |
| BGP-X Router v1 or Node v1 | 5-15W |
| **Total (satellite + BGP-X)** | **55-90W peak** |

**Solar Sizing**: For sustained 24/7 operation with Starlink:
- Minimum panel capacity: 200W (clear sky, favorable latitude)
- Recommended panel capacity: 300-400W
- Battery capacity: 200-400 Wh (LiFePO4 recommended)

### 11.2 Iridium Certus Power Requirements

| Component | Power Draw |
|---|---|
| Iridium Certus modem (active) | 5-10W |
| Iridium Certus modem (standby) | 1-2W |
| BGP-X Client Node | 0.5-1W |
| **Total (Iridium + BGP-X)** | **6-11W peak** |

**Solar Sizing**: For sustained operation with Iridium:
- Minimum panel capacity: 6W (clear sky)
- Recommended panel capacity: 10-15W
- Battery capacity: 20-50 Wh

### 11.3 Environmental Protection

Satellite terminals require:
- **Clear sky view** for satellite line-of-sight
- **Weather protection** for terminal electronics (IP-rated enclosures)
- **Temperature management** for extreme climates (heaters for cold, ventilation for hot)

BGP-X Router v1 outdoor carrier (IP67) combined with Starlink's weather-resistant terminal provides a complete outdoor solution.

---

## 12. Satellite Domain Type Clarification

### 12.1 Why Satellite Internet is NOT Domain Type 0x00000005

Domain type `0x00000005` is reserved for a future **BGP-X-native satellite network** where:

- Satellites themselves run BGP-X relay software
- Inter-satellite links carry BGP-X protocol directly
- Ground terminals act as domain bridges to the satellite mesh domain
- Traffic can route through satellite relays without touching ground-based internet infrastructure

**Current commercial satellite services** (Starlink, Iridium, Inmarsat, HughesNet, Viasat):

- Provide **BGP-routed internet connectivity** over satellite radio
- All traffic traverses ground station infrastructure and the public internet
- From BGP-X's perspective, this is **clearnet domain (0x00000001)** — just with higher latency
- No BGP-X protocol runs on the satellites themselves
- No direct satellite-to-satellite BGP-X routing

### 12.2 Implementation Requirements

Current BGP-X implementations MUST:

1. **Use domain type 0x00000001** for all satellite internet connections
2. **Set latency_class** to `satellite-leo`, `satellite-meo`, or `satellite-geo` based on observed latency
3. **NOT use domain type 0x00000005** until BGP-X Satellite infrastructure is deployed
4. **Reject any node advertisement** claiming domain type 0x00000005 (unverifiable without BGP-X Satellite infrastructure)

---

## 13. Summary

| Aspect | Status |
|---|---|
| **Current satellite internet** | Clearnet domain (0x00000001); high-latency clearnet transport |
| **Domain 0x00000005** | Reserved for future BGP-X-native satellite network; NOT ACTIVE |
| **Geo plausibility** | Exempt for satellite nodes; OPTIONAL feature globally |
| **HTTP/2 for .bgpx** | Recommended for satellite paths; reduces round-trips |
| **Unified DHT participation** | Identical to other clearnet nodes |
| **N-hop unlimited** | No hop cap; configurable defaults |
| **Starlink integration** | USB auto-detect; satellite-leo latency class |
| **Iridium integration** | Serial AT commands; satellite-leo latency class |
| **Inmarsat integration** | Serial/IP; satellite-geo latency class |
| **Power requirements** | Significant for Starlink (50-75W); low for Iridium (5-10W) |
| **BGP-X Satellite future** | Vision for orbital BGP-X relay nodes; long-term roadmap |

BGP-X treats satellite internet exactly as it treats fiber or cellular: clearnet domain, standard protocol participation, with latency-aware thresholds. The future BGP-X Satellite vision represents a fundamentally different capability — BGP-X routing in orbit — but requires dedicated infrastructure that does not exist today.
