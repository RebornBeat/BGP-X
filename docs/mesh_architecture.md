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

## 2. The Mesh Island Model

A **mesh island** is a named collection of BGP-X nodes communicating via radio transport. Islands are:

**Named**: each island has an `island_id` string (operator-chosen, globally unique via DHT collision detection, e.g., `lima-san-isidro-north-2026`).

**Addressable**: a domain_id derived as BLAKE3(island_id)[0:4] as the instance hash (type 0x00000003 + instance = 8-byte domain ID).

**DHT-registered**: when a bridge node is online, the island publishes MESH_ISLAND_ADVERTISE records to the unified DHT. Clearnet clients can discover the island and its services.

**Self-contained**: when no bridge node is online, the island operates with a local mesh DHT bootstrapped via MESH_BEACON broadcast. Intra-island traffic continues with full privacy guarantees.

**Reachable from clearnet**: clearnet clients with no mesh hardware can reach island services via domain bridge nodes that handle all radio transmission on their behalf.

---

## 3. Why Existing Mesh Systems Have Failed to Scale

Many mesh networking projects have been built. None achieved widespread adoption because each solves only part of the problem:

### Meshtastic

**What it does well**: LoRa mesh hardware, large community, demonstrated range.

**What's missing**: text messages only; no privacy (messages visible to all relays); no clearnet exit; no general IP routing; no anonymity; flooding-based routing.

**BGP-X adds**: full onion encryption over LoRa via adaptation layer. DHT-based discovery. Path selection. Clearnet access via gateway bridge nodes. Cross-island routing.

### GoTenna / ATAK

**What it does well**: LoRa mesh for emergency/tactical use.

**What's missing**: proprietary; no privacy from the vendor; no clearnet routing; expensive.

### Helium Network

**What it does well**: LoRaWAN coverage network with economic incentives.

**What's missing**: all traffic visible to network operators; no onion routing; IoT sensor data only; cannot route general internet traffic.

### NYC Mesh / Freifunk / LibreMesh

**What they do well**: community WiFi mesh providing internet access; run on commodity OpenWrt hardware.

**What's missing**: no privacy (traffic visible to mesh operators); no anonymity; communities isolated from each other; no inter-community federation.

**BGP-X adds**: privacy overlay on existing OpenWrt hardware (same devices, no hardware change). Cross-community routing via unified DHT and domain bridge nodes. Communities discover and route to each other through the overlay.

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
| No ISP needed | ✅ | ❌ | ❌ | ✅ | ✅ |
| Inter-community routing | ❌ | ❌ | ❌ | ❌ | ✅ |
| Clearnet↔mesh cross-domain | ❌ | ❌ | ❌ | ❌ | ✅ |
| Unified DHT (all domains) | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 4. Transport Types

BGP-X is transport-agnostic. The onion encryption and routing logic are identical regardless of transport.

### WiFi 802.11s Mesh

```
Protocol:  IEEE 802.11s mesh networking
MTU:       1280 bytes
Bandwidth: 10-100+ Mbps per hop
Latency:   1-20ms per hop
Range:     50-200m indoor; km with directional antennas
Use case:  Primary transport for dense deployments
```

### LoRa Radio

```
Protocol:  Raw LoRa (not LoRaWAN)
Frequency: 868 MHz (EU), 915 MHz (NA), 433 MHz
Bandwidth: 0.3-50 Kbps (SF-dependent)
Range:     5-50 km rural; 1-5 km urban
Latency:   100ms-5s per hop (duty cycle limited)
Fragmentation: REQUIRED (MESH_FRAGMENT 0x19)
Use case:  Long-range low-bandwidth; DHT sync; key exchange
```

LoRa is NOT suitable for general web browsing. It is ideal for DHT synchronization, key exchange, and control messages. BGP-X path selection treats LoRa hops as high-latency links and excludes them from interactive traffic paths unless configured otherwise.

EU duty cycle: 1% → ~36 seconds transmit per hour at SF7.

### Bluetooth BLE

```
Bandwidth: 1-2 Mbps
Range:     10-100m per hop
Latency:   10-100ms
Use case:  Short-range IoT device mesh; mobile pairing
Fragmentation: REQUIRED
```

### Ethernet Point-to-Point

```
Bandwidth: 100 Mbps to 10 Gbps
Range:     100m copper; 100km+ fiber
Latency:   <1ms
Use case:  High-bandwidth backbone between fixed locations
```

---

## 5. Unified DHT for Mesh

BGP-X operates **one unified DHT**. There is no separate mesh DHT. The previous two-DHT model (internet DHT + mesh DHT with gateway synchronization) is replaced.

### How Mesh-Only Nodes Participate

When a bridge node is online (internet-connected):
- Bridge node stores mesh-domain node advertisements in the unified internet DHT
- Clearnet clients can discover mesh island relay nodes via domain-filtered DHT queries
- Mesh-only nodes access the unified DHT via their bridge nodes' caches

When no bridge node is online (offline mode):
- Island operates with local mesh DHT bootstrapped via MESH_BEACON broadcast
- No records published to internet DHT (not internet-accessible)
- When bridge reconnects: pending records published automatically

### Bootstrap via MESH_BEACON

In mesh-only mode, new nodes bootstrap via broadcast beacons:

```
1. Broadcast MESH_BEACON on all active mesh transports (every 30s ± 5s jitter)
2. Receive MESH_BEACONs from nearby nodes
3. Verify Ed25519 signatures on all received beacons
4. Add verified nodes to local DHT routing table
5. Query new neighbors via DHT_FIND_NODE for more nodes
6. Bootstrap complete when routing table has ≥ 5 verified entries
```

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

### Multiple Bridge Nodes Per Island (IMPORTANT)

A mesh island with only ONE bridge node has a single point of failure. If that bridge goes offline, the island loses all cross-domain connectivity.

**Minimum recommended**: 2 independent bridge nodes per island, operated by different operators in different physical locations and ASNs.

BGP-X path construction automatically selects between available bridge nodes. Client-side operator diversity enforcement prevents two bridge nodes from the same operator from appearing in the same path.

### DHT Cross-Domain Synchronization (Simplified from Prior Model)

Bridge nodes are not "synchronizing two DHTs." They are participating in the unified DHT on behalf of mesh-only nodes:

- **Mesh → unified DHT**: When a mesh-only node publishes a record (via MESH_BEACON or DHT_PUT via relay), the bridge node stores that record in the unified internet DHT
- **Unified DHT → mesh**: When a clearnet client wants to discover a mesh island node, it queries the unified DHT; bridge nodes have stored the relevant records

No synchronization algorithm. No two DHTs. One DHT, with bridge nodes providing access for mesh-only members.

---

## 7. Clearnet Client Accessing Mesh Service

This is the key capability that prior mesh systems lack.

**What the clearnet client needs**: BGP-X daemon installed. No mesh hardware. No BGP-X router.

**What happens**:

```
Clearnet client (no mesh radio, standard internet connection)
         │ UDP/7474
         ▼
[Clearnet relay nodes — standard onion encryption]
         │ UDP/7474
         ▼
[Domain Bridge Node — clearnet UDP endpoint + mesh radio]
  Decrypts DOMAIN_BRIDGE onion layer
  Stores cross-domain path_id routing
  Forwards to mesh island via radio
         │ WiFi mesh or LoRa
         ▼
[Mesh relay nodes]
         │ mesh transport
         ▼
[Mesh service]
```

Return path:
```
Mesh service → mesh relays → bridge node (via radio, opaque)
Bridge node: cross-domain path_id lookup → clearnet relay predecessor
Bridge node → clearnet relays → client (via UDP, opaque)
```

**Privacy properties**:
- Clearnet relays see only: client IP (entry node) or relay-to-relay traffic (middle relays)
- Bridge node sees: last clearnet relay's address + first mesh relay's address
- Bridge node does NOT see: client IP or service identity
- Mesh relays see: only their immediate neighbors
- Mesh service sees: only last mesh relay's address
- No single node in any domain can link client to service

---

## 8. Mesh Island Services

BGP-X native services can be registered within a mesh island and made discoverable from the unified DHT.

### Service Registration (from SDK)

```rust
let listener = client.register_service(ServiceConfig {
    name: "community-forum".to_string(),
    advertise: true,    // Publish ServiceID to unified DHT
    auth_required: false,
    max_connections: 100,
}).await?;
```

When `advertise = true`: the service record is published to the unified DHT (via the bridge node when online). Clearnet clients discover the service by ServiceID and construct cross-domain paths to reach it.

When the bridge node is offline: the service record in the unified DHT expires (24-hour TTL). The service remains accessible from within the island. When the bridge reconnects: service record re-published automatically.

---

## 9. Mesh Deployment Scenarios

### Scenario 1: Pure Mesh (No Internet)

```
[Node A] ── LoRa ── [Node B] ── WiFi ── [Node C]
                 └── WiFi ── [Node D]
```

All nodes are mesh-only. Zero ISP dependency. Intra-island communications fully private. No clearnet access. DHT bootstrapped via beacons.

### Scenario 2: Mesh with Single Gateway

```
[Node A] ── LoRa ── [Node B] ── WiFi ── [Gateway] ── ISP ── Internet
```

One gateway bridges mesh to clearnet. All mesh users share the gateway's ISP connection. Individual mesh users have zero direct ISP exposure.

### Scenario 3: Multi-Gateway Redundancy

```
                    [Gateway EU] → EU ISP → clearnet
[Mesh Network] ──► [Gateway NA] → US ISP → clearnet
                    [Gateway AP] → AP ISP → clearnet
```

Three gateways in different jurisdictions. Mesh users route through any available gateway. If one goes offline, traffic automatically routes via others.

### Scenario 4: Multi-Island via Clearnet Bridge

```
[Island A: Lima]  ←──────────────────────────────→  [Island B: Bogotá]
[Pool: mesh-lima]         clearnet overlay          [Pool: mesh-bogota]
      │                                                      │
   Gateway A              [internet relays]               Gateway B

Cross-domain path:
User(A) → mesh hops → Gateway A → internet relays → Gateway B → mesh hops → User(B)
```

DHT synchronization via gateway nodes makes Island B discoverable from Island A. Path construction is automatic.

### Scenario 5: Mesh-to-Mesh Direct Bridge

```
[Island A WiFi mesh] ── direct LoRa link ── [Island B LoRa mesh]
                     (no internet involved)
```

A single node with both WiFi mesh and LoRa interfaces bridges two islands directly. No clearnet needed. The bridge pair is published to the unified DHT when a gateway comes online.

---

## 10. Elevated Relay Deployment

Elevated relay nodes dramatically extend mesh coverage.

### Coverage by Height (LoRa line-of-sight)

| Height | LoRa Range (Clear Terrain) |
|---|---|
| Ground level (2m) | 2-5 km |
| Rooftop (10-15m) | 5-15 km |
| Mast (30m) | 10-30 km |
| Tower (60m+) | 20-60 km |

### Fixed Mast (Recommended for Permanent Deployment)

A BGP-X Mesh Node in an outdoor enclosure (IP67) mounted at height. PoE-powered via CAT6 cable. No aviation authority involvement at typical heights (10-30m). See `/deployment/mast_tower_deployment.md` for installation details.

### Tethered Aerostat (Temporary Wide Coverage)

A weather balloon or aerostat carries a BGP-X relay payload at elevation. Power via tether cable. At 150m: LoRa line-of-sight radius ~45km — covers an entire small city.

Use case: temporary coverage during events or disaster response. Regulations: tethered aerostats generally have more permissive regulations than free-flying drones (under 100m is typically notify-only or unregulated in most jurisdictions).

### Vehicle Nodes

Vehicles with BGP-X Mesh Nodes serve as mobile relay infrastructure. A bus route passing through rural areas naturally extends and maintains mesh coverage, synchronizes DHT, and connects isolated mesh nodes dynamically.

---

## 11. Security Considerations

### Radio Interception

LoRa and WiFi signals can be received by anyone within range. All BGP-X overlay traffic is encrypted with ChaCha20-Poly1305. Even if radio traffic is captured, no plaintext is accessible. The transport adds no security surface to the BGP-X crypto model.

### Rogue Mesh Nodes

An adversary could introduce a node with a forged advertisement. Defense: all node advertisements must have valid Ed25519 signatures (including in MESH_BEACON). Nodes with invalid signatures are rejected. The reputation system penalizes anomalous behavior.

### Gateway Compromise

If a gateway/bridge node is compromised: the adversary can see destinations of clearnet traffic (same as any exit node compromise) and may block mesh-to-internet access. Defense: multiple gateways in different locations and jurisdictions; pool-based selection; path rebuilding when gateway becomes unresponsive.

### Mesh Island Isolation

An adversary could disrupt all bridge nodes connecting an island to other domains. This is an availability failure, not a privacy failure: intra-island communications continue with full privacy; only cross-domain connectivity is affected. Defense: multiple independent bridge nodes from different operators.

---

## 12. Integration with Existing Community Networks

### NYC Mesh / Freifunk / LibreMesh

These communities run BGP-X-compatible hardware (OpenWrt routers). No hardware changes required.

**Migration path**:
1. Install BGP-X as OpenWrt package on existing routers (no hardware change)
2. Configure existing mesh as a BGP-X pool with mesh island ID
3. Deploy one or more gateway/bridge nodes
4. Community members' internet traffic now routes through BGP-X overlay privately
5. Other communities adopt BGP-X → inter-community routing via unified DHT

### Meshtastic Communities

**Integration path**:
1. Add Raspberry Pi or GL.iNet router near existing Meshtastic nodes
2. Connect Meshtastic device to router via USB (as LoRa radio modem)
3. Flash BGP-X modem firmware to ESP32/nRF52840 (replaces Meshtastic routing protocol)
4. BGP-X router handles all routing, encryption, and DHT using LoRa radio
5. Existing Meshtastic users gain: full onion encryption, clearnet access, cross-island reach
