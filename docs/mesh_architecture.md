# BGP-X Mesh Architecture

This document describes BGP-X's mesh networking capabilities — operating without ISP connectivity via WiFi 802.11s, LoRa radio, Bluetooth, and other transports.

---

## Overview

BGP-X's mesh architecture enables communities to operate privacy-preserving networks without any ISP dependency for intra-community communications. Coverage gaps between geographically separated mesh islands can be bridged via internet relay pools, transparently and privately.

The mesh capability is not an add-on to BGP-X. It is a first-class deployment mode that uses the same protocol, the same onion encryption, and the same DHT discovery — with radio transports replacing UDP/IP where ISP connectivity is unavailable.

---

## Why Existing Mesh Networks Have Failed to Scale

Many mesh networking projects have been built. None have achieved widespread adoption because each solves only part of the problem:

### Meshtastic

**What it does**: LoRa-based mesh for text messages and GPS position. Excellent hardware, large community.

**Why limited**: Text messages only; no privacy (messages visible to all relay nodes); no clearnet exit; no general IP routing; no anonymity; flooding-based not optimized routing.

**What BGP-X adds**: Full onion encryption over LoRa transport via Meshtastic hardware adaptation layer. DHT-based discovery. Path selection. Pool-based isolation. Clearnet access via gateway nodes.

### GoTenna Pro / ATAK

**What it does**: Proprietary LoRa mesh for emergency/military communications.

**Why limited**: Proprietary (no third-party software); expensive; ecosystem-locked; no privacy from GoTenna; no clearnet routing.

**What BGP-X adds**: Open alternative using same class of hardware. Full privacy from any network observer.

### Helium Network

**What it does**: LoRaWAN network where operators earn cryptocurrency for coverage.

**Why limited**: Optimized for IoT sensor data; all traffic visible to network operators; no onion routing; cannot route general internet traffic; token incentive model has been controversial; coverage sparse.

**What BGP-X adds**: Privacy layer over LoRa transport. BGP-X can use Helium hardware via adaptation layer.

### NYC Mesh / Freifunk / LibreMesh

**What they do**: Community WiFi mesh networks providing free internet access. Run on commodity OpenWrt routers.

**Why limited**: No privacy (traffic visible to mesh operators); no anonymity; WiFi range limits coverage; each community isolated (no inter-community routing); volunteer-dependent; no standard federation.

**What BGP-X adds**: Privacy overlay directly on existing OpenWrt hardware. Inter-community routing via DHT pools. Automatic discovery and federation. Zero changes to existing network infrastructure — just install BGP-X daemon on existing routers.

### Reticulum Network Stack

**What it does**: Cryptographic mesh for low-bandwidth links. Multi-transport (LoRa, WiFi, serial). Active development.

**Why limited**: No onion routing (point-to-point encryption only); communication source visible to relays; no clearnet exit model; no hardware ecosystem.

**What BGP-X adds**: Multi-hop onion routing on top of multi-transport capability. Clearnet exit via gateways. Pool trust domains. Hardware ecosystem.

### The Unification BGP-X Provides

No existing system provides all of:
- Multiple transport types (WiFi, LoRa, Bluetooth, Ethernet, satellite)
- Full onion encryption (no relay sees both source and destination)
- Clearnet access via gateway nodes
- Pool-based trust domains
- Cross-network gap bridging via internet relay pools
- Unified DHT for global discovery
- Open firmware for specific hardware targets
- SDK for application integration
- Compatible with existing hardware (OpenWrt, Meshtastic adaptation)
- No fragmentation — one protocol, all transports

**BGP-X's innovation is unification, not invention.** Each component exists in prior systems. BGP-X is the first to combine all of them coherently.

---

## Transport Types

BGP-X is transport-agnostic. The onion encryption and routing logic are identical regardless of the underlying transport. The transport layer handles delivery of BGP-X datagrams between nodes.

### WiFi 802.11s Mesh

```
Protocol:  IEEE 802.11s mesh networking
Addressing: MAC-based within mesh, NodeID at BGP-X layer
Bandwidth: 10-100 Mbps per hop (hardware-dependent)
Range:     50-200m per hop (indoor), up to km with directional antennas
Latency:   1-20ms per hop
Use case:  Primary mesh transport for dense deployments (neighborhoods, buildings)
Multi-hop: Yes (802.11s handles IP routing within the segment)
```

### LoRa Radio

```
Protocol:  LoRaWAN or raw LoRa
Frequency: 868 MHz (EU), 915 MHz (NA), 433 MHz (general)
Bandwidth: 0.3-50 Kbps (spreading factor dependent)
Range:     5-50 km per hop (rural open terrain), 1-5 km urban
Latency:   100ms-5s per hop (duty cycle limits)
Use case:  Long-range low-bandwidth; DHT sync; key exchange; small messages
           NOT suitable for general web browsing or large data transfers
Multi-hop: Yes (BGP-X handles routing; LoRa handles each radio hop)
Notes:     Subject to regulatory duty cycle limits (1% in EU)
           BGP-X path selection treats LoRa hops as high-latency links
```

### Bluetooth Mesh (BLE)

```
Protocol:  Bluetooth mesh profile or custom BLE
Bandwidth: 1-2 Mbps
Range:     10-100m per hop
Latency:   10-100ms
Use case:  Short-range IoT device mesh; building-scale; mobile device pairing
```

### Ethernet Point-to-Point

```
Protocol:  Standard Ethernet or fiber
Bandwidth: 100 Mbps to 10 Gbps
Range:     100m (copper), 100km+ (fiber)
Latency:   <1ms
Use case:  High-bandwidth mesh backbone between fixed locations
           Building-to-building links in urban deployments
```

### Satellite (Gateway-Level)

```
Starlink: ~500 Mbps down, 20 Mbps up, 20-40ms latency (LEO)
Iridium Certus: 22-88 Kbps, global coverage including poles
HughesNet/ViaSat: 25-150 Mbps, 600ms+ latency (GEO — high latency)

Use case: WAN connection for gateway nodes in remote areas
          Not used for relay-to-relay mesh hops
BGP-X treatment: Satellite nodes tagged with latency class;
                  path selection avoids for interactive traffic;
                  suitable for bulk transfers and DHT sync
```

---

## Mesh Deployment Scenarios

### Scenario 1: Pure Mesh (No Internet)

```
         [Node A]────LoRa─────[Node B]────WiFi────[Node C]
                  └─────────WiFi──────────[Node D]
```

All nodes are Mode 4 (mesh only). No gateway. Intra-mesh communications are fully private, zero ISP involvement. No clearnet access. DHT bootstraps via beacons.

Use case: Remote community with no ISP; disaster recovery when ISP down; covert network.

### Scenario 2: Mesh with Gateway

```
         [Node A]────LoRa─────[Node B]────WiFi────[Gateway Node]────ISP────Internet
```

One or more gateway nodes (Mode 5) bridge the mesh to clearnet. All mesh users share the gateway's ISP connection. Individual mesh users have zero direct ISP exposure.

The gateway announces in both the mesh DHT and the internet BGP-X DHT, making internet BGP-X nodes discoverable from mesh nodes and vice versa.

### Scenario 3: Multi-Gateway Redundancy

```
                         [Gateway EU] → European ISP → clearnet
[Mesh Network]  ───► [Gateway NA] → American ISP → clearnet
                         [Gateway AP] → Asian ISP → clearnet
```

Mesh users route clearnet traffic through whichever gateway best serves their needs. BGP-X pool selection enables jurisdiction-specific routing. If one gateway goes offline, traffic automatically routes through others.

This gives mesh communities multi-jurisdiction clearnet access without any individual mesh user having direct ISP exposure.

### Scenario 4: Multi-Island Bridging

```
[Island A: Peru]                                 [Island B: Colombia]
[Pool: mesh-peru]                               [Pool: mesh-colombia]
     │                                               │
     └──────────────────────────────────────────────┘
                    bridged by
              [Pool: default (internet)]

Automatic: DHT gateway sync makes Island B's nodes appear in Island A's DHT
```

Path construction when a user on Island A connects to a user on Island B:

```
User(A) → [mesh-peru hops] → Gateway(A) → [default internet hops] → Gateway(B) → [mesh-colombia hops] → User(B)
```

This is fully automatic via pool configuration. Users on both islands see each other as any other BGP-X peer.

---

## DHT in Mesh-Only Mode

When operating without any internet connection, the DHT adapts:

### Bootstrap via Beacons

Instead of hardcoded internet bootstrap nodes, mesh nodes broadcast beacons:

```json
{
  "type": "bgpx_mesh_beacon",
  "node_id": "a3f2...",
  "public_key": "...",
  "transports_supported": ["wifi_mesh", "lora"],
  "dht_routing_table_size": 45,
  "timestamp": "2026-04-24T12:00:00Z",
  "signature": "..."
}
```

Beacons are broadcast every 30 seconds on all active mesh transports. Any node receiving a beacon adds the sender to its DHT routing table. Bootstrap is complete when the routing table reaches sufficient density.

### Routing Table Construction

Mesh DHT uses the same Kademlia algorithm as internet DHT. The difference is transport:
- Internet: UDP/IP to node IP addresses
- Mesh: WiFi 802.11s or LoRa to mesh transport addresses (derived from NodeID)

### Record Storage

Pool advertisements, node advertisements, and all DHT records operate identically in mesh mode. Records TTL and re-publication schedules are unchanged.

---

## Gateway DHT Cross-Domain Synchronization

Gateways (Mode 5) synchronize DHT records between the mesh DHT and internet DHT:

```
Internet BGP-X node publishes advertisement
  → Gateway fetches via internet DHT
  → Gateway propagates into mesh DHT
  → Mesh nodes can discover internet nodes

Mesh node publishes advertisement to mesh DHT
  → Gateway notices new record
  → Gateway publishes into internet DHT
  → Internet nodes can discover mesh-accessible nodes
```

This creates a unified global DHT spanning all transport types. A BGP-X path can mix internet and mesh hops transparently.

---

## Elevated Relay Deployment

Elevated relay nodes dramatically extend coverage. Options:

### Fixed Mast or Rooftop (Recommended for Permanent Deployment)

A BGP-X Mesh Node in an outdoor enclosure mounted at height. PoE-powered via cable. No regulatory complexity beyond normal antenna installation permits.

At 60m: LoRa line-of-sight radius ~30km. At 90m: ~38km.

This is the most practical option for permanent elevated coverage. No aviation authority involvement.

### Tethered Aerostat (Temporary Wide Coverage)

A weather balloon or aerostat tethered to the ground carries a BGP-X relay payload at elevation. Power via tether cable.

Regulatory note: tethered aerostats have different (generally more permissive) regulations than free-flying drones in most jurisdictions. Under 60m is typically unregulated or notify-only in most regions.

At 150m: LoRa line-of-sight radius ~45km — covers an entire small city.

Use case: temporary wide coverage during events or disaster response. Not ideal for permanent deployment (helium refill, weather limitations).

### Drone Relay (Emergency Only)

A drone carrying a BGP-X relay payload provides temporary elevated coverage.

**Honest assessment**: drones are not practical for persistent relay infrastructure. Limited flight time (30-60 minutes), regulatory limits (120-150m AGL in most jurisdictions), and need for continuous operation make drones suitable only for emergency temporary deployment.

For persistent coverage: use fixed masts or tethered aerostats.

---

## Vehicle Nodes

Vehicles with BGP-X Mesh Nodes serve as mobile relay infrastructure:

- A bus route with a BGP-X vehicle node connects static mesh nodes along the route as it passes
- DHT records propagate during transit
- Store-and-forward capability allows messages to be relayed even if sender and receiver are not simultaneously connected

The "traveling mesh" effect: vehicles driven regularly naturally extend and maintain mesh coverage, synchronize DHT, and connect isolated mesh islands dynamically.

---

## Security Considerations for Mesh

### Radio Interception

LoRa and WiFi signals can be received by anyone within radio range. **All BGP-X overlay traffic is encrypted with ChaCha20-Poly1305.** Even if radio traffic is captured, no plaintext is accessible. The mesh transport adds no security surface to the BGP-X crypto model.

### Rogue Mesh Nodes

An adversary could introduce a node with a forged advertisement. **Defense**: all node advertisements must have valid Ed25519 signatures. Nodes with invalid signatures are rejected. The reputation system penalizes nodes with anomalous behavior.

### Gateway Compromise

If a gateway node is compromised, the adversary can see destinations of clearnet traffic (same as any exit node compromise) and can block or disrupt mesh-to-internet access. **Defense**: multiple gateways in different locations and jurisdictions; pool-based gateway selection; path rebuilding when gateway becomes unresponsive.

### Mesh Partitioning Attack

An adversary could physically interfere with mesh radio transmissions (jamming, destroying equipment). **Defense**: multiple transport types (LoRa + WiFi provides redundancy); multiple gateways; BGP-X doesn't guarantee physical connectivity — it provides privacy for whatever connectivity exists.

---

## Migration from Existing Community Networks

### For NYC Mesh / Freifunk / LibreMesh Communities

These communities run BGP-X-compatible hardware (OpenWrt routers).

Step 1: Install BGP-X daemon as an OpenWrt package on existing routers (no hardware changes).

Step 2: Configure existing mesh as a BGP-X pool. All existing routing continues to work.

Step 3: Deploy one or more gateway nodes. Community members' internet traffic now routes privately through BGP-X overlay.

Step 4: Other communities adopt BGP-X. Inter-community routing becomes possible via DHT. The coverage gap between communities is bridged via internet relay pools.

### For Meshtastic Communities

Meshtastic hardware (ESP32/nRF52840) cannot run the full BGP-X daemon. However:

Step 1: Add a Raspberry Pi or GL.iNet router to locations that already have Meshtastic nodes and power.

Step 2: Connect the Meshtastic device to the router via USB. The router uses the Meshtastic device as a LoRa radio modem via the BGP-X adaptation layer.

Step 3: Existing Meshtastic users gain: full onion encryption, clearnet access via gateways, cross-network reach.

The existing Meshtastic hardware serves as radio infrastructure. BGP-X provides the privacy and routing layer on top.
