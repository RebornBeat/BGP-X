# BGP-X Satellite Architecture

**Version**: 0.1.0-draft

---

## 1. Summary

| Satellite Type | BGP-X Domain | Notes |
|---|---|---|
| Starlink, Iridium, Inmarsat, HughesNet, Viasat | **Clearnet** | Internet access over satellite; BGP-routed; BGP-X overlay runs on top |
| Future BGP-X Satellite Network | **bgpx-satellite (domain type 0x00000005)** | Reserved; requires BGP-X-native satellite hardware not yet built |

Satellite internet services are clearnet. They provide IP connectivity via BGP-routed ground station infrastructure. BGP-X treats them identically to fiber or cellular — same clearnet domain, different latency class.

---

## 2. How Commercial Satellite Internet Works

### The Path of a Starlink Packet

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
- Orbit type (LEO, MEO, GEO) → different latency
- Bandwidth (50 Mbps for Starlink down to 512 Kbps for some GEO services)
- Coverage (polar, global, regional)

None of them provide direct terminal-to-terminal communication that bypasses the internet. All inter-user communication on these networks goes through BGP-routed ground infrastructure.

---

## 3. Satellite as Clearnet in BGP-X

### Configuration

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

### Latency Classes

| Provider | Orbit | Typical RTT to Ground Station | BGP-X Latency Class |
|---|---|---|---|
| Starlink Gen 2/3 | LEO (550 km) | 20-40 ms | satellite-leo |
| Amazon Kuiper (planned) | LEO (630 km) | 25-45 ms | satellite-leo |
| OneWeb | LEO (1,200 km) | 30-60 ms | satellite-leo |
| O3b mPOWER | MEO (8,000 km) | 100-150 ms | satellite-meo |
| Inmarsat GEO | GEO (35,786 km) | 550-650 ms | satellite-geo |
| Viasat / HughesNet | GEO (35,786 km) | 550-650 ms | satellite-geo |
| Iridium NEXT (data) | LEO (780 km) | 30-60 ms | satellite-leo |

### Geographic Plausibility

**All satellite clearnet nodes are exempt from geographic plausibility RTT scoring.**

Reason: satellite nodes may have IP addresses assigned from a ground station in a different region than the terminal's physical location. A terminal in the Amazon basin may have an IP assigned from a ground station in Miami. Geo plausibility scoring based on RTT to IP region would be incorrect for such nodes.

The geo plausibility module returns `0.5` (neutral) for any node with `latency_class = satellite-*`. This never penalizes or rewards — the score has no effect.

---

## 4. Can Satellite Networks Form a BGP-X Mesh?

### With Current Commercial Satellites

**No.** Current commercial satellite services do not support direct terminal-to-terminal communication without traversing internet infrastructure.

When two Starlink users communicate:
1. User A → uplink to Starlink satellite A
2. Starlink satellite A → (if available: ISL to satellite B) or downlink to ground station
3. Ground station → internet → ground station
4. Starlink satellite B → downlink to User B

Even with Starlink's inter-satellite laser links (ISL), traffic still routes through ground station infrastructure for terminal-to-terminal communication. The ISLs are internal Starlink routing — not accessible to end users for direct communication.

A BGP-X mesh domain requires that nodes communicate directly with each other at the BGP-X protocol level, with BGP-X's own routing and onion encryption. Commercial satellite networks route all traffic through their own infrastructure and BGP — there is no mechanism to bypass this for mesh communication.

**Two Starlink terminals communicating via BGP-X overlay are both clearnet nodes.** They communicate via the clearnet domain (their traffic goes up to satellite, down to ground station, through internet, back to satellite, down to other terminal). BGP-X overlay runs on top of this — providing privacy — but this is not a mesh domain. It is two clearnet nodes with high-latency clearnet connections.

### With Satellite Constellations Offering Raw Inter-Satellite Links

If a satellite provider were to offer APIs for direct inter-satellite packet routing bypassing ground stations — and if such packets could carry arbitrary protocol data including BGP-X protocol — then a satellite mesh domain would technically be possible. No current provider offers this.

---

## 5. Future: BGP-X Satellite Network

BGP-X reserves domain type `0x00000005` for a future **BGP-X-native satellite network**.

### Vision

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

### Why This Matters

Today: BGP-X paths stay on earth. Adversaries can observe traffic at internet exchange points, submarine cable taps, and ground-based network infrastructure.

With BGP-X Satellites: a path can route through satellite relays in orbit, completely bypassing earth-based network observation infrastructure for that portion of the path. This provides a qualitatively different level of traffic correlation resistance.

### Technical Requirements for Future BGP-X Satellites

- Custom compact satellite platform: BGP-X relay daemon requires ~128 MB RAM and 500 MHz ARM — achievable with rad-tolerant embedded ARM systems
- Custom satellite radio: BGP-X satellite transport protocol (not current commercial protocols)
- Inter-satellite links: optical (preferred) or RF; carrying BGP-X protocol directly
- Onboard key management: HSM in space; signed node advertisements updated via ground control
- Launch: constellation of ≥30 satellites for global coverage with reasonable hop gaps
- Ground terminals: software-defined radio terminals with BGP-X satellite transport driver

### Current Status

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

### Starlink (Recommended for Remote BGP-X Deployment)

- Gen 3 terminal: USB-C port provides USB Ethernet; BGP-X daemon detects `0x48bf:0x0800` and loads satellite transport driver automatically
- Latency: 25-40ms to Starlink PoP; realistic BGP-X path RTT via Starlink: 100-200ms total for 4 hops
- Throughput: 50-200 Mbps down, 5-20 Mbps up; more than sufficient for BGP-X relay duties
- Power: Starlink Gen 3 dish: 50-75W peak; significant for solar deployments (require 200W+ panel for sustained operation)
- Obstruction: requires clear sky view; not suitable for dense forest or below-cliff deployments
- Cost: hardware ~$300-500; service ~$100-150/month depending on region and plan

### Iridium Certus (Truly Global, Including Poles)

- Coverage: every point on Earth including poles
- Latency: LEO orbit but ground-station routing: 60-120ms effective
- Bandwidth: 88 Kbps to 704 Kbps (Certus 100 to Certus 700)
- BGP-X use: limited throughput; suitable for BGP-X control plane (DHT, advertisements) and low-bandwidth user traffic; not suitable for high-throughput relay
- Power: low (modem: 5-10W)
- Use case: polar expeditions, maritime in extreme conditions, disaster response where Starlink is unavailable

### Inmarsat / BGAN

- Coverage: near-global (latitude-limited GEO)
- Latency: 600ms+ (GEO)
- Bandwidth: 32-650 Kbps
- BGP-X use: BGP-X overlay functions but high GEO latency means all paths through this node add 600ms+; acceptable for some use cases, not for interactive traffic
- Use case: maritime, aviation, extreme remote where Starlink unavailable

### LoRa + Satellite Combination (Recommended for Off-Grid Islands)

The recommended architecture for a truly remote mesh island with satellite backhaul:

- BGP-X Node v1 as satellite WAN + LoRa bridge node
- Solar powered (200W+ panel for Starlink; 6W panel for Iridium)
- BGP-X Router v1 units as island relay nodes (PoE from solar array)
- One bridge node handles all inter-island and clearnet traffic via satellite
- All intra-island traffic (LoRa/WiFi) operates fully offline without satellite dependency

---

## 8. Protocol Impact Summary

Satellite clearnet nodes participate in BGP-X protocol identically to fiber/cellular clearnet nodes:
- Same node advertisement format (clearnet domain)
- Same HANDSHAKE_INIT/RESP/DONE protocol
- Same onion encryption
- Same unified DHT participation
- Same reputation system (geo plausibility exempt, all other events scored normally)
- Same pool participation
- Same PATH_QUALITY_REPORT (domain_id = clearnet domain ID; latency bucket calibrated to satellite-class)

No protocol changes are needed to support satellite WAN nodes. Only configuration changes (latency_class declaration) and daemon-level transport detection (USB vendor ID auto-detect) are required.
