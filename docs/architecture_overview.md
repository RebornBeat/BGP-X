# BGP-X Architecture Overview

**Version**: 0.1.0-draft

---

## 1. What BGP-X Is

BGP-X is a **router-level privacy overlay network and inter-protocol domain router**. It runs on the network router or as a standalone daemon, protecting all connected devices transparently. Applications require no modification.

BGP-X provides:
- **Onion-encrypted routing**: no single node sees both endpoints of a connection
- **Client-selected paths**: the sender constructs the complete path before transmission
- **Cross-domain routing**: paths may span clearnet, overlay, and mesh islands in any combination, any order, unlimited hops
- **Three equal entry points**: clearnet, BGP-X overlay, and mesh islands are first-class citizens
- **Router-centric deployment**: all LAN devices protected without per-device configuration
- **Unified DHT**: A single decentralized discovery layer spanning all routing domains
- **Pool-based trust**: Named pools with curator signatures enable trust segmentation
- **N-hop unlimited**: No protocol-level maximum on path length, domain count, or segment count

---

## 2. Router-Centric Design

BGP-X is designed as **network infrastructure, not a per-application tool**.

The BGP-X daemon (`bgpx-node`) runs on the network router and is the single BGP-X routing stack for all devices on the network. Every other BGP-X component — SDK applications, the configuration client (`bgpx-cli`) — is a client of this daemon.

This means:

- **Standard applications** require zero modification to be protected
- **All devices on the LAN** are automatically protected by the router's BGP-X deployment
- **Application developers** use the SDK to connect to the daemon — not to build their own overlay stack
- **Configuration and management** happens via the Control Socket, accessed by the `bgpx-cli` or web UI

On **standalone devices** (laptop, server), the daemon runs locally — identical code, narrower scope. For **mobile and embedded environments**, the SDK offers an embedded mode where the daemon runs in-process.

---

## 3. The Core Problem

The internet was designed to connect machines, not to protect the identity of the people using those machines.

Every packet sent over the public internet carries a source IP address and a destination IP address. Both are visible to every network device that handles the packet — your router, your ISP, any ISP along the path, and the destination's ISP.

**HTTPS protects the *content* of the packet. It does not protect the metadata: who is talking to whom, when, and how often.** That metadata — often called traffic analysis data — can be as sensitive as the content itself.

Existing solutions address parts of this problem:

- **VPNs** — route all traffic through a single trusted server. The VPN provider sees everything the ISP previously saw. Trust is shifted, not eliminated.
- **Tor** — distributes trust across 3 fixed hops. Strong privacy model, but centralized directory authorities, fixed path length, TCP-only, and application-proxy architecture limit what can be built on top of it.
- **I2P** — strong internal network properties but primarily designed for internal services, not clearnet access. Complex to use for general internet traffic.
- **cjdns / Yggdrasil** — public key addressing with mesh transport; no onion routing; no anonymity.
- **Reticulum** — multi-transport mesh; no onion routing; no clearnet exit model.

BGP-X unifies: multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust + hardware targets — into one coherent router-level system with cross-domain routing.

---

## 4. Three Equal Entry Points

BGP-X treats three network classes as equal first-class citizens. No entry point is privileged or required:

```
CLEARNET                  BGP-X OVERLAY             MESH ISLANDS
(BGP-routed internet)     (onion-encrypted layer)    (radio transport)
        │                         │                         │
        └─────────────────────────┴─────────────────────────┘
                    Any combination, any order, unlimited hops
                         All reach all. None is secondary.
```

A **clearnet client with a BGP-X daemon** (no mesh hardware) can reach a service inside a mesh island via a domain bridge node that handles the radio transmission. A **mesh island client** can reach clearnet via a gateway. A path can traverse clearnet → mesh → clearnet. The protocol enforces no entry point restriction and no domain ordering.

---

## 5. System Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│  Standard apps (transparent) + SDK apps (aware)        │
├─────────────────────────────────────────────────────────┤
│                Routing Policy Engine                    │
│  Per-flow decision: BGP-X overlay vs. standard routing  │
│  Domain-aware rules: cross-domain path specifications   │
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
│  Path construction, onion encryption, forwarding        │
│  Unified DHT (all domains), pools, reputation, geo      │
├─────────────────────────────────────────────────────────┤
│                   Gateway Layer                         │
│  Overlay → Public Internet translation                  │
│  Domain bridge: clearnet ↔ mesh ↔ satellite            │
│  ECH support, DNS resolver, exit policy enforcement    │
├─────────────────────────────────────────────────────────┤
│                   Transport Layer                       │
│  UDP/IP (internet), WiFi 802.11s, LoRa, BLE, Ethernet │
│  Pluggable transport for obfuscation                   │
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

## 6. N-Hop Unlimited

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

## 7. Packet Lifecycle

### Standard App on LAN Device → Clearnet

```
1. App sends packet to OS network stack
2. Routing policy engine matches rule: → BGP-X overlay
3. Packet enters TUN interface (bgpx0)
4. BGP-X daemon reads from TUN
5. Client constructs session keypair (ephemeral X25519)
6. Client selects path: [Entry E1] → [Relay R1] → [Relay R2] → [Exit G1]
7. Client performs handshakes with each hop in order (domain-agnostic)
8. Client generates path_id (8 bytes CSPRNG)
9. Client wraps packet in 4 onion layers, each containing path_id
10. Client sends outermost-encrypted packet to E1

11. E1 decrypts its layer → stores path_id → client_addr; forwards to R1
12. R1 decrypts → stores path_id → E1_addr; forwards to R2
13. R2 decrypts → stores path_id → R1_addr; forwards to G1
14. G1 decrypts final layer → sees destination; applies exit policy; ECH if capable
15. G1 connects to destination; sends request

16. Response arrives at G1
17. G1 encrypts with K_G1; forwards via path_id → R2
18-20. Each relay forwards opaque blob via path_id (no decryption)
21. Client decrypts with K_G1 → response delivered to application
```

### Clearnet Client → Mesh Island Service (Cross-Domain)

```
1. App sends packet; routing policy matches cross-domain rule
2. Client discovers bridge nodes for required domain transition
3. Client selects path spanning clearnet + mesh segments
4. Domain-agnostic handshake with all hops including bridge node
5. Client constructs onion with DOMAIN_BRIDGE hop at bridge position
6. Bridge node receives clearnet UDP, decrypts DOMAIN_BRIDGE layer
7. Bridge node forwards remaining onion via WiFi/LoRa to M1
8. M1, M2 forward mesh hops; service delivers response
9. Return path: service → M2 → M1 → B1 (via mesh, opaque)
   → B1 looks up cross-domain path_id → R1_clearnet_addr → R1 → E1 → client
```

**Clearnet client needs no mesh hardware.** The bridge node handles all radio transmission. The service never sees the client's clearnet IP.

---

## 8. Return Path Architecture

Return traffic uses a different architecture than outbound:

- **Client holds all N session keys** (one per hop).
- **Exit encrypts response** with the session key it shares with the client (`K_exit`).
- **Intermediate relays forward the encrypted blob without decryption** using `path_id → predecessor` mapping.
- **The blob is not re-encrypted at each hop** — it is already encrypted for the client.
- **Two independent sequence number spaces per session** (inbound/outbound) prevent nonce reuse.

This means only the exit node and the client can decrypt the response. Intermediate relays are opaque forwarders for return traffic.

---

## 9. Component Dependency Map

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

## 10. Key Exchange and Session Establishment

BGP-X uses a Noise Protocol-inspired key exchange. **Domain-agnostic** — identical whether session is over UDP/IP, WiFi mesh, LoRa, BLE, or satellite.

**Process:**

1.  Client generates ephemeral X25519 keypair per session.
2.  For each hop, in order (entry first):
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

## 11. Node Discovery — Unified DHT

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

## 12. Trust Model

### Adversary Assumptions

- May control any subset of relay nodes in any routing domain
- May observe all public internet traffic
- May operate malicious entry, exit, or domain bridge nodes
- May observe mesh radio traffic
- Does NOT compromise the client's device

### Privacy Guarantees

With N-hop path and pool/domain diversity enforcement:
- No single relay can link sender to destination (split knowledge)
- Domain bridge nodes see only immediate predecessors in each domain — not full path
- `path_id` enables return routing without leaking path composition across domain boundaries
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic (using `session_key`, externally identical to RELAY) disrupts timing correlation in all domains

---

## 13. Deployment Modes

| Mode | BGP Role | Mesh Transport | ISP Required | Use Case |
|---|---|---|---|---|
| Dual-Stack Router | Full transport | Optional | Yes | Privacy with selective bypass |
| BGP-X Only Router | Full transport for overlay | Optional | Yes | Maximum enforcement |
| Standalone Device | Full transport | No | Yes | No router access |
| Mesh Node | None | Required | No | Community without ISP |
| Gateway/Domain Bridge Node | WAN for clearnet exit | Both | At gateway | Bridge mesh to internet |
| Broadcast Amplifier | None | Single transport | No | Range extension |

All modes except Amplifier support cross-domain routing through domain bridge configuration when applicable.

---

## 14. Application Type Model

### Standard Applications (BGP-X Unaware)

No modification required. Routing policy can specify cross-domain paths transparently.

### BGP-X Native Applications (SDK)

Full `RoutingDomain` and `DomainSegmentConfig` support in `PathConfig`. Can specify cross-domain paths, pool constraints, ECH requirements, mesh island service addressing. Can register as BGP-X native services in any routing domain.

### Configuration Client (bgpx-cli, GUIs)

Connects to daemon Control Socket. Cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing.

---

## 15. Geographic Plausibility Scoring

RTT-based verification of node region claims. No external database required. Domain-specific thresholds:

- **Clearnet nodes**: internet RTT thresholds by region pair
- **WiFi mesh nodes**: 1-20ms per hop expected
- **LoRa mesh nodes**: 100ms-5000ms per hop expected (not flagged for high latency)
- **Domain bridge nodes**: checked independently for each domain they serve
- **Satellite nodes**: exempt from geographic thresholds (return 0.5 neutral)

Not a hard exclusion — reputation signal only. Discrepancy generates `GEO_SUSPICIOUS` or `GEO_IMPLAUSIBLE` reputation events.

---

## 16. Extension Architecture

All extensions are v1. Nothing deferred.

| Extension | Default | Notes |
|---|---|---|
| Cover traffic | Disabled | Enable for timing resistance |
| Pluggable transport | Disabled | Enable for censorship resistance |
| Stream multiplexing | Enabled | Multiple streams per path |
| Path quality reporting | Enabled | Includes `domain_id` field |
| ECH at exit nodes | Enabled (if capable) | Hides domain at exit |
| Geographic plausibility | Enabled | Domain-specific thresholds |
| Pool support | Enabled | Including domain-scoped pools |
| Mesh transport | Enabled in mesh firmware | WiFi, LoRa, BLE |
| Cross-domain routing | Enabled | Requires bridge nodes |
| Domain bridge support | Enabled when configured | 2+ `routing_domains` required |
| Mesh island routing | Enabled in mesh firmware | Island DHT registration |
| Advertisement withdrawal | Supported | Signed `NODE_WITHDRAW` |
| Session re-handshake (24hr) | Enabled | Fresh keys for long sessions |

---

## 17. Comparison to Tor Architecture

| Property | Tor | BGP-X |
|---|---|---|
| **Deployment** | Per-application or per-device | Router-level (all devices) |
| **Path length** | Fixed 3 (circuit) | Unlimited (client-chosen) |
| **Entry points** | Internet only | Clearnet, overlay, mesh |
| **Cross-domain routing** | No | Yes — any combination |
| **Mesh/radio transport** | No | Yes — WiFi, LoRa, BLE |
| **N-hop unlimited** | No (fixed 3) | Yes |
| **Pool-based trust** | No | Yes |
| **DHT** | Distributed directory | Unified Kademlia DHT |
| **Hardware target** | General-purpose | Router + embedded hardware |
| **ECH at exit** | No | Yes |
| **Cover traffic** | Padding only | COVER uses `session_key` |
| **Decentralization** | Directory authorities | Fully decentralized DHT |
| **Transport Layer** | TCP only | UDP native with reliability layer; carries all IP protocols |
| **Network Layer** | SOCKS5 proxy (application must be aware) | TUN interface (all applications, all protocols, transparent) |
| **Multiplexing** | One circuit per TCP stream | Native stream multiplexing over single path |

---

## 18. Implementation Notes

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
