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

---

## 2. The Core Problem

The internet was designed to connect machines, not to protect the identity of the people using those machines.

Every packet carries a source IP and destination IP, visible to every network device along the path. HTTPS protects content. BGP-X protects the metadata: who is talking to whom, when, and how much.

Existing solutions each solve part of the problem:

- **VPNs**: shift ISP surveillance to VPN provider; single point of trust
- **Tor**: distributes trust across 3 fixed hops; centralized directory authorities; TCP-only; application proxy; internet-only
- **I2P**: internal network model; limited clearnet access; not mesh-transport aware
- **cjdns/Yggdrasil**: public key addressing with mesh transport; no onion routing; no anonymity
- **Reticulum**: multi-transport mesh; no onion routing; no clearnet exit model

BGP-X unifies: multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust + hardware targets — into one coherent router-level system with cross-domain routing.

---

## 3. Three Equal Entry Points

```
CLEARNET                  BGP-X OVERLAY             MESH ISLANDS
(BGP-routed internet)     (onion-encrypted layer)    (radio transport)
        │                         │                         │
        └─────────────────────────┴─────────────────────────┘
                    Any combination, any order, unlimited hops
                         All reach all. None is secondary.
```

A clearnet client with a BGP-X daemon (no mesh hardware) can reach a service inside a mesh island via a domain bridge node that handles the radio transmission. A mesh island client can reach clearnet via a gateway. A path can traverse clearnet → mesh → clearnet. The protocol enforces no entry point restriction and no domain ordering.

---

## 4. System Layers

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

---

## 5. Router-Centric Design

BGP-X is designed as **network infrastructure, not a per-application tool**.

The BGP-X daemon (`bgpx-node`) runs on the network router and is the single BGP-X routing stack for all devices on the network. Every other BGP-X component — SDK applications, the configuration client — is a client of this daemon.

This means:

- Standard applications require zero modification to be protected
- All devices on the LAN are automatically protected by the router's BGP-X deployment
- Application developers use the SDK to connect to the daemon — not to build their own overlay stack
- The configuration client (bgpx-cli) connects to the daemon for management

On standalone devices (laptop, server), the daemon runs locally — identical code, narrower scope.

For mobile and embedded environments, the SDK offers an embedded mode where the daemon runs in-process.

---

## 6. N-Hop Unlimited

BGP-X imposes no protocol maximum on path length. The common header has no hop counter. No mechanism drops a packet after N hops. The only minimum is 3 hops.

Applications choose path length based on their threat model and latency tolerance. A 20-hop path across 4 routing domains is as valid as a 3-hop single-domain path.

Default configurations (not enforced limits):

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
2. Client selects cross-domain path:
   [E1: clearnet relay] → [R1: clearnet relay] → [B1: domain bridge]
   → [M1: mesh relay] → [M2: mesh relay] → [Service]
3. Domain-agnostic handshakes with all hops:
   - E1, R1, B1: via clearnet UDP
   - M1, M2: via B1's established session (B1 forwards via mesh radio)
4. Constructs 5-layer onion with DOMAIN_BRIDGE hop at B1 position
5. B1 decrypts DOMAIN_BRIDGE layer → stores cross-domain path_id table
   → forwards remaining onion via WiFi/LoRa to M1
6. M1, M2 forward mesh hops; service delivers response
7. Return path: service → M2 → M1 → B1 (via mesh, opaque)
   → B1 looks up cross-domain path_id → R1_clearnet_addr → R1 → E1 → client
```

Clearnet client needs no mesh hardware. B1 handles all radio. The service never sees the client's clearnet IP.

---

## 8. Component Dependency Map

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
 cross-domain path         verification,          unified DHT sync,
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

## 9. Key Exchange and Session Establishment

BGP-X uses a Noise Protocol-inspired key exchange. **Domain-agnostic** — identical whether session is over UDP/IP, WiFi mesh, LoRa, BLE, or satellite.

```
Client generates ephemeral X25519 keypair per session

For each hop, in order (entry first):
1. Client fetches node's static public key from signed DHT advertisement
2. Client sends HANDSHAKE_INIT:
   - Cleartext: client_ephemeral_pub (32 bytes) — node needs this to compute dh1
   - Encrypted: session_id, timestamp, versions, extensions
3. Node sends HANDSHAKE_RESP encrypted with response key (requires both static+ephemeral)
4. Client sends HANDSHAKE_DONE with confirmation hash and path_id

Session key derived:
  dh1 = X25519(client_ephemeral_priv, node_static_pub)
  dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
  session_key = HKDF-SHA256(dh1 || dh2, salt, info)

Zeroization order (mandatory):
  client_ephemeral_priv → after dh2 computed
  dh1, dh2 → after session_key derived
  init_key, resp_key → after handshake complete
  node_ephemeral_priv → after session established (node side)
  session_key → when session closed

No separate cover_key. COVER uses session_key. Applies in all routing domains.
```

For cross-domain paths: mesh relay handshakes are delivered via the domain bridge node's established session. The bridge node relays handshake packets into the mesh via radio. The mesh relay sees a standard session establishment — it does not know the requesting client is on clearnet.

Sessions exceeding 24 hours SHOULD trigger re-handshake on the same path.

---

## 10. Node Discovery — Unified DHT

BGP-X operates **one unified Kademlia-style DHT** spanning all routing domains. All nodes — clearnet, mesh island, satellite — participate in the same key space.

### What is stored

- Signed node advertisement records (48hr TTL)
- Signed pool advertisement records (7-day TTL)
- Domain bridge records (24hr TTL)
- Mesh island records (24hr TTL)
- Pool curator key rotation records
- No client identities, no session records, no traffic metadata

### Bootstrap

**Internet mode**: hardcoded bootstrap node list; DNS fallback via DoH.

**Mesh-only mode**: broadcast beacons (MESH_BEACON); no hardcoded IPs; any reachable mesh node is a bootstrap peer.

**Unified DHT for mesh**: domain bridge nodes (gateway nodes) store and serve records for mesh-only nodes in the unified internet DHT. Mesh-only nodes access the unified DHT via their bridge nodes' caches. There is no separate mesh DHT and no synchronization between separate DHTs.

---

## 11. Trust Model

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
- path_id enables return routing without leaking path composition across domain boundaries
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic (session_key, externally identical to RELAY) disrupts timing correlation in all domains

---

## 12. Deployment Modes

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

## 13. Application Type Model

### Standard Applications (BGP-X Unaware)

No modification required. Routing policy can specify cross-domain paths transparently.

### BGP-X Native Applications (SDK)

Full `RoutingDomain` and `DomainSegmentConfig` support in `PathConfig`. Can specify cross-domain paths, pool constraints, ECH requirements, mesh island service addressing. Can register as BGP-X native services in any routing domain.

### Configuration Client (bgpx-cli, GUIs)

Connects to daemon Control Socket. Cross-domain management: domain listing, bridge node discovery, mesh island status, cross-domain path testing.

---

## 14. Geographic Plausibility Scoring

RTT-based verification of node region claims. No external database required. Domain-specific thresholds:

- **Clearnet nodes**: internet RTT thresholds by region pair
- **WiFi mesh nodes**: 1-20ms per hop expected
- **LoRa mesh nodes**: 100ms-5000ms per hop expected (not flagged for high latency)
- **Domain bridge nodes**: checked independently for each domain they serve
- **Satellite nodes**: exempt from geographic thresholds (return 0.5 neutral)

Not a hard exclusion — reputation signal only. Discrepancy generates GEO_SUSPICIOUS or GEO_IMPLAUSIBLE reputation events.

---

## 15. Extension Points

All extensions are v1. Nothing deferred.

| Extension | Default | Notes |
|---|---|---|
| Cover traffic | Disabled | Enable for timing resistance |
| Pluggable transport | Disabled | Enable for censorship resistance |
| Stream multiplexing | Enabled | Multiple streams per path |
| Path quality reporting | Enabled | Includes domain_id field |
| ECH at exit nodes | Enabled (if capable) | Hides domain at exit |
| Geographic plausibility | Enabled | Domain-specific thresholds |
| Pool support | Enabled | Including domain-scoped pools |
| Mesh transport | Enabled in mesh firmware | WiFi, LoRa, BLE |
| Cross-domain routing | Enabled | Requires bridge nodes |
| Domain bridge support | Enabled when configured | 2+ routing_domains required |
| Mesh island routing | Enabled in mesh firmware | Island DHT registration |
| Advertisement withdrawal | Supported | Signed NODE_WITHDRAW |
| Session re-handshake (24hr) | Enabled | Fresh keys for long sessions |

---

## 16. Implementation Notes

The reference implementation is in Rust. Design priorities:

- **Memory safety**: no undefined behavior in packet processing path; no raw pointer arithmetic in parsing
- **Zero-copy where possible**: packet buffers passed by reference through relay pipeline
- **Async I/O**: non-blocking via Tokio; receiver threads use `SO_REUSEPORT` for parallel receive
- **Explicit error handling**: no panics in relay path; all errors handled explicitly
- **Constant-time cryptography**: all crypto operations use constant-time implementations from audited libraries; no timing side channels
- **Zeroization**: all key material cleared from memory in mandatory order using `zeroize` crate
- **Domain-agnostic crypto**: identical ChaCha20-Poly1305, X25519, Ed25519, BLAKE3 in all routing domains; no domain-specific key derivation
- **Unified DHT participation**: one DHT instance; no per-domain DHT splitting
- **Cross-domain path table**: sharded concurrent hash map for path_id → cross-domain routing, separate from single-domain path table

See `/node/node.md`, `/node/api.md`, `/security/crypto_spec.md`, and `/data-plane/forwarding.md` for implementation detail.

---

## 17. Comparison to Tor Architecture

### Decentralization

Tor directory authorities are known, fixed targets. BGP-X uses a fully decentralized unified DHT — no consensus document, no directory authority.

### Path Length

Tor: fixed 3 hops. BGP-X: configurable, default 4, no protocol maximum. No global enforcement maximum. Applications choose.

### Transport Layer

Tor: TCP only. BGP-X: UDP native with reliability layer; carries all IP protocols; also WiFi mesh, LoRa, BLE for mesh domains.

### Network Layer

Tor: SOCKS5 proxy (application must be aware). BGP-X: TUN interface (all applications, all protocols, transparent).

### Cross-Domain

Tor: internet only. BGP-X: clearnet, overlay, and mesh islands as equals; any-to-any routing including clearnet clients reaching mesh island services without special hardware.

### Pool-Based Trust

Tor: no equivalent. BGP-X: named pools with curator signatures, multi-segment paths, double-exit architecture, domain-scoped pools.

### ECH

Tor: no exit-level ECH. BGP-X: exit nodes support ECH when destination publishes ECH config.

### Hardware Ecosystem

Tor: no hardware. BGP-X: unified hardware platform (Node, Gateway, Amplifier variants).
