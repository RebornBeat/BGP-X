# BGP-X System Architecture

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

## 2. System Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│  Standard apps (transparent) + SDK apps (aware)        │
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
│  UDP/IP (internet), WiFi 802.11s, LoRa, BLE, Ethernet  │
│  Pluggable transport for obfuscation                   │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Core Components

### 3.1 BGP-X Router Daemon (bgpx-node)

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

### 3.2 Routing Policy Engine

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

### 3.3 Node

A BGP-X **node** is any participant in the overlay network. Roles:

- **Relay node**: forwards encrypted packets; sees only previous and next hop
- **Entry node**: accepts client connections; knows client transport address, not destination
- **Exit node / Gateway**: translates overlay to public internet; knows destination, not source
- **Discovery node**: participates in unified DHT
- **Mesh node**: operates without ISP via mesh transport
- **Domain bridge node**: has endpoints in 2+ routing domains; bridges traffic between them using DOMAIN_BRIDGE hop type
- **Mesh island gateway**: domain bridge node specifically connecting a mesh island to clearnet

NodeID: `BLAKE3(Ed25519_public_key)`

### 3.4 Routing Domains

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

### 3.5 DHT Pools

Pools are named, signed collections of nodes grouped by trust level or operational intent.

Types:
- **Public**: open DHT, anyone can query
- **Curated**: curator signs membership; elevated trust signal
- **Private**: membership not published; shared out-of-band
- **Semi-private**: discoverable but restricted join
- **Ephemeral**: temporary, session-scoped
- **Domain-scoped**: restricted to nodes in a specific routing domain (new in v1)
- **Domain-bridge**: composed of bridge nodes for a specific domain pair (new in v1)

### 3.6 Unified DHT

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

### 3.7 Three Local Interfaces

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

### 3.8 path_id

An 8-byte CSPRNG-generated identifier assigned to each path at construction time.

Purpose: enables return traffic routing without leaking path composition. Each relay node stores `path_id → predecessor_address` in memory. For cross-domain paths, domain bridge nodes additionally store `path_id → domain_a_predecessor` and `path_id → domain_b_predecessor` in their cross-domain path table. Return traffic is forwarded opaquely using these mappings across domain boundaries — intermediate relays do NOT decrypt return traffic.

### 3.9 Client Application

The `bgpx-cli` configuration client and any GUI tools. This is NOT a routing stack. It connects to the daemon via the Control Socket. Provides cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing.

### 3.10 SDK

A library that BGP-X native applications use to connect to the daemon SDK Socket. Exposes `RoutingDomain` in `PathConfig` and `DomainSegmentConfig`, enabling applications to specify cross-domain paths. The SDK does not implement its own routing — the daemon handles all construction, handshakes, and crypto.

---

## 4. Traffic Flow (Full End-to-End)

### 4.1 Standard App on LAN Device → Clearnet (Single Domain)

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

### 4.2 Cross-Domain: Clearnet Client → Mesh Island Service

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

### 4.3 SDK App → BGP-X Native Service (Single Domain)

```
1. SDK app calls client.connect_stream("bgpx://service_id_hex")
2. SDK sends STREAM_OPEN to daemon via SDK Socket
3. Daemon resolves ServiceID via unified DHT
4. Path terminates at service's BGP-X entry node (no gateway needed)
5. End-to-end onion encryption throughout
6. SDK receives local socket endpoint
7. App reads/writes as normal TCP connection
```

### 4.4 Mesh Node → Clearnet (via Domain Bridge / Gateway)

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

## 5. Identity Model

### 5.1 Node Identity

```
(node_private_key, node_public_key)  ← Ed25519 long-term keypair
NodeID = BLAKE3(node_public_key)     ← 32 bytes
```

Used for: signing advertisements, establishing sessions, reputation tracking. Same keypair for all routing domains a node serves.

### 5.2 Client Identity

Ephemeral by default — fresh Ed25519 keypair generated per session, destroyed after handshake. Long-term client identity supported for authenticated services but not recommended for general use.

### 5.3 Service Identity

```
ServiceID = Ed25519 public key  ← stable, long-term
```

Services register reachability in unified DHT. Services can be in any routing domain (clearnet-accessible, mesh-island-local, or any domain).

### 5.4 Pool Curator Identity

```
curator_private_key, curator_public_key  ← Ed25519, separate from node key
```

Pool curators sign pool advertisements and member records. Key rotatable via dual-signature rotation records.

### 5.5 Routing Domain and Mesh Island Identity

```
Domain ID:    domain_type (uint32 BE) || BLAKE3(instance_string)[0:4] (uint32 BE)
Mesh Island:  island_id string (operator-chosen, unique by DHT collision detection)
```

Island IDs should be descriptive and geographically specific to minimize collision probability (e.g., `lima-san-isidro-north` not `community-1`).

---

## 6. Trust Model

### 6.1 Adversary Assumptions

- May control any subset of relay nodes in any routing domain
- May observe all public internet traffic (global passive adversary)
- May operate malicious entry or exit nodes
- May operate malicious domain bridge nodes
- May observe mesh radio traffic
- Does NOT compromise the client's device

### 6.2 Privacy Guarantees

With N-hop path and pool/domain diversity enforcement:

- No single relay can link sender to destination (split knowledge enforced by onion layers)
- Domain bridge nodes see only their immediate predecessor in each domain — not the full path
- path_id enables return routing without leaking path composition across domain boundaries
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic (same session_key as RELAY, externally identical) disrupts timing correlation
- Cross-domain paths: adversary observing clearnet cannot link to mesh island traffic without simultaneously observing mesh radio

### 6.3 What BGP-X Does NOT Guarantee

- Protection against compromised client device
- Protection against application-layer identity leakage
- Timing correlation resistance against sufficiently resourced global adversary with multi-domain observation
- Legal protection for any activity
- Physical connectivity (BGP-X provides privacy for whatever connectivity exists)

---

## 7. Threat Model Summary

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

## 8. BGP and BGP-X Layer Relationship

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

## 9. Deployment Modes

| Mode | BGP Role | Mesh Transport | ISP Required | Use Case |
|---|---|---|---|---|
| Dual-Stack Router | Full transport | Optional | Yes | Privacy with selective bypass |
| BGP-X Only Router | Full transport for overlay | Optional | Yes | Maximum enforcement |
| Standalone Device | Full transport | No | Yes | No router access |
| Mesh Node | None | Required | No | Community without ISP |
| Gateway Node (Domain Bridge) | WAN for clearnet exit | Both | At gateway | Bridge mesh to internet |
| Broadcast Amplifier | None | Single transport | No | Range extension |

All modes support cross-domain routing through domain bridge configuration. A Mesh Node without a bridge node operates in offline mode for cross-domain traffic.

---

## 10. BGP Replacement Spectrum

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

## 11. Application Type Model

### Standard Applications (BGP-X Unaware)

No modification required. Routing policy can specify cross-domain paths transparently — the application experiences standard network behavior while the daemon handles domain transitions invisibly.

### BGP-X Native Applications (SDK)

Full `RoutingDomain` and `DomainSegmentConfig` support in `PathConfig`. Can specify cross-domain segments, domain transitions, preferred/avoided domains, and mesh island service addressing. Can register as BGP-X native services in any routing domain.

### Configuration Client (bgpx-cli, GUIs)

Connects to daemon Control Socket. Cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing, bridge pair health monitoring.

---

## 12. Hardware Taxonomy

### Unified BGP-X Node Platform

Single PCB design with carrier board and enclosure variants.

**Compute**: MT7981B ARM Cortex-A53 dual-core 1.3GHz, 512MB DDR4, 128MB NAND + 8GB eMMC, TPM 2.0, hardware crypto

**Radios**: WiFi 6 (MT7915, 802.11s + AP), LoRa (SX1262, sub-GHz), optional Bluetooth 5.0

**Enclosure variants**: Indoor Desktop, Outdoor Mast (IP67), Maritime (IP68), Industrial/ATEX, Mobile/Vehicle

**Firmware variants**: bgpx-mesh-node, bgpx-gateway-node (domain bridge with clearnet exit), bgpx-amplifier

**Gateway variant**: Upgraded to MT7988A quad-core + 1GB RAM + WAN ports + SFP+ + mPCIe (4G/5G)

**Amplifier variant**: MCU-only (STM32H7), single radio, <800mW, no routing intelligence

### Existing Compatible Hardware

Full BGP-X daemon runs on: GL.iNet routers, Raspberry Pi, Orange Pi, Banana Pi BPi-R3, x86 mini PCs, OpenWrt-compatible consumer routers, industrial outdoor routers, 4G/5G cellular routers. Meshtastic hardware (ESP32/nRF52840) serves as LoRa radio modems via BGP-X adaptation layer.

---

## 13. Geographic Plausibility Scoring

RTT-based verification of node region claims. Domain-specific thresholds:

- **Clearnet nodes**: internet RTT thresholds by region pair (EU-EU: 10-100ms; NA-EU: 100-240ms; etc.)
- **WiFi mesh nodes**: 1-20ms per hop expected; higher RTT is suspicious
- **LoRa mesh nodes**: 100ms-5000ms per hop expected; NOT flagged for high latency
- **Domain bridge nodes**: checked independently for each domain they serve
- **Satellite nodes**: exempt from geographic thresholds; return neutral score 0.5

No external database required. RTT is unforgeable below the speed of light. Discrepancy generates reputation events (GEO_SUSPICIOUS, GEO_IMPLAUSIBLE). Not a hard exclusion — reputation signal only.

---

## 14. Extension Points

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

## 15. Design Decisions Log

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

---

## 16. Prior Art Acknowledgment

BGP-X integrates concepts from multiple prior systems:

- **cjdns**: inspired public key addressing and NodeID-as-address model
- **Yggdrasil**: informed DHT routing design
- **Reticulum**: validated multi-transport approach for low-bandwidth mesh
- **Tor**: foundational onion routing and circuit model
- **I2P**: garlic routing and native service model
- **Kademlia DHT**: node discovery algorithm
- **Meshtastic**: demonstrated LoRa mesh viability for community networks

BGP-X's innovation is unification: combining all of these into one coherent system with inter-protocol domain routing, hardware targets, pool-based trust domains, router-level deployment, and no protocol-level restrictions on path length, domain count, or domain ordering.
