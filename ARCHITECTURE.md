# BGP-X System Architecture

This document describes the full system architecture of BGP-X: a router-level privacy overlay network.

---

## 1. Design Philosophy

BGP-X is built on five non-negotiable principles:

### 1.1 No single party should see both endpoints of a connection

This is the foundational property. No node in the BGP-X network, no ISP, and no gateway can link a source identity to a destination. This is enforced at the protocol level, not by policy.

### 1.2 Identity is cryptographic, not location-based

BGP-X separates identity from location:

- **Location** is handled by underlying IP/BGP (opaque to BGP-X clients)
- **Identity** is a public key — self-certifying, portable, and optionally ephemeral

### 1.3 Routing decisions belong to the sender

BGP delegates path selection to the network. BGP-X delegates path selection to the client. The sender constructs the complete path before transmitting a single packet.

### 1.4 The existing internet is transport, not adversary

BGP-X uses the public internet as a raw packet transport layer and builds all privacy and routing logic above it. For mesh deployments, BGP-X uses local radio transport with zero ISP involvement.

### 1.5 Layered security, not security by obscurity

Every privacy property must be cryptographically enforceable. Honest operators improve performance; dishonest operators must fail to compromise privacy.

### 1.6 BGP-X is infrastructure, not a tool

BGP-X runs on the network router and protects all connected devices transparently. It is not a per-application tool. The daemon is the single BGP-X routing stack — all other components (SDK apps, config client) are clients of the daemon.

---

## 2. System Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│  Standard apps (transparent) + SDK apps (aware)        │
├─────────────────────────────────────────────────────────┤
│                Routing Policy Engine                    │
│  Per-flow decision: BGP-X overlay vs. standard routing  │
├─────────────────────────────────────────────────────────┤
│                   Identity Layer                        │
│  Public key identities, resolution, rotation           │
├─────────────────────────────────────────────────────────┤
│                   Overlay Routing Layer                 │
│  Path construction, onion encryption, forwarding       │
│  DHT Pools, geographic plausibility, reputation        │
├─────────────────────────────────────────────────────────┤
│                   Gateway Layer                         │
│  Overlay → Public Internet translation                  │
│  ECH support, DNS resolver, exit policy enforcement    │
├─────────────────────────────────────────────────────────┤
│                   Transport Layer                       │
│  UDP/IP (internet), WiFi 802.11s, LoRa, BLE, Ethernet │
│  Pluggable transport for obfuscation                   │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Core Components

### 3.1 BGP-X Router Daemon (bgpx-node)

The single BGP-X routing stack. Runs on the router or standalone device. All other components are clients of this daemon.

Responsibilities:
- Path construction and management
- Session establishment (handshake with each hop)
- Onion encryption and decryption
- DHT participation and node discovery
- Pool management
- Routing policy enforcement
- TUN interface management
- SDK Socket server
- Control Socket server
- Reputation tracking
- Geographic plausibility measurement
- Pluggable transport management
- Mesh transport handling

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
- Default action for unmatched traffic

### 3.3 Node

A BGP-X **node** is any participant in the overlay network. Roles:

- **Relay node**: forwards encrypted packets; sees only previous and next hop
- **Entry node**: accepts client connections; knows client IP, not destination
- **Exit node / Gateway**: translates overlay to public internet; knows destination, not source
- **Discovery node**: participates in DHT
- **Mesh node**: operates without ISP via mesh transport
- **Gateway node**: bridges mesh to clearnet

NodeID: `BLAKE3(Ed25519_public_key)`

### 3.4 DHT Pools

Pools are named, signed collections of nodes grouped by trust level or operational intent.

Types:
- **Public**: open DHT, anyone can query
- **Curated**: curator signs membership; elevated trust signal
- **Private**: membership not published; shared out-of-band
- **Semi-private**: discoverable but restricted join
- **Ephemeral**: temporary, session-scoped

Pools enable:
- Multi-segment paths with different trust tiers per segment
- Double-exit architecture (two independent exit pools in sequence)
- Geographic jurisdiction routing
- Pool randomization for traffic diversity
- Coverage gap bridging between mesh islands via internet relay pools

### 3.5 Three Local Interfaces

**TUN Interface (bgpx0)**:
- Virtual network interface at OS level
- Traffic routed here is captured by daemon for overlay routing
- Applications require no modification
- Handles all IP protocols transparently

**SDK Socket** (`/var/run/bgpx/sdk.sock`):
- BGP-X native applications connect here
- JSON-RPC 2.0 with binary extensions
- Provides: stream open with path constraints, service registration, event subscription, identity management
- Also accessible as TCP endpoint for LAN devices connecting to router daemon

**Control Socket** (`/var/run/bgpx/control.sock`):
- Configuration client (bgpx-cli and GUIs) connects here
- JSON-RPC 2.0 text only
- Full daemon management: status, paths, nodes, pools, policy, reputation

### 3.6 path_id

An 8-byte CSPRNG-generated identifier assigned to each path at construction time.

Purpose: enables return traffic routing without leaking path composition. Each relay node stores `path_id → predecessor_address` in memory. Return traffic is forwarded opaquely using this mapping — intermediate relays do not decrypt return traffic.

### 3.7 Client Application

The `bgpx-cli` configuration client and any GUI tools. This is NOT a routing stack. It connects to the daemon via the Control Socket and provides management capabilities.

### 3.8 SDK

A library that BGP-X native applications use to connect to the daemon SDK Socket. The SDK does not implement path construction, onion encryption, or DHT — these are done by the daemon. The SDK translates high-level API calls into daemon protocol messages.

For mobile/embedded environments, the SDK offers an embedded mode where the full daemon runs in-process.

---

## 4. Traffic Flow (Full End-to-End)

### 4.1 Standard App on LAN Device → Clearnet

```
1. App sends packet to OS network stack
2. Routing policy engine matches rule: → BGP-X overlay
3. Packet enters TUN interface (bgpx0)
4. BGP-X daemon reads from TUN
5. Client constructs session keypair (ephemeral)
6. Client selects path from pool: [Entry E1] → [Relay R1] → [Relay R2] → [Exit G1]
7. Client performs handshakes with each hop in order (entry first, then each subsequent hop via established sessions)
8. Client generates path_id (8 bytes random)
9. Client wraps packet in 4 onion layers, each containing path_id
10. Client sends outermost-encrypted packet to E1

11. E1 decrypts its layer → extracts path_id, next hop (R1)
    E1 stores: path_id → client_addr (in-memory, for return routing)
    E1 forwards remaining ciphertext to R1

12. R1 decrypts its layer → extracts path_id, next hop (R2)
    R1 stores: path_id → E1_addr
    R1 forwards to R2

13. R2 decrypts its layer → extracts path_id, next hop (G1)
    R2 stores: path_id → R1_addr
    R2 forwards to G1

14. G1 decrypts final layer → sees destination (example.com) and application payload
    G1 checks exit policy (ECH if capable, DNS DoH resolution, ECS stripping)
    G1 connects to example.com, sends request

15. Response arrives at G1
    G1 encrypts response using session key shared with client
    G1 forwards encrypted blob toward R2 (using path_id → R2 mapping)

16. R2 receives return blob → looks up path_id → forwards to R1 (opaque, no decryption)
17. R1 receives return blob → looks up path_id → forwards to E1 (opaque)
18. E1 receives return blob → looks up path_id → forwards to client (opaque)
19. Client receives blob encrypted with K_G1 → decrypts → gets plaintext response
20. Daemon writes response to TUN → OS delivers to application
```

### 4.2 SDK App → BGP-X Native Service

```
1. SDK app calls client.connect_stream("bgpx://service_id_hex")
2. SDK sends STREAM_OPEN to daemon via SDK Socket
3. Daemon resolves ServiceID via DHT
4. Path terminates at service's BGP-X entry node (no gateway needed)
5. End-to-end onion encryption throughout
6. SDK receives local socket endpoint
7. App reads/writes as normal TCP connection
```

### 4.3 Mesh Node → Clearnet (via Gateway)

```
1. Mesh device sends packet to router running BGP-X mesh firmware
2. BGP-X daemon selects path: [Mesh relay] → [Gateway] → [Internet relay] → [Exit]
3. Mesh hops use LoRa or WiFi 802.11s transport
4. Gateway bridges mesh to internet
5. Exit node delivers to clearnet destination
6. Return path follows path_id chain back through all hops
```

---

## 5. Identity Model

### 5.1 Node Identity

```
(node_private_key, node_public_key)  ← Ed25519 long-term keypair
NodeID = BLAKE3(node_public_key)     ← 32 bytes
```

Used for: signing advertisements, establishing sessions, reputation tracking.

### 5.2 Client Identity

Ephemeral by default — fresh Ed25519 keypair generated per session, destroyed after:

```
client_ephemeral_priv → zeroize after dh2 computed
```

Long-term client identity supported for authenticated services but not required and not recommended for general use.

### 5.3 Service Identity

```
ServiceID = Ed25519 public key  ← stable, long-term
```

Services register reachability in DHT. Infrastructure can change without changing ServiceID.

### 5.4 Pool Curator Identity

```
curator_private_key, curator_public_key  ← Ed25519, separate from node key
```

Pool curators sign pool advertisements and member records. Key rotation uses dual-signature records (old key + new key both sign the rotation record).

---

## 6. Trust Model

### 6.1 Adversary Assumptions

- May control any subset of relay nodes
- May observe all public internet traffic (global passive adversary)
- May operate malicious entry or exit nodes
- May control pool membership if curator is compromised
- Does NOT compromise the client's device

### 6.2 Privacy Guarantees

With N-hop path and pool diversity enforcement:

- No single relay can link sender to destination (split knowledge enforced by onion layers)
- path_id enables return routing without leaking path composition to intermediate relays
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic disrupts timing correlation (raises cost, does not eliminate)
- Pool diversity: even with compromised curator, individual node verification prevents pure pool-based attack

### 6.3 What BGP-X Does NOT Guarantee

- Protection against compromised client device
- Protection against application-layer identity leakage
- Timing correlation resistance against sufficiently resourced global adversary
- Legal protection for any activity

---

## 7. Threat Model Summary

| Adversary | BGP-X Protection |
|---|---|
| ISP (passive) | ✅ Encrypted UDP to entry node; ISP sees participation but not destination |
| ISP (active MITM) | ✅ Authentication tags detect modification; injection fails without session key |
| Single malicious relay | ✅ Cannot link source to destination |
| Multiple colluding relays (minority) | ✅ Diversity constraints prevent single operator controlling entry+exit |
| Entry + exit simultaneously controlled | ⚠️ Pool diversity and operator diversity reduce probability; not eliminated |
| Global passive adversary | ⚠️ Timing correlation residual risk; cover traffic helps |
| Malicious exit node | ⚠️ Sees destination; ECH hides domain from SNI; HTTPS hides content |
| Pool curator compromised | ⚠️ Individual node verification provides fallback; cross-pool diversity limits damage |
| Compromised client device | ❌ Out of scope |
| Application-layer leakage | ❌ Out of scope |
| Routing policy bypass (LAN) | ⚠️ BGP-X-only mode eliminates; dual-stack: bypass is configurable feature |

---

## 8. BGP and BGP-X Layer Relationship

BGP-X does not participate in BGP. BGP-X uses the public internet — which is routed by BGP — as a transport medium.

```
BGP-X Application Layer     ← BGP-X apps, SDK
BGP-X Overlay Routing       ← path selection, onion encryption, pools
BGP-X Transport (UDP/IP)    ← BGP-X packets as UDP datagrams
         │
         │  BGP-X packets are ordinary UDP/7474 to BGP
         ▼
BGP Routing Layer           ← ISP routing, AS-level decisions
Physical / Link Layer       ← fiber, radio, physical infrastructure
```

For mesh deployments, the transport layer is replaced by radio (LoRa, WiFi mesh, BLE) with zero BGP involvement for intra-mesh traffic.

---

## 9. Deployment Modes

| Mode | BGP Role | Mesh Transport | ISP Required | Use Case |
|---|---|---|---|---|
| Dual-Stack Router | Full transport | Optional additional | Yes | Privacy with selective bypass |
| BGP-X Only Router | Full transport for overlay | Optional | Yes | Maximum enforcement |
| Standalone Device | Full transport | No | Yes | No router access |
| Mesh Node | None | Required | No | Community without ISP |
| Gateway Node | WAN for clearnet exit | Both | At gateway | Bridge mesh to internet |
| Broadcast Amplifier | None | Single transport | No | Range extension |

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

Honest limit: BGP-X replaces ISP dependency and routing decisions; it cannot replace physical infrastructure (fiber, towers, spectrum).

---

## 11. Application Type Model

### Standard Applications (BGP-X Unaware)

- No modification required
- OS routes traffic to bgpx0 based on routing policy
- Transparent protection for selected traffic
- No path control, no service registration, no native addressing

### BGP-X Native Applications (SDK)

- Use SDK to connect to daemon SDK Socket
- Can specify path constraints (pools, hops, ECH, jurisdiction)
- Can register as BGP-X native services (reachable without clearnet)
- Can open isolated paths for sensitive streams
- SDK does NOT implement its own routing — daemon does all work

### Configuration Client (bgpx-cli, GUIs)

- Connects to daemon Control Socket
- Management only — no routing capability
- Status, statistics, policy management, pool management, node database
- Not required for BGP-X to function — daemon runs without it

---

## 12. Hardware Taxonomy

### Unified BGP-X Node Platform

Single PCB design with carrier board and enclosure variants:

**Compute**: MT7981B ARM Cortex-A53 dual-core 1.3GHz, 512MB DDR4, 128MB NAND + 8GB eMMC, TPM 2.0, hardware crypto

**Radios**: WiFi 6 (MT7915, 802.11s + AP), LoRa (SX1262, sub-GHz), optional Bluetooth 5.0

**Enclosure variants**: Indoor Desktop, Outdoor Mast (IP67), Maritime (IP68), Industrial/ATEX, Mobile/Vehicle

**Firmware variants**: bgpx-mesh-node, bgpx-gateway-node, bgpx-amplifier

**Gateway variant**: Upgraded to MT7988A quad-core + 1GB RAM + WAN ports + SFP+ + mPCIe (4G/5G)

**Amplifier variant**: MCU-only (STM32H7), single radio, <800mW, no routing intelligence

### Existing Compatible Hardware

Full BGP-X daemon runs on: GL.iNet routers, Raspberry Pi, Orange Pi, Banana Pi BPi-R3, x86 mini PCs, OpenWrt-compatible consumer routers, industrial outdoor routers, 4G/5G cellular routers.

Meshtastic hardware (ESP32/nRF52840) serves as LoRa radio modems via BGP-X adaptation layer.

---

## 13. Geographic Plausibility Scoring

RTT-based verification of node region claims. No external database required.

- Client measures round-trip time to each node via KEEPALIVE timing
- Compares measured RTT to expected range for claimed region pair
- Discrepancy generates reputation events (GEO_SUSPICIOUS, GEO_IMPLAUSIBLE)
- Not a hard exclusion criterion — reputation signal only
- Mesh transport has separate thresholds (WiFi mesh 1-20ms/hop; LoRa 100ms-5s/hop)
- Satellite nodes exempt from internet-calibrated thresholds

---

## 14. Extension Points

| Extension | Default | Version |
|---|---|---|
| Cover traffic | Disabled | v1 |
| Pluggable transport | Disabled | v1 |
| Stream multiplexing | Enabled | v1 |
| Path quality reporting | Enabled | v1 |
| ECH at exit nodes | Enabled (if ech_capable=true) | v1 |
| Geographic plausibility scoring | Enabled | v1 |
| Pool support | Enabled | v1 |
| Mesh transport | Disabled (enabled in mesh firmware) | v1 |
| Advertisement withdrawal | Supported | v1 |
| Session re-handshake (24hr) | Enabled | v1 |
| Economic incentive layer | Not in scope | — |

All extensions are v1. Nothing is deferred.

---

## 15. Design Decisions Log

| Decision | Rationale |
|---|---|
| UDP as primary transport | Lower overhead, multi-hop latency, custom reliability layer |
| Client-selected paths | Removes network-level adversary from path decisions |
| BLAKE3 for hashing | High performance; suitable for embedded/firmware |
| Ed25519 for signatures | Compact, fast, well-audited, deterministic |
| X25519 for key exchange | State-of-the-art ECDH, widely supported |
| ChaCha20-Poly1305 | Constant-time; no AES-NI required; fast on embedded |
| No separate cover_key | COVER uses session_key; external indistinguishability preserved |
| path_id for return routing | Enables bidirectional sessions without leaking path composition |
| DHT Pools vs. guard nodes | Pools provide trust isolation without creating observable long-term entry relationships |
| Geographic plausibility over external DB | No external dependency; RTT is unforgeable below speed of light |
| Local-first reputation | Local observations authoritative; global is advisory |
| Router-centric architecture | All devices protected without per-device config |
| Unified hardware platform | Single PCB with variants reduces manufacturing complexity |
| No global hop enforcement | Privacy requirements vary; applications set what they need |
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

BGP-X's innovation is unification: combining all of these into one coherent system with hardware targets, pool-based trust domains, and router-level deployment.
