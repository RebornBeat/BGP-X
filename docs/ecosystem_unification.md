# BGP-X: Ecosystem Unification

**Version**: 0.1.0-draft

This document explains the fragmented mesh networking ecosystem, why existing systems have failed to reach widespread adoption, and how BGP-X provides the unifying architecture.

---

## 1. The Fragmentation Problem

The current privacy and mesh networking ecosystem is fragmented. Privacy technology and mesh networking have developed in parallel silos. Users who need privacy typically use Tor or a VPN — internet-only tools with no mesh capability. Community mesh networks (NYC Mesh, Freifunk, Meshtastic) provide coverage without ISP dependency but no privacy. No existing system provides both, let alone cross-domain routing between them.

Each system works in isolation:

- **Tor** provides internet anonymity but cannot reach mesh island services
- **Reticulum** provides mesh networking but has no onion-routing privacy model
- **NYC Mesh / Freifunk** provide community internet access but are isolated from other communities
- **Meshtastic** provides LoRa mesh communication but is disconnected from internet services
- **cjdns / Yggdrasil** provide encrypted routing but no source anonymity
- **I2P** provides internal anonymity but limited clearnet access

A Tor user cannot reach a Meshtastic-connected service. A Freifunk community in Berlin cannot communicate privately with one in Lima. A Reticulum mesh island cannot route through the Tor/I2P overlay for additional privacy.

When a community wants to deploy private mesh networking, they face an impossible choice:

```
Want LoRa range?                    → Meshtastic (no privacy, no clearnet)
Want community WiFi?                → NYC Mesh (no privacy, no mesh federation)
Want anonymity?                     → Tor (no mesh transport, internet-only)
Want privacy over mesh?             → Reticulum (no onion routing, no clearnet)
Want public key addressing?         → cjdns (no anonymity, no clearnet exit)
Want ISP-free operation?            → Meshtastic or Reticulum (limited capabilities)
Want full internet access?          → VPN (single trust point, no mesh)
```

No existing system provides all of these properties simultaneously. Every community must choose and sacrifice.

---

## 2. The Unification: Routing Domains

BGP-X's unification is the **routing domain model** that makes all of these first-class participants in one network.

Every BGP-X routing domain is a first-class participant in a single unified topology:

1. **No domain is secondary**: clearnet, overlay, and mesh are equals
2. **Any domain can originate a path**: clearnet client, mesh client, satellite client — same routing algorithm
3. **Any domain can be a destination**: services in any domain reachable from any other
4. **Any combination** of domains can be intermediate hops — no ordering enforced
5. **N-hop unlimited across all domains** — no protocol maximum at any level
6. **One unified DHT**: all domains, one discovery layer, one consistent view

This is ecosystem unification — not "these systems coexist" but "these systems are one system."

A Meshtastic community in Lima, a privacy-conscious clearnet user in Berlin, and a satellite-connected research station in Antarctica can all participate in the same network, route to each other's services, and maintain end-to-end privacy throughout.

---

## 3. Capability Comparison — Detailed

### What Each System Provides (and Doesn't)

| Feature | Meshtastic | NYC Mesh | Tor | I2P | cjdns | Yggdrasil | Reticulum | SCION | BGP-X |
|---|---|---|---|---|---|---|---|---|---|
| LoRa transport | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| WiFi mesh | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Onion routing | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Clearnet exit | ❌ | ✅ | ✅ | Limited | ❌ | ❌ | ❌ | Limited | ✅ |
| Anonymity | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| No ISP needed | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Public key addressing | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Decentralized discovery | ✅ | ❌ | ❌ | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ✅ |
| Pool trust domains | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Router firmware | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Hardware ecosystem | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Inter-network routing | ❌ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ |
| ECH at exit | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Cross-domain routing | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| .bgpx / .onion services | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 4. Hardware Ecosystem Unification

BGP-X provides hardware for every participant tier:

### End Users — BGP-X Router v1

Users replace their home router with BGP-X Router v1. All their devices are protected. They participate in the BGP-X overlay as relay nodes. If they have line-of-sight to neighbors with mesh hardware, they participate in the local mesh island. Zero additional configuration.

### Community Contributors — BGP-X Node v1

Community members mount BGP-X Node v1 units at elevation — rooftops, poles, masts. Solar-powered. No ISP connection required at the node (unless configured as a gateway). They extend mesh coverage. Their nodes relay BGP-X traffic for island members. With WAN Ethernet, they serve as the island's internet gateway. With LoRa radio, they extend coverage for kilometers.

The BGP-X Node v1 replaced the earlier "Amplifier v1" concept. A dumb LoRa repeater operating at the PHY level would break end-to-end encryption. Node v1 in Range Extension mode provides equivalent coverage while maintaining full BGP-X protocol security at every hop.

### Providers — BGP-X Gateway v1

ISPs, hosting providers, and community organizations run BGP-X Gateway v1 in datacenters. High-throughput, fiber-connected, certified exit nodes serving thousands of simultaneous users. These provide the clearnet exit capacity for the global BGP-X network.

### Individual Endpoints — BGP-X Client Node (Tier 2)

Low-cost, battery-powered endpoint for connecting to BGP-X mesh networks. Does not route traffic for others. Designed for individual users, sensors, and portable use.

Reference hardware: LILYGO T3S3, T-Beam, Heltec V3, RAK WisBlock. ESP32-S3 or nRF52840 based boards with integrated SX1262 LoRa radios. Firmware: BGP-X Client Firmware (subset implementation).

### USB LoRa Modem — BGP-X Adapter/Dongle (Tier 3)

USB dongle that acts as a LoRa radio modem. Contains no routing intelligence. The host computer runs the full BGP-X daemon; the dongle handles only LoRa transmission and reception. Allows any laptop or server with a USB port to become a BGP-X mesh client or domain bridge node.

### Existing Hardware — BGP-X OpenWrt Package

Community members who already own compatible OpenWrt routers don't need new hardware. Install the OpenWrt package, add a USB LoRa adapter, and contribute to the network. Same daemon, same protocol, same capabilities (minus TPM and native LoRa integration).

---

## 5. Software Ecosystem Unification

### Transparent Protection (Standard Apps)

Every application on a LAN device protected by BGP-X Router v1 is automatically protected. No code changes. No proxy configuration. The TUN interface intercepts traffic at the routing policy layer.

### SDK Applications (BGP-X Aware)

Applications that want explicit path control, cross-domain routing, pool constraints, or `.bgpx` service hosting use the BGP-X SDK. Available in Rust (primary), Python, TypeScript, Go.

### BGP-X Browser (Native .bgpx Experience)

BGP-X Browser handles `.bgpx` URL resolution, ServiceID authentication, latency-adaptive rendering, and path status display. This is the interface through which users browse `.bgpx` services — equivalent to how Tor Browser is the interface for `.onion` services. BGP-X Browser also routes clearnet traffic through the daemon transparently for non-.bgpx sites.

**LoRa browsing in BGP-X Browser**: the browser detects LoRa-class paths from daemon events and activates batched resource loading and progressive rendering. Users browsing `.bgpx` services via LoRa paths experience latency but not failure — the browser adapts rather than timing out.

### bgpx-cli (Operator Interface)

Full command-line management of the daemon: sessions, paths, pools, domains, islands, reputation, names, certificates. Operators manage their nodes entirely via CLI or bgpx-luci web UI.

---

## 6. Satellite Ecosystem Integration

### What Satellite Is in BGP-X

Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat, Amazon Kuiper) are **clearnet** in BGP-X. They provide BGP-routed IP connectivity via satellite radio as the physical layer. From the protocol's perspective: clearnet domain, domain type 0x00000001, with a satellite-class latency annotation.

BGP-X Router v1 auto-detects satellite USB modems and configures them as clearnet WAN. No user configuration needed beyond selecting the satellite provider in the setup wizard.

### What Satellite Enables for BGP-X

Satellite internet as clearnet WAN enables BGP-X deployment in locations where fiber and cellular are unavailable:
- Remote rural communities forming mesh islands with Starlink as their gateway WAN
- Maritime vessels participating in the BGP-X overlay while at sea
- Polar and extreme-remote research stations with Iridium as their only WAN
- Disaster response scenarios deploying BGP-X coverage rapidly via satellite terminal

In all cases: the satellite is the clearnet-domain WAN. The BGP-X overlay runs on top. The mesh domain (LoRa, WiFi) operates via radio. Domain bridge nodes bridge the two. The result: a fully privacy-capable BGP-X mesh island with satellite internet access, deployed anywhere on Earth.

---

## 7. What Each Prior System Contributes

BGP-X inherits and extends:

- **cjdns**: public key addressing → BGP-X NodeID model
- **Yggdrasil**: DHT-based routing → BGP-X unified Kademlia DHT
- **Reticulum**: multi-transport mesh → BGP-X mesh transport abstraction
- **Tor**: onion routing → BGP-X onion encryption (domain-agnostic)
- **I2P**: garlic routing and native service model → BGP-X SDK native services
- **Meshtastic**: LoRa community mesh → BGP-X mesh transport + adaptation layer
- **NYC Mesh / Freifunk**: community WiFi mesh → BGP-X OpenWrt deployment
- **Kademlia DHT**: node discovery algorithm
- **Noise Protocol**: handshake model

**BGP-X adds**: the domain routing layer that connects all of these into one coherent topology. The DOMAIN_BRIDGE hop type. The unified DHT spanning all domains. The domain bridge node model. The cross-domain path_id return routing. The N-hop unlimited protocol design.

---

## 8. How BGP-X Works With Existing Systems

### NYC Mesh / Freifunk / LibreMesh Integration

These communities run OpenWrt routers — the same hardware BGP-X targets.

**Integration path**:
1. Install BGP-X as an OpenWrt package on existing routers:
   ```bash
   opkg update
   opkg install bgpx-node bgpx-cli bgpx-luci
   ```
2. Configure island_id matching the community's identifier:
   ```toml
   [[routing_domains]]
   domain_type = "mesh"
   island_id = "nyc-mesh-community"
   transports = ["wifi-mesh"]
   ```
3. Existing mesh routing continues unchanged
4. BGP-X adds privacy overlay on top of existing connectivity
5. Deploy one or two bridge nodes (BGP-X Router v1 or Node v1 with WAN) to connect community to global BGP-X network

**What changes for community members**: Traffic is now privately routed through BGP-X overlay. The mesh itself is unchanged.

**What community gains**:
- Privacy for all member traffic
- Federation with other BGP-X communities worldwide
- Coverage gap bridging via internet relay pools

### Meshtastic Integration

Meshtastic devices (ESP32/nRF52840) cannot run full BGP-X daemon (too constrained). Integration via adaptation layer:

**Integration path**:
1. Add a BGP-X Router v1 or Raspberry Pi running bgpx-node near existing Meshtastic infrastructure
2. Connect Meshtastic device via USB
3. Flash BGP-X modem firmware to Meshtastic device (replaces Meshtastic routing with raw LoRa modem mode)
4. BGP-X node uses Meshtastic device as LoRa radio
5. BGP-X handles all routing, encryption, and DHT

**Hardware reuse**: Same Meshtastic antennas and radios. Same physical infrastructure.

**What Meshtastic community gains**:
- Full onion encryption over existing LoRa infrastructure
- Clearnet access via BGP-X gateway nodes
- Cross-island reach via unified DHT
- Access from clearnet BGP-X users without LoRa hardware

### Helium Network Integration

Helium hotspots contain LoRaWAN radio hardware that could be repurposed. With modified firmware:

1. Helium hotspot hardware (RAK concentrator + gateway SBC) runs BGP-X daemon
2. LoRa radio serves BGP-X mesh transport instead of (or in addition to) LoRaWAN
3. Helium coverage becomes BGP-X mesh coverage

Note: Helium's blockchain-based incentive model is separate from BGP-X. Any economic incentive layer for BGP-X node operators is out of scope for the base protocol.

### Reticulum Network Stack

Reticulum users get the missing privacy layer:

- Reticulum provides multi-transport mesh capability (LoRa, WiFi, serial) — BGP-X provides the same transport support plus onion routing
- BGP-X can be deployed on Reticulum-compatible hardware (any Linux-capable device with LoRa)
- BGP-X and Reticulum operate independently on the same hardware (different daemons, different protocols)
- Future: BGP-X transport driver for Reticulum's HDLC transport (interop research area)

---

## 9. Inter-Community Routing

Previously: communities using different mesh systems (Meshtastic, NYC Mesh, Freifunk) could not communicate privately with each other.

With BGP-X domain routing: each community runs BGP-X with its own `island_id`. Domain bridge nodes connect each community to the BGP-X clearnet overlay. Through the unified DHT, communities discover each other. Cross-domain paths connect them through the overlay.

The coverage gap between communities — previously an absolute barrier — becomes a clearnet overlay segment in a cross-domain path.

---

## 10. The Coverage Gap Bridging Solution

The most significant architectural capability BGP-X provides that no existing system offers: **automatic, transparent bridging between geographically separated mesh islands via internet relay pools**.

### The Problem

Two communities each have functional mesh networks but cannot directly connect because they're physically separated (different cities, different countries, too far for radio).

With existing systems: no connection. The communities are isolated.

### The BGP-X Solution

Each community deploys a gateway node (Mode 5) connecting their mesh to the internet. Each community configures a pool for their mesh nodes.

```
Community A (Lima)                    Community B (Bogotá)
mesh:lima-district-1                  mesh:bogota-district-1
         │                                      │
    Bridge A          BGP-X overlay       Bridge B
    (gateway)     (internet-connected      (gateway)
                   BGP-X relays)

Cross-domain path:
  mesh:lima → Bridge A → clearnet overlay → Bridge B → mesh:bogota
```

The unified DHT cross-domain synchronization makes each community's nodes appear in the other's DHT. Path construction automatically uses the three-segment path across all pools.

**What the user experiences**: connecting to a peer in the other community works just like connecting to a local mesh node. No manual configuration. No awareness of the internet bridge.

**Privacy**: all traffic is onion-encrypted throughout. The internet relay nodes see only encrypted BGP-X traffic. The gateways see which segment they're connecting (but not the full path).

---

## 11. Protocol Ecosystem Unification

BGP-X unifies what was previously a fragmented landscape:

| Protocol | Privacy | Clearnet | LoRa | WiFi Mesh | No-ISP | .Address System | Cross-Network |
|---|---|---|---|---|---|---|---|
| Tor | ✅ | ✅ | ❌ | ❌ | ❌ | .onion | ❌ |
| I2P | ✅ | Limited | ❌ | ❌ | ❌ | .i2p | ❌ |
| cjdns | Limited | ✅ | ❌ | ✅ | ❌ | IPv6 | ❌ |
| Reticulum | ❌ | ❌ | ✅ | ✅ | ✅ | None | ❌ |
| Meshtastic | ❌ | ❌ | ✅ | ❌ | ✅ | None | ❌ |
| Helium | ❌ | ❌ | ✅ | ❌ | ❌ | None | ❌ |
| **BGP-X** | **✅** | **✅** | **✅** | **✅** | **✅** | **.bgpx** | **✅** |

BGP-X is not positioned against these systems — it learns from each:
- From Tor: onion encryption, distributed trust, .address system
- From Reticulum: multi-transport mesh, low-bandwidth focus
- From Meshtastic: LoRa hardware community, real-world deployment
- From cjdns/Yggdrasil: mesh routing on WiFi infrastructure
- From I2P: native service model, self-contained network
- From Helium: economic incentive model (separate from BGP-X base protocol)

BGP-X integrates these into one coherent system: one daemon, one protocol, one hardware platform, all transports, all domains, all users.

---

## 12. What BGP-X Innovates vs. Builds On

### What BGP-X Innovates

- **Pool-based trust domains** with double-exit architecture
- **Cross-domain routing** (any domain → any domain, any order, N-hop unlimited)
- **Unified DHT** spanning all routing domains
- **Router-level deployment** (all devices protected without per-device configuration)
- **Hardware platform** with unified core and deployment-specific variants
- **ECH at exit nodes** for domain name privacy
- **Clearnet clients reaching mesh island services** without any special hardware

### What BGP-X Builds On

- **cjdns**: public key addressing model
- **Yggdrasil**: DHT-based routing foundation
- **Reticulum**: multi-transport mesh validation
- **Tor**: onion routing model
- **I2P**: native service architecture
- **Meshtastic**: LoRa mesh hardware demonstration
- **NYC Mesh / Freifunk**: community WiFi mesh deployment
- **Kademlia DHT**: node discovery algorithm
- **Noise Protocol**: handshake model

---

## 13. The User Journey

### Starting from Zero

1. User purchases BGP-X Router v1
2. Plugs it in where their old router was
3. Setup wizard runs: WAN type → role → keypair generated in TPM → DHT bootstrap → done
4. All LAN devices immediately protected by 4-hop clearnet BGP-X overlay
5. Install BGP-X Browser on one device: now browsing `.bgpx` services
6. Their node appears in the DHT as a public relay: they are now contributing to the network

### Expanding to Mesh

1. User buys a BGP-X Node v1 for their rooftop
2. Mounts it with LoRa antenna and Ethernet cable for PoE power
3. Configures: `island_id = "my-neighborhood"`, mesh mode, no WAN needed
4. Neighbors with BGP-X Router v1 join the same island_id
5. Island forms: all island traffic routes locally via LoRa/WiFi; clearnet via user's Router v1 gateway
6. Someone on clearnet anywhere in the world can now reach services on this island

### Contributing as Provider

1. Organization with datacenter space sets up BGP-X Gateway v1
2. Configures as certified exit node with signed exit policy
3. Published to DHT; appears in default exit pool
4. Serves thousands of concurrent exit connections
5. Community depends on this high-throughput exit node

All three of these users — end user, community contributor, provider — are participating in the same unified BGP-X network, running the same protocol, using the same unified DHT.

---

## 14. Honest Competitive Positioning

BGP-X should not overclaim. The honest positioning:

**BGP-X IS** the first to:
- Unify onion routing + mesh transport + clearnet exit + decentralized unified DHT + pool trust domains + hardware ecosystem into one coherent system
- Enable any-to-any cross-domain routing (clearnet ↔ mesh ↔ satellite in any combination)
- Enable clearnet clients to reach mesh island services without any special hardware
- Implement N-hop unlimited path construction with no protocol maximum
- Operate a unified DHT spanning all routing domains
- Provide router-level deployment with transparent protection for all LAN devices

**BGP-X is NOT** the first:
- Mesh network (Reticulum, Meshtastic predate it)
- Anonymous communication system (Tor, I2P, cjdns predate it)
- To use public key addressing (cjdns pioneered this)
- To support LoRa transport (Reticulum, Meshtastic predate it)
- To provide onion routing (Tor, I2P predate it)

**BGP-X builds on** decades of prior work by the mesh networking, anonymity, and cryptographic networking communities. The goal is to complete the picture that prior systems have each partially drawn.

BGP-X's value is in synthesis and unification, not in any single primitive.

---

## 15. Key Architectural Summary

BGP-X unifies the fragmented privacy and mesh networking ecosystem by implementing a **routing domain model** where clearnet, BGP-X overlay, and mesh islands are equal first-class citizens.

Key architectural innovations:
- **Unified DHT** — one discovery layer for all domains
- **Domain bridge nodes** — nodes with endpoints in multiple routing domains
- **DOMAIN_BRIDGE hop type** — explicit protocol support for domain transitions
- **Cross-domain path_id routing** — return traffic routing across domain boundaries
- **N-hop unlimited** — no protocol maximum at any level
- **Pool-based trust** — configurable trust segments with double-exit capability
- **Hardware ecosystem** — Router, Node, Gateway, Client, Adapter covering all participant tiers

This is ecosystem unification: not "these systems coexist" but "these systems are one system."
