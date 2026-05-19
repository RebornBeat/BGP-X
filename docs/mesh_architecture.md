# BGP-X Mesh Architecture

**Version**: 0.1.0-draft

---

## 1. Mesh as a First-Class Routing Domain

BGP-X mesh networking is not an add-on. Mesh is a first-class routing domain with equal status to clearnet. It:

- Has its own type identifier in the domain ID wire format (type 0x00000003)
- Is addressable from any other routing domain
- Has services that are discoverable from clearnet via the unified DHT
- Can originate paths to any other routing domain
- Operates independently when internet connectivity is unavailable
- Scales from small community networks to island chains spanning continents

This document describes the mesh architecture: island model, transports, discovery, cross-domain connectivity, and how BGP-X unifies the mesh networking ecosystem.

---

## 2. Satellite is Not Mesh

A mesh island gateway using Starlink as its WAN connection is a domain bridge node that bridges the clearnet domain (via Starlink) to the mesh domain (via LoRa or WiFi radio). The satellite provides the clearnet side of the bridge. See `satellite_architecture.md` for the full satellite discussion.

---

## 3. The Mesh Island Model

A **mesh island** is a named collection of BGP-X nodes communicating via radio transport. Islands are:

**Named**: each island has an `island_id` string (operator-chosen, globally unique via DHT collision detection, e.g., `lima-san-isidro-north-2026`).

**Addressable**: a domain_id derived as `0x00000003-BLAKE3(island_id)[0:4]`.

**DHT-registered**: when a bridge node is online, the island publishes MESH_ISLAND_ADVERTISE records to the unified DHT. Clearnet clients can discover the island and its services.

**Self-contained**: when no bridge node is online, the island operates with a local mesh DHT bootstrapped via MESH_BEACON broadcast. Intra-island traffic continues with full privacy guarantees.

**Reachable from clearnet**: clearnet clients with no mesh hardware can reach island services via domain bridge nodes that handle all radio transmission on their behalf.

---

## 4. Mesh Transports

BGP-X is transport-agnostic. The onion encryption and routing logic are identical regardless of transport. Mesh domain nodes select the transport that best serves each relay hop based on peer reachability, latency, and bandwidth.

### WiFi 802.11s Mesh

```
Protocol:  IEEE 802.11s (infrastructure-independent mesh)
MTU:       1280 bytes
Bandwidth: 10-100+ Mbps per hop
Latency:   1-20ms per hop
Range:     50-200m indoor; up to kilometers with directional antennas
Use case:  Primary transport for dense urban deployments; high-bandwidth backhaul
```

WiFi 802.11s enables multi-hop mesh without any access point or infrastructure. Every BGP-X node with a compatible WiFi chip can participate. The MT7915 WiFi 6 chip (reference for BGP-X Router v1) supports 802.11s natively via Linux mac80211.

### LoRa Radio

```
Protocol:  Raw LoRa (NOT LoRaWAN — BGP-X uses raw LoRa for protocol flexibility)
Frequency: 868 MHz (EU), 915 MHz (NA), 433 MHz (Asia)
Data rate: 250 bps (SF12) to ~37 Kbps (SF5 BW500)
Range:     5-50 km rural; 1-5 km urban
Latency:   100ms-5s per hop (SF and duty cycle dependent)
Fragmentation: REQUIRED (MESH_FRAGMENT 0x19); MTU 200 bytes at SF7
Use case:  Long-range coverage; rural mesh; gap-bridging between WiFi clusters
```

**LoRa and browsing**: LoRa paths introduce significant latency (hundreds of milliseconds to seconds per hop). Standard web browsers designed for <100ms responsiveness will provide a poor experience on LoRa paths. **However, browsing on LoRa paths is fully supported and functional with the right client.**

BGP-X Browser and BGP-X-native applications are designed to handle LoRa-class latency gracefully:
- Batched resource requests (critical path resources fetched together in one round-trip)
- Progressive rendering (render HTML skeleton immediately; fill content as it arrives)
- Latency-aware UI (shows estimated load time; never shows blank white page)
- Aggressive caching (long TTLs for .bgpx resources; reduce repeat fetches)
- Prioritized resource loading (HTML and CSS first; images last)

This is similar to Tor Browser's approach to .onion browsing — latency is real and noticeable, but the browser adapts to make it workable. A BGP-X user browsing a .bgpx service via a LoRa path is in the same position as a Tor user browsing a .onion service via a slow exit: it works, it is just slower.

**LoRa and EU duty cycle**: EU ETSI regulations limit LoRa transmitters to 1% duty cycle (~36 seconds transmit per hour at SF7). BGP-X firmware enforces this via a token bucket. When the duty cycle budget is exhausted, streams entering or using the LoRa segment enter `PAUSED` state; they resume when the duty cycle resets. This is a hard regulatory constraint, not a protocol limitation.

**LoRa is NOT suitable for high-throughput applications** (video streaming, large file transfer) due to its intrinsic bandwidth limits. It IS fully suitable for: web browsing, messaging, email, API calls, and other interactive applications that tolerate 1-10 second round-trip times.

### Bluetooth BLE

```
Protocol:  BGP-X BLE transport over GATT
Bandwidth: 1-2 Mbps (DLE extended)
Range:     10-100m per hop
Latency:   10-100ms per hop
Fragmentation: REQUIRED (MESH_FRAGMENT for packets >244 bytes)
Use case:  Short-range IoT device mesh; mobile device peering; indoor dense deployment
```

BLE enables very compact BGP-X nodes (phones, small sensors, ESP32 devices) to participate in the mesh. At current data rates, BLE is well-suited for interactive applications but not high-bandwidth transfers.

### Ethernet Point-to-Point

```
Bandwidth: 100 Mbps to 10 Gbps
Range:     100m copper; 100km+ fiber
Latency:   <1ms
Use case:  High-bandwidth fixed backbone between mesh nodes; apartment building wiring
```

Ethernet P2P treated as mesh transport (not clearnet) when the link is a direct BGP-X node-to-node connection not routed through the internet. Example: two BGP-X Router v1 units connected by a direct Ethernet cable form a two-node mesh segment.

---

## 5. Unified DHT for Mesh

BGP-X operates **one unified DHT**. There is no separate mesh DHT. The previous two-DHT model (internet DHT + mesh DHT with gateway synchronization) is replaced by a unified approach.

### How Mesh-Only Nodes Participate

When a bridge node is online (internet-connected):
- Bridge node stores mesh-domain node advertisements in the unified internet DHT
- Clearnet clients can discover mesh island relay nodes via domain-filtered DHT queries
- Mesh-only nodes access the unified DHT via their bridge nodes' caches

When no bridge node is online (offline mode):
- Island operates with local mesh DHT bootstrapped via MESH_BEACON broadcast
- No records published to internet DHT (not internet-accessible until bridge reconnects)
- When bridge reconnects: pending records published automatically

### Bootstrap via MESH_BEACON

In mesh-only mode, new nodes bootstrap via broadcast beacons:

```
1. Broadcast MESH_BEACON on all active mesh transports (every 30s ± 5s jitter)
2. Receive MESH_BEACONs from nearby nodes
3. Verify Ed25519 signatures on all received beacons
4. Add verified nodes to local DHT routing table
5. Query new neighbors via DHT_FIND_NODE for more nodes
6. Bootstrap complete when routing table has ≥5 verified entries
```

**MESH_BEACON format**:
```json
{
  "type": "bgpx_mesh_beacon",
  "node_id": "a3f2b9c1...",
  "public_key": "<base64-encoded Ed25519 public key>",
  "transports_supported": ["wifi_mesh", "lora"],
  "dht_routing_table_size": 45,
  "timestamp": "2026-04-24T12:00:00Z",
  "bridge_capable": false,
  "served_domains": ["mesh:lima-district-1"],
  "signature": "<base64-encoded Ed25519 signature>"
}
```

All fields are signed. The `bridge_capable` and `served_domains` fields indicate whether this node can bridge to other routing domains.

---

## 6. Domain Bridge Nodes (Gateways)

Domain bridge nodes connect the mesh island domain to other routing domains (usually clearnet). They are the enabling component for cross-domain access.

### What a Bridge Node Does

1. Receives clearnet UDP packets destined for the mesh island
2. Decrypts the DOMAIN_BRIDGE onion layer (its own layer only)
3. Records cross-domain path_id routing (source_clearnet_addr ↔ target_mesh_addr)
4. Forwards remaining encrypted payload into the mesh island via radio
5. Receives return traffic from the mesh island
6. Routes return traffic back to the clearnet predecessor via cross-domain path_id lookup
7. Publishes MESH_ISLAND_ADVERTISE to unified internet DHT when online

### BGP-X Node v1 as Gateway

The BGP-X Node v1 with WAN Ethernet is a fully capable mesh island gateway. It requires no additional hardware beyond the WAN cable. Configuration:

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[[routing_domains]]
domain_type = "mesh"
island_id = "my-island"
transports = ["lora", "wifi-mesh"]
```

With this configuration, the Node v1 bridges clearnet to the mesh island. Community members deploy one or two Node v1 devices with WAN connections as their island's gateway nodes, and additional Node v1 devices (no WAN needed) as coverage-extending relay nodes throughout the island.

### Multiple Bridge Nodes Per Island

A mesh island with only ONE bridge node has a single point of failure. If that bridge goes offline, the island loses all cross-domain connectivity.

**Minimum recommended**: 2 independent bridge nodes per island, operated by different operators in different physical locations and ASNs.

BGP-X path construction automatically selects between available bridge nodes. Client-side operator diversity enforcement prevents two bridge nodes from the same operator from appearing in the same path.

---

## 7. Clearnet Client Accessing Mesh Service

A clearnet client with no mesh hardware can reach a service inside a mesh island.

```
Clearnet client (no mesh radio, standard internet connection)
         │ UDP/7474
         ▼
[Clearnet relay nodes — standard onion encryption]
         │ UDP/7474
         ▼
[Domain Bridge Node — clearnet UDP + mesh radio]
  Decrypts DOMAIN_BRIDGE onion layer
  Stores cross-domain path_id routing
  Forwards to mesh island via radio
         │ WiFi 802.11s or LoRa or BLE
         ▼
[Mesh relay nodes]
         │ mesh transport
         ▼
[Mesh service]
```

Privacy properties:
- Clearnet relays see only: client IP (entry node) or relay-to-relay traffic (middle relays)
- Bridge node sees: last clearnet relay's address + first mesh relay's address
- Bridge node does NOT see: client IP or service identity
- Mesh relays see: only their immediate neighbors
- Mesh service sees: only last mesh relay's address
- No single node in any domain can link client to service

---

## 8. Why Existing Mesh Systems Failed to Scale

Many mesh networking projects have been built. None achieved widespread adoption because each solves only part of the problem:

### Meshtastic

**What it does well**: LoRa mesh hardware, large community, demonstrated range.

**What's missing**: text messages only; no privacy (messages visible to all relays); no clearnet exit; no general IP routing; no anonymity; flooding-based routing.

**BGP-X adds**: full onion encryption over LoRa via adaptation layer. DHT-based discovery. Path selection. Clearnet access via gateway bridge nodes. Cross-island routing.

### GoTenna / ATAK

**What it does well**: LoRa mesh for emergency/tactical use.

**What's missing**: proprietary; no privacy from the vendor; no clearnet routing; expensive.

**BGP-X adds**: Open alternative using same class of hardware. Full privacy from any network observer.

### Helium Network

**What it does well**: LoRaWAN coverage network with economic incentives.

**What's missing**: all traffic visible to network operators; no onion routing; IoT sensor data only; cannot route general internet traffic.

**BGP-X adds**: Privacy layer over LoRa transport. BGP-X can use Helium hardware via adaptation layer (Helium hotspots can be flashed with BGP-X modem firmware).

### NYC Mesh / Freifunk / LibreMesh

**What they do well**: community WiFi mesh providing internet access; run on commodity OpenWrt hardware.

**What's missing**: no privacy (traffic visible to mesh operators); no anonymity; communities isolated from each other; no inter-community federation.

**BGP-X adds**: privacy overlay on existing OpenWrt hardware (same devices, no hardware change). Cross-community routing via unified DHT and domain bridge nodes.

### Reticulum Network Stack

**What it does well**: multi-transport mesh (LoRa, WiFi, serial); designed for low-bandwidth links.

**What's missing**: no onion routing; routing metadata visible to all relays; no clearnet exit model; no hardware ecosystem.

**BGP-X adds**: multi-hop onion routing on top of multi-transport capability. Clearnet exit via bridge nodes. Pool trust domains. Hardware ecosystem.

### The Unification BGP-X Provides

| Feature | Meshtastic | NYC Mesh | Tor | Reticulum | BGP-X |
|---|---|---|---|---|---|
| LoRa transport | ✅ | ❌ | ❌ | ✅ | ✅ |
| WiFi mesh | ❌ | ✅ | ❌ | ✅ | ✅ |
| Onion routing | ❌ | ❌ | ✅ | ❌ | ✅ |
| Clearnet exit | ❌ | ✅ | ✅ | ❌ | ✅ |
| Anonymity | ❌ | ❌ | ✅ | ❌ | ✅ |
| LoRa browsing | ❌ | ❌ | ❌ | ❌ | ✅ (latency-tolerant) |
| No ISP needed | ✅ | ❌ | ❌ | ✅ | ✅ |
| Inter-community routing | ❌ | ❌ | ❌ | ❌ | ✅ |
| Clearnet↔mesh cross-domain | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 9. Mesh Deployment Scenarios

### Scenario 1: Pure Mesh (No Internet)

```
[BGP-X Node v1 A] ── LoRa ── [BGP-X Node v1 B] ── WiFi ── [BGP-X Router v1 C]
```

All nodes are mesh-only. Zero ISP dependency. Intra-island communications fully private. No clearnet access. DHT bootstrapped via MESH_BEACON beacons.

Any mix of BGP-X hardware works: Router v1, Node v1, and OpenWrt package installs all participate equally in the mesh island.

### Scenario 2: Mesh with Single Gateway (BGP-X Node v1)

```
[Node v1 A] ── LoRa ── [Node v1 B] ── WiFi ── [Node v1 C (WAN)] ── ISP ── Internet
```

Node v1 C has WAN Ethernet and acts as gateway. All mesh users share Node v1 C's ISP connection. Individual mesh users have zero direct ISP exposure.

### Scenario 3: Multi-Gateway Redundancy

```
                    [Gateway: Node v1 EU] → EU ISP
[Mesh Network] ──► [Gateway: Node v1 NA] → US ISP
                    [Gateway: Router v1]  → Local ISP
```

Three gateways. If one goes offline, traffic automatically routes via others.

### Scenario 4: Satellite-WAN Gateway

```
[Mesh Island — Rural] ←── LoRa mesh ──→ [Node v1 (Starlink WAN)] → Starlink → Internet
```

BGP-X Node v1 with Starlink WAN bridges the rural mesh island to clearnet. Starlink provides the clearnet side (satellite internet = clearnet domain). The LoRa radio provides the mesh side. This is a standard domain bridge configuration — Starlink is just one way to provide the clearnet WAN.

### Scenario 5: Multi-Island via Clearnet Bridge

```
[Island A: Lima]  ←──── clearnet BGP-X overlay ────→  [Island B: Bogotá]
      │                                                        │
  Gateway A              [internet relays]                 Gateway B

Path: User(A) → mesh hops → Gateway A → internet relays → Gateway B → mesh hops → User(B)
```

The unified DHT cross-domain synchronization (performed by gateway nodes) makes Island B discoverable from Island A. Path construction is automatic.

---

## 10. Elevated Relay Deployment

Elevated BGP-X Node v1 deployments dramatically extend mesh coverage.

### Coverage by Height (LoRa line-of-sight, SX1262 +14 dBm, SF9)

| Height | LoRa Range (Clear Terrain) |
|---|---|
| Ground level (2m) | 2-5 km |
| Rooftop (10-15m) | 5-15 km |
| Mast (30m) | 10-30 km |
| Tower (60m+) | 20-60 km |

### Fixed Mast (Permanent Mesh Coverage)

BGP-X Node v1 in outdoor IP67 carrier mounted at height. PoE-powered via Cat6 cable from building. No aviation authority involvement at typical heights (under 30m in most jurisdictions). See `deployment/mast_tower_deployment.md` for installation details.

### Solar-Powered Remote Node

BGP-X Node v1 with integrated solar charge controller and LiFePO4 battery. 10W solar panel + 10 Ah LiFePO4 provides reliable 24/7 operation in locations with ≥4 hours daily sun. Zero grid power dependency. IP67 enclosure with built-in solar charging circuitry. See `deployment/solar_deployment.md` for solar sizing.

### Tethered Aerostat (Temporary Wide Coverage)

A weather balloon or aerostat carries a BGP-X relay payload at elevation. Power via tether cable. At 150m: LoRa line-of-sight radius ~45km — covers an entire small city.

Regulatory note: tethered aerostats have different (generally more permissive) regulations than free-flying drones in most jurisdictions. Under 60m is typically unregulated or notify-only in most regions.

Use case: temporary wide coverage during events or disaster response. Not ideal for permanent deployment (helium refill, weather limitations). See `deployment/aerial_deployment.md`.

### Drone Relay (Emergency Only)

A drone carrying a BGP-X relay payload provides temporary elevated coverage.

**Honest assessment**: drones are not practical for persistent relay infrastructure. Limited flight time (30-60 minutes), regulatory limits (120-150m AGL in most jurisdictions), and need for continuous operation make drones suitable only for emergency temporary deployment.

For persistent coverage: use fixed masts or tethered aerostats.

### Vehicle Nodes

BGP-X Router v1 or Node v1 in vehicle (bus, delivery truck, emergency vehicle). Serves as a mobile relay that extends mesh coverage and synchronizes DHT records as it travels. A bus route passing through rural areas naturally connects isolated mesh nodes.

The "traveling mesh" effect: vehicles driven regularly naturally extend and maintain mesh coverage, synchronize DHT, and connect isolated mesh islands dynamically.

---

## 11. Security Considerations

### Radio Interception

LoRa and WiFi signals can be received by anyone within range. All BGP-X overlay traffic is encrypted with ChaCha20-Poly1305. Even if radio traffic is captured, no plaintext is accessible. The transport adds no security surface to the BGP-X crypto model.

### Rogue Mesh Nodes

An adversary could introduce a node with a forged advertisement. Defense: all node advertisements must have valid Ed25519 signatures (including in MESH_BEACON). Nodes with invalid signatures are rejected. The reputation system penalizes nodes with anomalous behavior.

### Gateway Compromise

If a gateway/bridge node is compromised: the adversary can see destinations of clearnet traffic (same as any exit node compromise) and may block mesh-to-internet access. Defense: multiple gateways in different locations and jurisdictions; pool-based selection; path rebuilding when gateway becomes unresponsive.

### Mesh Island Isolation

An adversary could disrupt all bridge nodes connecting an island to other domains. This is an availability failure, not a privacy failure: intra-island communications continue with full privacy; only cross-domain connectivity is affected. Defense: multiple independent bridge nodes from different operators.

---

## 12. Integration with Existing Community Networks

### NYC Mesh / Freifunk / LibreMesh

These communities run BGP-X-compatible hardware (OpenWrt routers). No hardware changes required.

**Migration path**:
1. Install BGP-X as OpenWrt package on existing routers
2. Configure existing mesh as a BGP-X pool with mesh island ID
3. Deploy one or more Node v1 or Router v1 units as gateway/bridge nodes
4. Community members' internet traffic routes through BGP-X overlay privately
5. Other communities adopt BGP-X → inter-community routing via unified DHT

### Meshtastic Communities

**Integration path**:
1. Connect Meshtastic device to BGP-X Router v1 or Node v1 via USB
2. Flash BGP-X modem firmware to ESP32/nRF52840 (replaces Meshtastic routing protocol)
3. BGP-X router handles all routing, encryption, and DHT using the LoRa radio
4. Existing Meshtastic users gain: full onion encryption, clearnet access, cross-island reach

The existing Meshtastic hardware serves as radio infrastructure. BGP-X provides the privacy and routing layer on top. See `hardware/meshtastic_adapter.md`.
