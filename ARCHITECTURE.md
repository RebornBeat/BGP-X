# BGP-X System Architecture

**Version**: 0.1.0-draft

This document describes the full system architecture of BGP-X: a router-level privacy overlay network and inter-protocol domain router.

---

## 1. Design Philosophy

BGP-X is built on seven non-negotiable principles:

### 1.1 No single party should see both endpoints of a connection

This is the foundational property. No node in the BGP-X network, no ISP, and no gateway can link a source identity to a destination. This is enforced at the protocol level, not by policy.

### 1.2 Identity is cryptographic, not location-based

BGP-X separates identity from location:

- **Location** is handled by underlying IP/BGP or mesh transport (opaque to BGP-X clients)
- **Identity** is a public key — self-certifying, portable, and optionally ephemeral

### 1.3 Routing decisions belong to the sender

BGP delegates path selection to the network. BGP-X delegates path selection to the client. The sender constructs the complete path before transmitting a single packet. No protocol-level limit on path length, domain count, or segment count.

### 1.4 The existing internet is transport, not adversary

BGP-X uses the public internet as one of several raw packet transport layers and builds all privacy and routing logic above it. For mesh deployments, BGP-X uses local radio transport with zero ISP involvement.

### 1.5 Layered security, not security by obscurity

Every privacy property must be cryptographically enforceable. Honest operators improve performance; dishonest operators must fail to compromise privacy.

### 1.6 BGP-X is infrastructure, not a tool

BGP-X runs on the network router and protects all connected devices transparently. It is not a per-application tool. The daemon is the single BGP-X routing stack — all other components (SDK apps, config client) are clients of the daemon.

### 1.7 All network domains are equal entry and exit points

Clearnet, BGP-X overlay, and mesh islands are three equal first-class citizens. A client in any domain can reach a service in any other domain. No domain is secondary, no entry point is privileged, and no domain ordering is enforced by the protocol.

---

## 2. What BGP-X Is

BGP-X is a **router-level privacy overlay network and inter-protocol domain router**. It runs on the network router or as a standalone daemon, protecting all connected devices transparently. Applications require no modification.

BGP-X provides:
- **Onion-encrypted routing**: no single node sees both endpoints of a connection
- **Client-selected paths**: the sender constructs the complete path before transmission
- **Cross-domain routing**: paths may span clearnet, overlay, and mesh islands in any combination, any order, unlimited hops
- **Three equal entry points**: clearnet, BGP-X overlay, and mesh islands are first-class citizens
- **Router-centric deployment**: all LAN devices protected without per-device configuration
- **.bgpx domain system**: self-authenticating service addresses, equivalent to .onion for Tor
- **Unified DHT**: a single decentralized discovery layer spanning all routing domains
- **Pool-based trust**: named pools with curator signatures enable trust segmentation
- **N-hop unlimited**: no protocol-level maximum on path length, domain count, or segment count

---

## 3. The Core Problem

The internet was designed to connect machines, not to protect the identity of the people using those machines. Every packet carries a source IP and destination IP, visible to every network device along the path. Your router, your ISP, any ISP along the path, and the destination's ISP all see who is talking to whom.

HTTPS protects the *content* of the packet. It does not protect the metadata: who is talking to whom, when, and how often. That metadata — often called traffic analysis data — can be as sensitive as the content itself.

Existing solutions each solve part of the problem:

- **VPNs**: shift ISP surveillance to VPN provider; single point of trust. The VPN provider sees everything the ISP previously saw. Trust is shifted, not eliminated.
- **Tor**: distributes trust across 3 fixed hops; centralized directory authorities; internet-only; fixed path length; TCP-only transport; application-proxy architecture limits what can be built on top of it
- **I2P**: garlic routing for internal services; strong internal network properties but limited clearnet access model; primarily designed for internal services, not general internet traffic
- **cjdns/Yggdrasil**: public key addressing with mesh transport; no onion routing; no anonymity
- **Reticulum**: multi-transport mesh capability; no onion routing; no clearnet exit model

BGP-X unifies: multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust + hardware targets — into one coherent router-level system with cross-domain routing and a native .bgpx domain system.

---

## 4. Router-Centric Design

BGP-X is designed as **network infrastructure, not a per-application tool**.

The BGP-X daemon (`bgpx-node`) runs on the network router and is the single BGP-X routing stack for all devices on the network. Every other BGP-X component — SDK applications, the configuration client (`bgpx-cli`) — is a client of this daemon.

This means:

- **Standard applications** require zero modification to be protected
- **All devices on the LAN** are automatically protected by the router's BGP-X deployment
- **Application developers** use the SDK to connect to the daemon — not to build their own overlay stack
- **Configuration and management** happens via the Control Socket, accessed by the `bgpx-cli` or web UI

On **standalone devices** (laptop, server), the daemon runs locally — identical code, narrower scope. For **mobile and embedded environments**, the SDK offers an embedded mode where the daemon runs in-process.

---

## 5. System Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│  Standard apps (transparent) + SDK apps (aware)        │
│  BGP-X Browser (.bgpx URLs, latency-adaptive rendering)│
├─────────────────────────────────────────────────────────┤
│              .bgpx Domain System                        │
│  ServiceID addressing, Name Registry DHT,              │
│  HTTP/2 over BGP-X streams, latency-tolerant rendering │
├─────────────────────────────────────────────────────────┤
│                Routing Policy Engine                    │
│  Per-flow decision: BGP-X overlay vs. standard routing  │
│  Domain-aware: cross-domain path rules supported        │
├─────────────────────────────────────────────────────────┤
│                   Identity Layer                        │
│  Public key identities, resolution, rotation           │
├─────────────────────────────────────────────────────────┤
│              Inter-Protocol Domain Router               │
│  Cross-domain path construction (clearnet↔overlay↔mesh) │
│  Domain bridge node selection and management            │
│  N-hop unlimited routing across all domain types        │
├─────────────────────────────────────────────────────────┤
│                   Overlay Routing Layer                 │
│  Path construction, onion encryption, forwarding       │
│  Unified DHT (all domains), pools, reputation, geo     │
├─────────────────────────────────────────────────────────┤
│                   Gateway Layer                         │
│  Overlay → Public Internet translation                  │
│  Domain bridge: clearnet ↔ mesh ↔ satellite            │
│  ECH support, DNS resolver, exit policy enforcement    │
├─────────────────────────────────────────────────────────┤
│                   Transport Layer                       │
│  UDP/IP (clearnet, including all satellite internet)  │
│  WiFi 802.11s, LoRa, BLE, Ethernet P2P (mesh)        │
│  Pluggable transport for obfuscation                  │
└─────────────────────────────────────────────────────────┘
```

### Layer 0: Transport

The physical delivery medium. For internet deployments: UDP/IP via BGP-routed infrastructure. For mesh deployments: WiFi 802.11s, LoRa radio, Bluetooth BLE, or Ethernet point-to-point. BGP-X is transport-agnostic. The onion encryption and routing logic are identical regardless of whether the underlying transport is UDP, LoRa, or WiFi mesh.

### Layer 1: Overlay Routing (Data Plane)

BGP-X nodes exchange onion-encrypted packets. Each packet contains:
- A `path_id` (8 bytes, random per path) — enables return traffic routing
- A routing layer (encrypted for the next hop only)
- A payload (encrypted for each subsequent hop in onion layers)
- An authentication tag (integrity protection)

Each hop decrypts its layer, extracts the `path_id` and next-hop address, records `path_id → predecessor` in memory, and forwards the remaining ciphertext.

For return traffic: nodes look up `path_id` and forward opaque encrypted blobs toward the client. **Intermediate relays do NOT decrypt return traffic.**

### Layer 2: Control Plane

Manages how nodes discover each other and how paths are constructed.

- **Node discovery**: Kademlia-style **Unified DHT**. Nodes publish signed advertisements. Clients query DHT to discover nodes and their properties. In mesh-only mode, nodes bootstrap via broadcast beacons (no internet bootstrap nodes needed).
- **DHT Pools**: Named, signed collections of nodes. Pool discovery uses the same DHT infrastructure. Pools enable multi-segment paths with different trust tiers per segment.
- **Path construction**: Client-side. The client selects nodes from the DHT (respecting pool membership if specified), verifies signatures, applies diversity constraints, and constructs the path before sending a single packet.

### Layer 3: Identity and Resolution

- **Node identity**: long-term Ed25519 keypair. `NodeID = BLAKE3(public_key)`. Stable across IP changes.
- **Client identity**: ephemeral Ed25519 keypair by default. Generated per session, destroyed after handshake. Long-term client identity supported for authenticated services.
- **Service identity**: stable public key. Services register reachability in DHT. Infrastructure can change without changing `ServiceID`.
- **Pool curator identity**: Ed25519 keypair separate from node identity. Signs pool advertisements and membership records. Rotatable via dual-signature key rotation record.

### Layer 4: Routing Policy Engine

The policy engine evaluates each packet or flow from LAN devices and decides:
- Route via BGP-X overlay (into `bgpx0` TUN)
- Route via standard BGP (directly to WAN)
- Block

Rules are evaluated in order; first match wins. Rule types: device (IP/MAC), application (process/UID), destination (IP/CIDR/domain), protocol/port, pool-specific path constraints, default.

The routing policy engine is what makes the dual-stack model work: one WAN connection serves both BGP-X overlay traffic (encrypted UDP) and standard traffic, with the policy engine deciding which path each flow takes.

### Layer 5: Gateway (BGP Interoperability)

Exit nodes bridge the BGP-X overlay to the public internet. They are the only nodes that touch both worlds.

A gateway node:
- Receives onion-encrypted packets from the overlay
- Decrypts the final onion layer
- Sees the destination address (not the originating client)
- Resolves DNS via DoH with DNSSEC validation and ECS stripping
- **Supports ECH** when destination publishes ECH configuration in DNS HTTPS records
- Sends traffic to the destination via the public internet
- Receives responses and routes them back via `path_id`

---

## 6. Core Components

### 6.1 BGP-X Router Daemon (bgpx-node)

The single BGP-X routing stack. Runs on the router or standalone device. All other components are clients of this daemon.

Responsibilities:
- Path construction and management (single-domain and cross-domain)
- Cross-domain routing domain management
- Domain bridge node discovery and selection
- Session establishment (handshake with each hop, domain-agnostic)
- Onion encryption and decryption (domain-agnostic crypto suite)
- Unified DHT participation and node discovery
- Pool management (including domain-scoped pools)
- Routing policy enforcement (domain-aware)
- TUN interface management
- SDK Socket server
- Control Socket server
- Reputation tracking (with per-domain event tagging)
- Geographic plausibility measurement (domain-specific thresholds)
- Pluggable transport management
- Mesh transport handling
- Mesh island advertisement propagation to unified DHT

### 6.2 Routing Policy Engine

Evaluates each packet or flow and decides:
- Route via BGP-X overlay (into bgpx0 TUN)
- Route via standard BGP (directly to WAN)
- Block

Rules can specify:
- Source device (IP, MAC)
- Destination (IP, CIDR, domain)
- Protocol/port
- Pool and path constraints inline with routing decision
- Cross-domain path specifications using `domain_segments`
- Default action for unmatched traffic

### 6.3 Node

A BGP-X **node** is any participant in the overlay network. Roles:

- **Relay node**: forwards encrypted packets; sees only previous and next hop
- **Entry node**: accepts client connections; knows client transport address, not destination
- **Exit node / Gateway**: translates overlay to public internet; knows destination, not source
- **Discovery node**: participates in unified DHT
- **Mesh node**: operates without ISP via mesh transport
- **Domain bridge node**: has endpoints in 2+ routing domains; bridges traffic between them using DOMAIN_BRIDGE hop type
- **Mesh island gateway**: domain bridge node specifically connecting a mesh island to clearnet

NodeID: `BLAKE3(Ed25519_public_key)`

### 6.4 Routing Domains

A **Routing Domain** is a named, addressable network segment with its own transport characteristics. BGP-X routes packets *between* routing domains, not just through one.

Types:

| Domain Type | Type ID | Description | Transport |
|---|---|---|---|
| clearnet | 0x00000001 | BGP-routed public internet | UDP/IP |
| bgpx-overlay | 0x00000002 | BGP-X onion layer (virtual) | Any |
| mesh:\<island_id\> | 0x00000003 | Named WiFi/LoRa/BLE mesh island | Radio |
| lora-regional:\<region_id\> | 0x00000004 | LoRa-only regional zone | LoRa |
| satellite:\<orbit_id\> | 0x00000005 | Satellite connectivity segment | Satellite |

Domain ID wire format: 8 bytes = type (4 bytes BE) + instance hash (4 bytes BE, BLAKE3(instance_string)[0:4] or 0x00000000 for singletons).

Domain bridge nodes connect routing domains. Any domain pair is bridgeable. Any path may traverse any combination of domains in any order any number of times.

### 6.5 DHT Pools

Pools are named, signed collections of nodes grouped by trust level or operational intent.

Types:
- **Public**: open DHT, anyone can query
- **Curated**: curator signs membership; elevated trust signal
- **Private**: membership not published; shared out-of-band
- **Semi-private**: discoverable but restricted join
- **Ephemeral**: temporary, session-scoped
- **Domain-scoped**: restricted to nodes in a specific routing domain
- **Domain-bridge**: composed of bridge nodes for a specific domain pair

### 6.6 Unified DHT

BGP-X operates **one unified DHT** spanning all routing domains:

- All nodes regardless of routing domain participate in the same Kademlia key space
- Domain bridge nodes with clearnet endpoints serve as the physical DHT storage infrastructure accessible from the internet
- Mesh-only nodes access the unified DHT via their bridge nodes' caches
- No separate "mesh DHT" — one DHT, accessible from all domains
- Domain-filtered queries allow clients to find nodes in specific routing domains efficiently

Record types in unified DHT:
- Node advertisements (keyed by NodeID)
- Pool advertisements (keyed by pool_id)
- Domain bridge records (keyed by ordered domain pair)
- Mesh island records (keyed by island_id)
- Pool curator key rotation records

### 6.7 Three Local Interfaces

**TUN Interface (bgpx0)**:
- Virtual network interface at OS level
- Traffic routed here is captured by daemon for overlay routing
- Applications require no modification
- Handles all IP protocols transparently

**SDK Socket** (`/var/run/bgpx/sdk.sock`):
- BGP-X native applications connect here
- JSON-RPC 2.0 with binary extensions
- Provides: stream open with path constraints (including cross-domain domain_segments), service registration, event subscription, identity management, domain/island status queries
- Also accessible as TCP endpoint for LAN devices connecting to router daemon

**Control Socket** (`/var/run/bgpx/control.sock`):
- Configuration client (bgpx-cli and GUIs) connects here
- JSON-RPC 2.0 text only
- Full daemon management: status, paths, nodes, pools, domains, domain bridges, mesh islands, policy, reputation

### 6.8 path_id

An 8-byte CSPRNG-generated identifier assigned to each path at construction time.

Purpose: enables return traffic routing without leaking path composition. Each relay node stores `path_id → predecessor_address` in memory. For cross-domain paths, domain bridge nodes additionally store `path_id → domain_a_predecessor` and `path_id → domain_b_predecessor` in their cross-domain path table. Return traffic is forwarded opaquely using these mappings across domain boundaries — intermediate relays do NOT decrypt return traffic.

### 6.9 Client Application

The `bgpx-cli` configuration client and any GUI tools. This is NOT a routing stack. It connects to the daemon via the Control Socket. Provides cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing.

### 6.10 SDK

A library that BGP-X native applications use to connect to the daemon SDK Socket. Exposes `RoutingDomain` in `PathConfig` and `DomainSegmentConfig`, enabling applications to specify cross-domain paths. The SDK does not implement its own routing — the daemon handles all construction, handshakes, and crypto.

---

## 7. Component Dependency Map

```
Standard App / SDK App
      │
      ├── (transparent) → TUN Interface (bgpx0)
      │                         │
      └── (SDK) → SDK Socket ──►┤
                                │
                        BGP-X Daemon (bgpx-node)
                                │
     ┌──────────────────────────┼─────────────────────────┐
     │                          │                         │
     ▼                          ▼                         ▼
Path Manager              Pool Manager           Domain Manager
(DHT queries,             (pool discovery,       (bridge nodes,
 node selection,           membership             island registry,
 cross-domain path         verification,          unified DHT access,
 construction,             domain-scoped          cross-domain
 bridge discovery)         pools)                 path_id table)
     │                          │                         │
     └──────────────────────────┼─────────────────────────┘
                                │
                                ▼
                       Reputation System
                       (local primary,
                        global advisory,
                        geo plausibility,
                        domain-aware events)
                                │
                                ▼ (UDP / mesh transport / pluggable transport)
                         BGP-X Relay Network
          (Entry → Relay(s) → [Domain Bridge(s)] → Exit/Service)
                                │
                                ▼ (public internet or mesh or satellite)
                           Destination
```

---

## 8. Hardware Ecosystem

BGP-X provides purpose-built hardware for every deployment context. All hardware is **adaptable by design**: reference implementations use current commodity components, but the firmware and daemon are hardware-agnostic. Future revisions may substitute different SoCs, custom BGP-X ASICs, or alternative radio chipsets without protocol changes.

### 8.1 BGP-X Router v1 — End User All-in-One

The primary user-facing product. **Replaces a standard home or office router entirely.** One device provides all BGP-X capabilities:

- WAN routing (clearnet domain)
- LAN routing and switching for all connected devices
- BGP-X overlay relay participation
- WiFi 802.11s mesh networking
- LoRa radio mesh (integrated SX1262)
- Bluetooth BLE mesh
- Domain bridge node (any interface combination)
- Mesh island gateway
- Satellite WAN (USB modem: Starlink, Iridium, Inmarsat)

Reference hardware: ARM dual-core Cortex-A53 @ 1.3 GHz (MT7981B), 512 MB RAM, TPM 2.0, SX1262 LoRa, MT7915 WiFi 6. Carrier boards: desktop, outdoor IP67, DIN rail, 1U rack. All deployment scenarios (mast, solar, vehicle, maritime, aerial, underground) use the outdoor IP67 carrier.

### 8.2 BGP-X Node v1 — Community Contributor

A compact multi-radio BGP-X node for community mesh expansion. **Adds to an existing network; does not replace a home router.** Configurable operating modes via software:

- Mesh Relay (no WAN needed): participates in mesh island, extends coverage
- Domain Bridge (with WAN): bridges clearnet to mesh island
- Community Gateway (with WAN + exit): provides clearnet exit for island
- Range Extension: low-overhead relay, prioritizes forwarding
- LoRa-only, WiFi-only, or all radios simultaneously

Solar and battery native. Default IP67 outdoor enclosure. Reference: low-power ARM Cortex-A53, 256-512 MB RAM, SX1262 LoRa, 802.11s WiFi, BLE, solar MPPT.

### 8.3 BGP-X Gateway v1 — Provider Infrastructure

High-throughput provider-grade exit node and domain bridge. **For ISPs, hosting providers, and enterprise operators.** Not a consumer product. Reference: ARM quad-core Cortex-A53 @ 1.8 GHz (MT7988A), 1 GB RAM, SFP+ uplink, 2×2.5GbE WAN, >1 Gbps throughput target.

### 8.4 BGP-X Client Node — Tier 2

Low-cost, battery-powered endpoint for connecting to BGP-X mesh networks. **Does not route traffic for others.**

Reference hardware: LILYGO T3S3, T-Beam, Heltec V3, RAK WisBlock.
Firmware: BGP-X Client Firmware (subset of protocol).
Use: Individual users, IoT sensors, portable operation.

### 8.5 BGP-X Adapter/Dongle — Tier 3

USB LoRa modem for existing computers. **Pure radio interface.** The BGP-X daemon runs on the host computer.

### 8.6 BGP-X OpenWrt Package

Software-only package for compatible third-party routers (GL.iNet GL-MT6000, Banana Pi BPi-R3, Raspberry Pi 4/5, x86 servers). Community members contribute without purchasing new hardware. Domain bridge capable with USB adapters.

---

## 9. N-Hop Unlimited

BGP-X imposes no protocol maximum on path length. The common header has no hop counter. No mechanism drops a packet after N hops. The only minimum is 3 hops.

Applications choose path length based on their threat model and latency tolerance. A 20-hop path across 4 routing domains is as valid as a 3-hop single-domain path.

**Default configurations (not enforced limits):**

| Path Type | Default Hops |
|---|---|
| Single-domain clearnet | 4 |
| Cross-domain, 2 segments + 1 bridge | 7 |
| Cross-domain, 3 segments + 2 bridges | 10 |
| High-security application-specified | Any |

---

## 10. Traffic Flow (Full End-to-End)

### 10.1 Standard App on LAN Device → Clearnet (Single Domain)

```
1. App sends packet to OS network stack
2. Routing policy engine matches rule: → BGP-X overlay
3. Packet enters TUN interface (bgpx0)
4. BGP-X daemon reads from TUN
5. Client constructs session keypair (ephemeral)
6. Client selects path from pool: [Entry E1] → [Relay R1] → [Relay R2] → [Exit G1]
7. Client performs handshakes with each hop in order (entry first)
8. Client generates path_id (8 bytes CSPRNG)
9. Client wraps packet in 4 onion layers, each containing path_id
10. Client sends outermost-encrypted packet to E1

11. E1 decrypts its layer → extracts path_id, next hop (R1)
    E1 stores: path_id → client_addr
    E1 forwards remaining ciphertext to R1

12. R1 decrypts its layer → extracts path_id, next hop (R2)
    R1 stores: path_id → E1_addr
    R1 forwards to R2

13. R2 decrypts its layer → extracts path_id, next hop (G1)
    R2 stores: path_id → R1_addr
    R2 forwards to G1

14. G1 decrypts final layer → sees destination + application payload
    G1 checks exit policy, resolves DNS via DoH, applies ECH if capable
    G1 connects to destination, sends request

15. Response arrives at G1
    G1 encrypts response using K_G1 (session key shared with client)
    G1 forwards encrypted blob toward R2 via path_id lookup

16. R2 receives return blob → path_id lookup → forwards to R1 (opaque, no decryption)
17. R1 receives return blob → path_id lookup → forwards to E1 (opaque)
18. E1 receives return blob → path_id lookup → forwards to client (opaque)
19. Client decrypts with K_G1 → gets plaintext response
20. Daemon writes response to TUN → OS delivers to application
```

### 10.2 Cross-Domain: Clearnet Client → Mesh Island Service

```
1. App sends packet; routing policy matches cross-domain rule
2. Daemon reads from TUN
3. Selects cross-domain path:
   [E1: clearnet relay] → [R1: clearnet relay] → [B1: domain bridge node]
   → [M1: mesh relay] → [M2: mesh relay] → [Service]

4. Performs handshakes (domain-agnostic handshake identical for all hops):
   - HANDSHAKE with E1 via UDP (clearnet)
   - HANDSHAKE with R1 via E1's established session (clearnet)
   - HANDSHAKE with B1 via R1's session (B1 has clearnet UDP endpoint; arrives as UDP)
   - HANDSHAKE with M1: client sends via B1's session; B1 delivers via mesh radio to M1
     M1 receives handshake via WiFi mesh / LoRa; sends response back via radio → B1 → client
   - HANDSHAKE with M2: same relay chain; M2 session established

5. Generates path_id (8 bytes CSPRNG)
6. Constructs 5-layer onion including DOMAIN_BRIDGE hop at B1 position:
   Layer 5 (outermost): encrypted for E1 (RELAY hop type)
   Layer 4: encrypted for R1 (RELAY)
   Layer 3: encrypted for B1 (DOMAIN_BRIDGE hop type, target domain = mesh:lima-district-1)
   Layer 2: encrypted for M1 (MESH_RELAY)
   Layer 1 (innermost): encrypted for Service (DELIVERY)

7. E1 decrypts → RELAY; stores path_id → client_addr; forwards to R1
8. R1 decrypts → RELAY; stores path_id → E1_addr; forwards to B1
9. B1 decrypts → DOMAIN_BRIDGE; target domain = mesh:lima-district-1
   B1 stores path_id → R1_addr (clearnet side)
   B1 stores path_id → M1_mesh_addr (mesh side) in cross-domain path table
   B1 selects mesh transport; forwards remaining onion via WiFi/LoRa to M1
10. M1 decrypts → MESH_RELAY; stores path_id → B1_mesh_addr; forwards to M2
11. M2 decrypts → DELIVERY; stores path_id → M1_mesh_addr; delivers to service

12. Service generates response; encrypts with K_service (session key shared with client)
13. M2: path_id lookup → M1_mesh_addr; forward blob via mesh (opaque)
14. M1: path_id lookup → B1_mesh_addr; forward blob via mesh (opaque)
15. B1: cross-domain path_id lookup → R1_addr (clearnet);
    forward blob via clearnet UDP to R1 (opaque, no decryption)
16. R1: path_id lookup → E1_addr; forward via clearnet UDP (opaque)
17. E1: path_id lookup → client_addr; forward via clearnet UDP (opaque)
18. Client decrypts with K_service → application receives response
```

The clearnet client has no mesh hardware. B1 handles all radio transmission. The mesh service never sees the client's clearnet IP. The clearnet relays never know the destination is a mesh service.

### 10.3 SDK App → BGP-X Native Service (Single Domain)

```
1. SDK app calls client.connect_stream("bgpx://service_id_hex")
2. SDK sends STREAM_OPEN to daemon via SDK Socket
3. Daemon resolves ServiceID via unified DHT
4. Path terminates at service's BGP-X entry node (no gateway needed)
5. End-to-end onion encryption throughout
6. SDK receives local socket endpoint
7. App reads/writes as normal TCP connection
```

### 10.4 Mesh Node → Clearnet (via Domain Bridge / Gateway)

```
1. Mesh device sends packet to BGP-X mesh daemon
2. Selects cross-domain path:
   [mesh relays] → [domain bridge/gateway] → [clearnet relays] → [exit]
3. Mesh hops: LoRa or WiFi 802.11s transport
4. Domain bridge: transitions from mesh domain to clearnet domain
5. Clearnet hops: standard UDP
6. Exit node delivers to clearnet destination
7. Return path via path_id chain back through all hops across domain boundary
```

---

## 11. Return Path Architecture

Return traffic uses a different architecture than outbound:

- **Client holds all N session keys** (one per hop).
- **Exit encrypts response** with the session key it shares with the client (`K_exit`).
- **Intermediate relays forward the encrypted blob without decryption** using `path_id → predecessor` mapping.
- **The blob is not re-encrypted at each hop** — it is already encrypted for the client.
- **Two independent sequence number spaces per session** (inbound/outbound) prevent nonce reuse.

This means only the exit node and the client can decrypt the response. Intermediate relays are opaque forwarders for return traffic.

---

## 12. Identity Model

### 12.1 Node Identity

```
(node_private_key, node_public_key)  ← Ed25519 long-term keypair
NodeID = BLAKE3(node_public_key)     ← 32 bytes
```

Used for: signing advertisements, establishing sessions, reputation tracking. Same keypair for all routing domains a node serves.

### 12.2 Client Identity

Ephemeral by default — fresh Ed25519 keypair generated per session, destroyed after handshake. Long-term client identity supported for authenticated services but not recommended for general use.

### 12.3 Service Identity

```
ServiceID = Ed25519 public key  ← stable, long-term
```

Services register reachability in unified DHT. Services can be in any routing domain (clearnet-accessible, mesh-island-local, or any domain).

### 12.4 Pool Curator Identity

```
curator_private_key, curator_public_key  ← Ed25519, separate from node key
```

Pool curators sign pool advertisements and member records. Key rotatable via dual-signature rotation records.

### 12.5 Routing Domain and Mesh Island Identity

```
Domain ID:    domain_type (uint32 BE) || BLAKE3(instance_string)[0:4] (uint32 BE)
Mesh Island:  island_id string (operator-chosen, unique by DHT collision detection)
```

Island IDs should be descriptive and geographically specific to minimize collision probability (e.g., `lima-san-isidro-north` not `community-1`).

---

## 13. Key Exchange and Session Establishment

BGP-X uses a Noise Protocol-inspired key exchange. **Domain-agnostic** — identical whether session is over UDP/IP, WiFi mesh, LoRa, BLE, or satellite.

**Process:**

1. Client generates ephemeral X25519 keypair per session.
2. For each hop, in order (entry first):
   - Client fetches node's static public key from signed DHT advertisement.
   - Client sends `HANDSHAKE_INIT`:
     - **Cleartext**: `client_ephemeral_pub` (32 bytes) — node needs this to compute `dh1`.
     - **Encrypted**: `session_id`, timestamp, versions, extensions.
   - Node sends `HANDSHAKE_RESP` encrypted with response key (requires both static+ephemeral).
   - Client sends `HANDSHAKE_DONE` with confirmation hash and `path_id`.

**Session key derivation:**
```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
session_key = HKDF-SHA256(dh1 || dh2, salt, info)
```

**Zeroization order (mandatory):**
- `client_ephemeral_priv` → after `dh2` computed
- `dh1`, `dh2` → after `session_key` derived
- `init_key`, `resp_key` → after handshake complete
- `node_ephemeral_priv` → after session established (node side)
- `session_key` → when session closed

**Cover Traffic**: No separate `cover_key`. **COVER uses `session_key`.** Applies in all routing domains.

For cross-domain paths: mesh relay handshakes are delivered via the domain bridge node's established session. The bridge node relays handshake packets into the mesh via radio. The mesh relay sees a standard session establishment — it does not know the requesting client is on clearnet.

Sessions exceeding 24 hours SHOULD trigger re-handshake on the same path.

---

## 14. Node Discovery — Unified DHT

BGP-X operates **one unified Kademlia-style DHT** spanning all routing domains. All nodes — clearnet, mesh island, satellite — participate in the same key space.

### What is Stored

- Signed node advertisement records (48hr TTL)
- Signed pool advertisement records (7-day TTL)
- Domain bridge records (24hr TTL)
- Mesh island records (24hr TTL)
- Pool curator key rotation records

### What is NOT Stored

- Client identities
- Session records
- Traffic metadata

### Bootstrap

**Internet mode**: Hardcoded bootstrap node list; DNS-based fallback (`_bgpx-bootstrap._udp.bgpx.network`) via DoH.

**Mesh-only mode**: Broadcast beacons (`MESH_BEACON`); no hardcoded IPs needed; any reachable node serves as bootstrap peer.

**Unified DHT for mesh**: Domain bridge nodes (gateway nodes) store and serve records for mesh-only nodes in the unified internet DHT. Mesh-only nodes access the unified DHT via their bridge nodes' caches. There is no separate mesh DHT and no synchronization between separate DHTs.

---

## 15. Trust Model

### 15.1 Adversary Assumptions

- May control any subset of relay nodes in any routing domain
- May observe all public internet traffic (global passive adversary)
- May operate malicious entry or exit nodes
- May operate malicious domain bridge nodes
- May observe mesh radio traffic
- Does NOT compromise the client's device

### 15.2 Privacy Guarantees

With N-hop path and pool/domain diversity enforcement:

- No single relay can link sender to destination (split knowledge enforced by onion layers)
- Domain bridge nodes see only their immediate predecessor in each domain — not the full path
- path_id enables return routing without leaking path composition across domain boundaries
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic (same session_key as RELAY, externally identical) disrupts timing correlation
- Cross-domain paths: adversary observing clearnet cannot link to mesh island traffic without simultaneously observing mesh radio

### 15.3 What BGP-X Does NOT Guarantee

- Protection against compromised client device
- Protection against application-layer identity leakage
- Timing correlation resistance against sufficiently resourced global adversary with multi-domain observation
- Legal protection for any activity
- Physical connectivity (BGP-X provides privacy for whatever connectivity exists)

---

## 16. Threat Model Summary

| Adversary | BGP-X Protection |
|---|---|
| ISP (passive) | ✅ Encrypted UDP to entry node; ISP sees participation but not destination |
| ISP (active MITM) | ✅ Authentication tags detect modification; injection fails without session key |
| Single malicious relay | ✅ Cannot link source to destination |
| Multiple colluding relays (minority) | ✅ Diversity constraints prevent single operator controlling entry+exit |
| Entry + exit simultaneously controlled | ⚠️ Pool/domain diversity and operator diversity reduce probability |
| Global passive adversary | ⚠️ Timing correlation residual risk; cover traffic helps |
| Malicious exit node | ⚠️ Sees destination; ECH hides domain from SNI; HTTPS hides content |
| Malicious domain bridge node | ⚠️ Sees only immediate predecessor in each domain; same as relay node |
| Pool curator compromised | ⚠️ Individual node verification provides fallback; cross-pool diversity limits damage |
| Compromised client device | ❌ Out of scope |
| Application-layer leakage | ❌ Out of scope |
| Cross-domain radio observation | ⚠️ Timing correlation across domains; harder than single domain; cover traffic helps |
| Mesh island isolation (availability) | ⚠️ Multiple bridge nodes from different operators mitigate; not a privacy failure |

---

## 17. Comparison to Existing Systems

### 17.1 vs. VPN

A VPN routes all traffic through a single trusted provider. The VPN provider sees source, destination, and content (unless HTTPS). BGP-X distributes trust across N hops with no single party able to see both ends.

### 17.2 vs. Tor

| Dimension | Tor | BGP-X |
|---|---|---|
| Directory | Centralized authorities | Decentralized DHT |
| Path length | Fixed 3 hops | Configurable N hops, no maximum |
| Transport | TCP | UDP + reliability layer |
| Network layer | App-layer proxy | Full IP overlay |
| Firmware support | No | Yes |
| Multiplexing | No (one circuit = one stream) | Yes (N streams per path) |
| Congestion control | Inherited from TCP | Native |
| Cover traffic | Off by default | Pluggable module |
| Mesh transport | No | Yes (WiFi 802.11s, LoRa, BLE) |
| Cross-domain routing | No | Yes (clearnet ↔ mesh) |
| Pool-based trust | No | Yes |
| Hardware ecosystem | No | Yes (Router, Node, Gateway, Client, Adapter) |
| Deployment | Per-application or per-device | Router-level (all devices) |
| Entry points | Internet only | Clearnet, overlay, mesh |
| N-hop unlimited | No (fixed 3) | Yes |
| DHT | Distributed directory | Unified Kademlia DHT |
| Hardware target | General-purpose | Router + embedded hardware |
| ECH at exit | No | Yes |
| Cover traffic | Padding only | COVER uses `session_key` |
| Decentralization | Directory authorities | Fully decentralized DHT |
| Transport Layer | TCP only | UDP native with reliability layer; carries all IP protocols |
| Network Layer | SOCKS5 proxy (application must be aware) | TUN interface (all applications, all protocols, transparent) |

### 17.3 vs. I2P (Invisible Internet Project)

I2P uses garlic routing (multiple messages bundled) and is primarily designed for internal services. BGP-X is designed for full internet access with gateway exit nodes, and integrates at the network layer rather than the application layer.

### 17.4 vs. cjdns/Yggdrasil

cjdns and Yggdrasil provide public key addressing with mesh transport but no onion routing and no anonymity. BGP-X adds onion encryption, multi-hop paths, pool trust domains, and hardware targets.

### 17.5 vs. Reticulum

Reticulum provides multi-transport mesh capability (LoRa, WiFi, serial) but no onion routing, no clearnet exit model, and no hardware ecosystem. BGP-X integrates multi-transport mesh AND onion routing AND clearnet exit AND hardware.

### 17.6 vs. SCION

SCION (Scalability, Control, and Isolation On Next-generation networks) provides path-aware, source-selected routing at the infrastructure level, requiring ISP adoption. BGP-X achieves similar path control as an overlay without any infrastructure changes.

---

## 18. BGP and BGP-X Layer Relationship

BGP-X does not participate in BGP. BGP-X uses the public internet — which is routed by BGP — as one transport medium among several. BGP is unmodified dumb transport.

```
BGP-X Application Layer             ← BGP-X apps, SDK
BGP-X Inter-Protocol Domain Router  ← cross-domain path construction, domain bridges
BGP-X Overlay Routing               ← path selection, onion encryption, unified DHT
BGP-X Transport (UDP/IP)            ← BGP-X packets as UDP datagrams (clearnet)
BGP-X Transport (WiFi/LoRa/BLE)    ← BGP-X packets as radio frames (mesh)
         │
         │  UDP datagrams are ordinary UDP/7474 to BGP routers
         ▼
BGP Routing Layer                   ← ISP routing, AS-level decisions
Physical / Link Layer               ← fiber, radio, physical infrastructure
```

For mesh deployments, the transport layer is replaced by radio with zero BGP involvement for intra-mesh traffic. Cross-domain paths using internet relay pools still use BGP as transport for the clearnet segments.

---

## 19. Deployment Modes

| Mode | BGP Role | Mesh Transport | ISP Required | Use Case |
|---|---|---|---|---|
| Dual-Stack Router | Full transport | Optional | Yes | Privacy with selective bypass |
| BGP-X Only Router | Full transport for overlay | Optional | Yes | Maximum enforcement |
| Standalone Device | Full transport | No | Yes | No router access |
| Mesh Node | None | Required | No | Community without ISP |
| Gateway Node (Domain Bridge) | WAN for clearnet exit | Both | At gateway | Bridge mesh to internet |
| Range Extension Node | None | Single transport | No | Coverage extension via Node v1 |

---

## 20. BGP Replacement Spectrum

| Level | BGP Role | BGP-X Role | User BGP Exposure |
|---|---|---|---|
| 0 | Full transport | Encryption/anonymization layer | Full (via ISP) |
| 1 | None (intra-mesh) | Full intra-mesh routing | None |
| 2 | Gateway WAN only | Full overlay + gateway exit | None (gateway mediates) |
| 3 | Gap-filling transport | Full overlay across islands and gaps | None |
| 4 | Transport per mode | Full overlay across all segments | Variable by position |
| 5 | Aerial/satellite backhaul | Near-complete overlay | Minimal |
| 6 | Physical infrastructure | Complete privacy overlay | None (ISP decisions replaced) |

Domain routing enhances Levels 2 and above: mesh island users have zero BGP exposure AND clearnet users reaching mesh services cannot be distinguished from standard BGP-X overlay traffic by BGP routers.

---

## 21. Application Type Model

### Standard Applications (BGP-X Unaware)

No modification required. Routing policy can specify cross-domain paths transparently — the application experiences standard network behavior while the daemon handles domain transitions invisibly.

### BGP-X Native Applications (SDK)

Full `RoutingDomain` and `DomainSegmentConfig` support in `PathConfig`. Can specify cross-domain segments, domain transitions, preferred/avoided domains, and mesh island service addressing. Can register as BGP-X native services in any routing domain.

### Configuration Client (bgpx-cli, GUIs)

Connects to daemon Control Socket. Cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing, bridge pair health monitoring.

---

## 22. Geographic Plausibility Scoring

RTT-based verification of node region claims. No external database required. Domain-specific thresholds:

- **Clearnet nodes**: internet RTT thresholds by region pair
- **WiFi mesh nodes**: 1-20ms per hop expected
- **LoRa mesh nodes**: 100ms-5000ms per hop expected (not flagged for high latency)
- **Domain bridge nodes**: checked independently for each domain they serve
- **Satellite nodes**: exempt from geographic thresholds (return 0.5 neutral)

Not a hard exclusion — reputation signal only. Discrepancy generates `GEO_SUSPICIOUS` or `GEO_IMPLAUSIBLE` reputation events.

---

## 23. Extension Points

| Extension | Default | Version |
|---|---|---|
| Cover traffic | Disabled | v1 |
| Pluggable transport | Disabled | v1 |
| Stream multiplexing | Enabled | v1 |
| Path quality reporting (with domain ID) | Enabled | v1 |
| ECH at exit nodes | Enabled (if ech_capable=true) | v1 |
| Geographic plausibility scoring | Enabled | v1 |
| Pool support (including domain-scoped) | Enabled | v1 |
| Mesh transport | Disabled (enabled in mesh firmware) | v1 |
| Cross-domain routing | Enabled (requires bridge nodes) | v1 |
| Domain bridge support | Enabled (when 2+ routing_domains configured) | v1 |
| Mesh island routing | Enabled (in mesh firmware) | v1 |
| Advertisement withdrawal | Supported | v1 |
| Session re-handshake (24hr) | Enabled | v1 |
| Economic incentive layer | Not in scope | — |

All extensions are v1. Nothing is deferred.

---

## 24. Design Decisions Log

| Decision | Rationale |
|---|---|
| Three equal routing domains | No entry point should be privileged; any-to-any connectivity is the network's fundamental value |
| N-hop unlimited | Privacy requirements vary; no protocol cap appropriate; applications choose based on threat model |
| Unified DHT (not two) | One discovery layer for all domains is simpler, more consistent, and eliminates sync complexity |
| Domain bridge node as regular hop | Same onion encryption applies; privacy properties identical to relay; implementation reuses existing pipeline |
| DOMAIN_BRIDGE hop type (0x06) | Explicit in onion layer; bridge node knows to switch transport; inner layers remain encrypted |
| Cross-domain path_id routing | Same mechanism as single-domain; extends naturally across domain boundaries |
| Domain bridge selection scoring | Specialized scoring for multi-transport nodes; uptime must be checked per domain |
| Mesh island as DHT entity | Islands must be discoverable by clearnet clients without any special client hardware |
| Any-to-any connectivity | Network value grows with every new domain and bridge node; no artificial constraints |
| Domain-agnostic handshake | One handshake protocol for all transports; reduces implementation complexity and audit surface |
| No re-encryption at domain boundaries | Bridge node forwards inner layers without modification; crypto properties maintained |
| UDP as primary transport | Lower overhead, multi-hop latency, custom reliability layer |
| Client-selected paths | Removes network-level adversary from path decisions |
| BLAKE3 for hashing | High performance; suitable for embedded/firmware |
| Ed25519 for signatures | Compact, fast, well-audited, deterministic |
| X25519 for key exchange | State-of-the-art ECDH, widely supported |
| ChaCha20-Poly1305 | Constant-time; no AES-NI required; fast on embedded |
| No separate cover_key | COVER uses session_key; external indistinguishability preserved in all domains |
| path_id for return routing | Enables bidirectional sessions without leaking path composition across domain boundaries |
| DHT Pools vs. guard nodes | Pools provide trust isolation without creating observable long-term entry relationships |
| Geographic plausibility over external DB | No external dependency; RTT is unforgeable below speed of light |
| Domain-specific plausibility thresholds | LoRa is inherently high-latency; satellite is orbital; different physics, different expectations |
| Local-first reputation | Local observations authoritative; global is advisory |
| Router-centric architecture | All devices protected without per-device config |
| Unified hardware platform | Single PCB with variants reduces manufacturing complexity |
| ECH at exit layer | Hides domain name from SNI when destination supports it |
| Mesh transport native | Enables ISP-free operation; extends BGP-X reach beyond internet |
| Satellite = clearnet | Commercial satellite internet is clearnet domain 0x00000001; domain 0x00000005 reserved for future BGP-X-native satellite network |

---

## 25. Prior Art Acknowledgment

BGP-X integrates concepts from multiple prior systems:

- **cjdns**: inspired public key addressing and NodeID-as-address model
- **Yggdrasil**: informed DHT routing design
- **Reticulum**: validated multi-transport approach for low-bandwidth mesh
- **Tor**: foundational onion routing and circuit model
- **I2P**: garlic routing and native service model
- **Kademlia DHT**: node discovery algorithm
- **Meshtastic**: demonstrated LoRa mesh viability for community networks

BGP-X's innovation is unification: combining all of these into one coherent system with inter-protocol domain routing, hardware targets, pool-based trust domains, router-level deployment, and no protocol-level restrictions on path length, domain count, or domain ordering.

---

## 26. Implementation Notes

The reference implementation is in Rust. Design priorities:

- **Memory safety**: no undefined behavior in packet processing path; no raw pointer arithmetic in parsing
- **Zero-copy where possible**: packet buffers passed by reference through relay pipeline
- **Async I/O**: non-blocking via Tokio; receiver threads use `SO_REUSEPORT` for parallel receive
- **Explicit error handling**: no panics in relay path; all errors handled explicitly
- **Constant-time cryptography**: all crypto operations use constant-time implementations from audited libraries; no timing side channels
- **Zeroization**: all key material cleared from memory in mandatory order using `zeroize` crate
- **Domain-agnostic crypto**: identical ChaCha20-Poly1305, X25519, Ed25519, BLAKE3 in all routing domains; no domain-specific key derivation
- **Unified DHT participation**: one DHT instance; no per-domain DHT splitting
- **Cross-domain path table**: sharded concurrent hash map for `path_id → cross-domain` routing, separate from single-domain path table

See `/node/node.md`, `/node/api.md`, `/security/crypto_spec.md`, and `/data-plane/forwarding.md` for implementation detail.
