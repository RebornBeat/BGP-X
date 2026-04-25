# BGP-X Architecture Overview

This document provides a structured narrative overview of the BGP-X system architecture, suitable for engineers evaluating the system or preparing to implement it. For the formal specification, see `ARCHITECTURE.md`.

---

## Router-Centric Design

BGP-X is designed as **network infrastructure, not a per-application tool**.

The BGP-X daemon (`bgpx-node`) runs on the network router and is the single BGP-X routing stack for all devices on the network. Every other BGP-X component — SDK applications, the configuration client — is a client of this daemon.

This means:

- Standard applications require zero modification to be protected
- All devices on the LAN are automatically protected by the router's BGP-X deployment
- Applications that want explicit control use the SDK to connect to the daemon
- The configuration client (bgpx-cli) connects to the daemon for management

On standalone devices (laptop, server), the daemon runs locally — identical code, narrower scope.

For mobile and embedded environments, the SDK offers an embedded mode where the daemon runs in-process.

---

## The Core Problem

The internet was designed to connect machines, not to protect the identity of the people using those machines.

Every packet carries a source IP and destination IP, visible to every network device along the path. HTTPS protects content. BGP-X protects the metadata: who is talking to whom, when, and how much.

Existing solutions each solve part of the problem:

- **VPNs**: shift ISP surveillance to VPN provider; single point of trust
- **Tor**: distributes trust across 3 fixed hops; centralized directory authorities; TCP-only; application proxy
- **I2P**: internal network model; limited clearnet access; not mesh-transport aware
- **cjdns/Yggdrasil**: public key addressing with mesh transport; no onion routing; no anonymity
- **Reticulum**: multi-transport mesh; no onion routing; no clearnet exit model

BGP-X unifies: multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust + hardware targets — into one coherent router-level system.

---

## Layers

### Layer 0: Transport

The physical delivery medium. For internet deployments: UDP/IP via BGP-routed infrastructure. For mesh deployments: WiFi 802.11s, LoRa radio, Bluetooth BLE, or Ethernet point-to-point.

BGP-X is transport-agnostic. The onion encryption and routing logic are identical regardless of whether the underlying transport is UDP, LoRa, or WiFi mesh.

### Layer 1: Overlay Routing (Data Plane)

BGP-X nodes exchange onion-encrypted packets. Each packet contains:

- A path_id (8 bytes, random per path) — enables return traffic routing
- A routing layer (encrypted for the next hop only)
- A payload (encrypted for each subsequent hop in onion layers)
- An authentication tag (integrity protection)

Each hop decrypts its layer, extracts the path_id and next-hop address, records `path_id → predecessor` in memory, and forwards the remaining ciphertext.

For return traffic: nodes look up path_id and forward opaque encrypted blobs toward the client. Intermediate relays do NOT decrypt return traffic.

### Layer 2: Control Plane

The control plane manages how nodes discover each other and how paths are constructed.

**Node discovery**: Kademlia-style DHT. Nodes publish signed advertisements. Clients query DHT to discover nodes and their properties. In mesh-only mode, nodes bootstrap via broadcast beacons (no internet bootstrap nodes needed).

**DHT Pools**: Named, signed collections of nodes. Pool discovery uses the same DHT infrastructure. Pools enable multi-segment paths with different trust tiers per segment.

**Path construction**: Client-side. The client selects nodes from the DHT (respecting pool membership if specified), verifies signatures, applies diversity constraints, and constructs the path before sending a single packet.

### Layer 3: Identity and Resolution

**Node identity**: long-term Ed25519 keypair. NodeID = BLAKE3(public_key). Stable across IP changes.

**Client identity**: ephemeral Ed25519 keypair by default. Generated per session, destroyed after handshake. Long-term client identity supported for authenticated services.

**Service identity**: stable public key. Services register reachability in DHT. Infrastructure can change without changing ServiceID.

**Pool curator identity**: Ed25519 keypair separate from node identity. Signs pool advertisements and membership records. Rotatable via dual-signature key rotation record.

### Layer 4: Routing Policy Engine

The policy engine evaluates each packet or flow from LAN devices and decides:

- Route via BGP-X overlay (into bgpx0 TUN)
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
- Supports ECH when destination publishes ECH configuration in DNS HTTPS records
- Sends traffic to the destination via the public internet
- Receives responses and routes them back via path_id

---

## Packet Lifecycle

### Outbound (client to destination)

```
1. Client constructs session keypair (ephemeral)
2. Client selects path respecting pool configuration
3. Client generates path_id (8 bytes, CSPRNG)
4. Client performs handshakes: entry first (direct), then each relay through established sessions
5. Client wraps packet in N onion layers, each containing path_id
6. Client sends outermost-encrypted packet to entry node

7. Entry decrypts its layer → extracts path_id, next hop
   Entry stores: path_id → client_addr
   Entry forwards remaining ciphertext

8. Each relay decrypts its layer → extracts path_id, next hop
   Relay stores: path_id → predecessor_addr
   Relay forwards

9. Exit decrypts final layer → sees destination + application payload
   Exit resolves DNS via DoH, supports ECH if capable
   Exit sends to clearnet destination

10. Response arrives at exit
11. Exit encrypts response using client's session key (K_exit)
12. Exit sends encrypted blob to last relay via path_id routing
13. Each relay forwards opaque blob using path_id → predecessor mapping
14. Client receives blob, decrypts with K_exit, delivers to application
```

### Return Path Architecture

Return traffic uses a different architecture than outbound:

- Client holds all N session keys (one per hop)
- Exit encrypts response with the session key it shares with the client (K_exit)
- Intermediate relays forward the encrypted blob without decryption using path_id → predecessor mapping
- The blob is not re-encrypted at each hop — it's already encrypted for the client
- Two independent sequence number spaces per session (inbound/outbound) prevent nonce reuse

This means only the exit node and the client can decrypt the response. Intermediate relays are opaque forwarders for return traffic.

---

## Key Exchange and Session Establishment

BGP-X uses a Noise Protocol-inspired key exchange:

```
Client generates ephemeral X25519 keypair per session

For each hop, in order (entry first):
1. Client fetches node's static public key from signed DHT advertisement
2. Client sends HANDSHAKE_INIT:
   - Cleartext portion: client_ephemeral_pub (32 bytes) — needed by node to compute dh1
   - Encrypted portion: session_id, timestamp, versions, extensions
3. Node sends HANDSHAKE_RESP encrypted with response key
4. Client sends HANDSHAKE_DONE encrypted with session key

Session key derived via two-stage X25519:
  dh1 = X25519(client_ephemeral_priv, node_static_pub)  — allows encrypting INIT
  dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)  — adds forward secrecy
  session_key = HKDF-SHA256(dh1 || dh2, salt, info)

Zeroization order (mandatory):
  client_ephemeral_priv → after dh2 computed
  node_ephemeral_priv → after session established
  dh1 → after session_key derived
  dh2 → after session_key derived
  init_key → after handshake complete
  resp_key → after handshake complete
  session_key → when session closed
```

Sessions exceeding 24 hours SHOULD trigger re-handshake on the same path, generating fresh ephemeral keys and new session keys.

---

## Node Discovery (DHT)

BGP-X uses a Kademlia-style DHT for node advertisement storage and retrieval.

### What is stored

- Signed node advertisement records (48hr TTL, re-published every 12hr)
- Signed pool advertisement records (7-day TTL, re-published every 3 days)
- Pool curator key rotation records
- BGP-X network: no client identities, no session records, no traffic metadata

### Bootstrap

**Internet mode**: hardcoded bootstrap node list; DNS-based fallback (`_bgpx-bootstrap._udp.bgpx.network`).

**Mesh-only mode**: broadcast beacons on mesh transport; no hardcoded IPs needed; any reachable node serves as bootstrap.

**Gateway cross-domain sync**: gateways synchronize DHT records between mesh and internet DHT, enabling unified global discovery spanning all transport types.

---

## Trust Model

BGP-X operates under these trust assumptions:

- Adversary may control any subset of relay nodes
- Adversary may observe all public internet traffic
- Adversary may operate malicious entry or exit nodes
- Adversary does NOT compromise the client device

Privacy guarantees with proper path diversity:
- No single relay can link sender to destination (split knowledge)
- path_id does not reveal path composition to intermediate relays
- A global passive adversary can attempt timing correlation — primary residual risk
- Cover traffic (using session_key, externally identical to RELAY) raises correlation attack cost

---

## Comparison to Tor Architecture

### Decentralization

Tor directory authorities are known, fixed targets. BGP-X uses a fully decentralized DHT — no consensus document, no directory authority.

### Path Length

Tor: fixed 3 hops. BGP-X: configurable, default 4, no global enforcement maximum. Applications choose what they need.

### Transport Layer

Tor: TCP only (cannot natively carry UDP traffic). BGP-X: UDP native with reliability layer; carries all IP protocols.

### Network Layer

Tor: SOCKS5 proxy (application must be SOCKS5-aware). BGP-X: TUN interface (all applications, all protocols, transparent).

### Multiplexing

Tor: one circuit per TCP stream. BGP-X: native stream multiplexing over single path.

### Pool-Based Trust

Tor: no equivalent. BGP-X: named pools with curator signatures, multi-segment paths, double-exit architecture.

### ECH

Tor: no exit-level ECH. BGP-X: exit nodes support ECH when destination publishes ECH config.

### Mesh Transport

Tor: no mesh/radio transport. BGP-X: WiFi mesh, LoRa, Bluetooth, Ethernet — operates without ISP.

---

## Component Dependency Map

```
Standard App / SDK App
      │
      ├── (transparent) → TUN Interface (bgpx0)
      │                         │
      └── (SDK) → SDK Socket ──►┤
                                │
                        BGP-X Daemon
                                │
          ┌─────────────────────┼──────────────────────┐
          │                     │                      │
          ▼                     ▼                      ▼
    Path Manager         Pool Manager          Reputation System
    (DHT queries,        (pool discovery,      (local primary,
     node selection,      membership            global advisory,
     onion build)         verification)         geo plausibility)
          │                     │
          └─────────────────────┘
                         │
                         ▼ (UDP / mesh transport / pluggable transport)
                  BGP-X Relay Network
                  (Entry → Relay(s) → Exit)
                         │
                         ▼ (public internet or mesh)
                    Destination
```

---

## Extension Architecture

All extensions are v1 — nothing is deferred.

| Extension | Description | Default |
|---|---|---|
| Cover traffic | COVER uses session_key; same size classes; externally indistinguishable | Disabled |
| Pluggable transport | Obfuscates BGP-X traffic at transport layer | Disabled |
| Stream multiplexing | N streams per path | Enabled |
| Path quality reporting | Per-hop latency bucket + congestion flag, encrypted for client | Enabled |
| ECH | Exit nodes support ECH for TLS destinations | Enabled (if ech_capable) |
| Geographic plausibility | RTT-based node region verification as reputation signal | Enabled |
| Pool support | Multi-segment paths, double-exit, trust tiers | Enabled |
| Mesh transport | WiFi mesh, LoRa, BLE, Ethernet P2P | Enabled in mesh firmware |
| Advertisement withdrawal | Signed NODE_WITHDRAW message | Supported |
| Session re-handshake | Fresh keys at 24hr for long-lived sessions | Enabled |

---

## Implementation Notes

The reference implementation is in Rust. Design priorities:

- **Memory safety**: no undefined behavior in packet processing path
- **Zero-copy where possible**: packet buffers passed by reference through relay pipeline
- **Async I/O**: non-blocking via Tokio
- **Explicit error handling**: no panics in relay path
- **Constant-time cryptography**: all crypto ops use constant-time implementations
- **Zeroization**: all key material cleared from memory in mandatory order

See `/node/node.md`, `/security/crypto_spec.md`, and `/node/api.md` for implementation detail.
