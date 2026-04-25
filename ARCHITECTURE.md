# BGP-X System Architecture

This document describes the full system architecture of BGP-X: a privacy-preserving, identity-routed overlay routing network.

---

## 1. Design Philosophy

BGP-X is built on five non-negotiable principles:

### 1.1 No single party should see both endpoints of a connection

This is the foundational property. No node in the BGP-X network, no ISP, and no gateway should be able to link a source identity to a destination. This is enforced at the protocol level, not by policy.

### 1.2 Identity is cryptographic, not location-based

On the public internet, IP addresses serve as both network location and de facto identity. BGP-X separates these:

- **Location** is handled by underlying IP/BGP (opaque to BGP-X clients)
- **Identity** is a public key, self-certifying, portable, and optionally ephemeral

### 1.3 Routing decisions belong to the sender

BGP delegates path selection to the network (ISPs). BGP-X delegates path selection to the client. The sender chooses which relays to use, in what order, and under what trust constraints.

### 1.4 The existing internet is transport, not adversary

BGP-X does not attempt to fix or circumvent BGP. It uses the public internet as a raw packet transport layer and builds all privacy and routing logic above it.

### 1.5 Layered security, not security by obscurity

Every privacy property of BGP-X must be cryptographically enforceable, not dependent on the honesty of any individual operator. Honest operators improve performance; dishonest operators should fail to compromise privacy.

---

## 2. System Layers

BGP-X defines five logical layers. Each is independent and composable.

┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│     Any IP application, service, or SDK consumer       │
├─────────────────────────────────────────────────────────┤
│                   Identity Layer                        │
│     Public key identities, resolution, rotation        │
├─────────────────────────────────────────────────────────┤
│                   Overlay Routing Layer                 │
│     Path construction, onion encryption, forwarding    │
├─────────────────────────────────────────────────────────┤
│                   Gateway Layer                         │
│     Overlay → Public Internet translation              │
├─────────────────────────────────────────────────────────┤
│                   Transport Layer                       │
│     UDP/IP, encrypted tunnels, existing BGP routing    │
└─────────────────────────────────────────────────────────┘

---

## 3. Core Components

### 3.1 Node

A BGP-X **node** is any participant in the overlay network. Nodes may serve one or more roles:

- **Relay node** — forwards encrypted packets, sees only previous and next hop
- **Entry node** — accepts client connections into the overlay; knows client IP, not destination
- **Exit node / Gateway** — translates overlay traffic to public internet traffic; knows destination, not client IP
- **Discovery node** — participates in DHT-based node advertisement

Nodes are identified by their `NodeID`:
NodeID = BLAKE3(PublicKey)
### 3.2 Client

A **client** is the initiating party. Clients:

- Generate a session identity (keypair)
- Query the resolution layer for destination
- Select a path (entry + relays + exit)
- Construct and transmit onion-encrypted packets
- Manage session state locally

Clients do not advertise themselves to the network. They are purely consumers of overlay routing services.

### 3.3 Identity Resolution Layer

Analogous to DNS, but:

- Distributed (DHT-based, no root authority)
- Records are signed by the identity owner
- Resolution queries are privacy-preserving (routed through overlay, not sent in plaintext)

Resolution maps:
ServiceName → ServiceID (PublicKey)
ServiceID   → Reachable Entry Nodes + Path Hints
### 3.4 Path

A path is a client-defined ordered sequence of overlay nodes:
[Entry] → [Relay_1] → ... → [Relay_N] → [Exit]
Minimum path length: 3 hops (entry + 1 relay + exit)
Default path length: 4 hops
Maximum path length: Uncapped (latency trade-off)

Path selection criteria:
- Geographic diversity (no two hops in same AS or country)
- Latency budget
- Node reputation score
- Bandwidth availability

### 3.5 Gateway

The **gateway** is the component that bridges the BGP-X overlay to the public internet. It is the exit point for any traffic destined for a non-BGP-X endpoint (i.e., any normal website or service).

Gateway properties:
- Sees only the destination (not the originating client)
- Operates under a published exit policy
- Must be signed and registered in the overlay
- Maintains no per-connection logs (policy-enforced, auditable)

---

## 4. Traffic Flow (Full End-to-End)

### 4.1 Client visiting a clearnet destination (e.g., example.com)

1. Client constructs session keypair
2. Client resolves destination via BGP-X resolution layer
3. Client selects path: [Entry E1] → [Relay R1] → [Relay R2] → [Exit G1]
4. Client wraps packet in 4 onion layers (one per hop)
5. Client sends encrypted packet to E1
6. E1 decrypts outer layer → learns next hop is R1 → forwards
7. R1 decrypts its layer → learns next hop is R2 → forwards
8. R2 decrypts its layer → learns next hop is G1 (exit) → forwards
9. G1 decrypts final layer → sees destination (example.com) → sends via public internet
10. Response arrives at G1 → re-encrypted for return path → forwarded back
11. Client receives response

### 4.2 Client visiting a BGP-X native service

1. Client resolves ServiceID via DHT
2. Destination is a BGP-X node, not a clearnet address
3. Path terminates at destination node (no gateway needed)
4. End-to-end onion encryption applies throughout

---

## 5. Identity Model

### 5.1 Node Identity

Every node holds a long-term keypair:
(node_private_key, node_public_key)
The `NodeID` is derived deterministically:
NodeID = BLAKE3(node_public_key)

Node identity is used for:
- Signing advertisements
- Establishing encrypted sessions with peers
- Reputation tracking

### 5.2 Client Identity

Clients generate session identities:
(session_private_key, session_public_key)

Session identities are ephemeral by default — generated fresh per connection. Long-term client identities are supported for authenticated services but not required.

### 5.3 Service Identity

Services (servers, APIs, applications) register a stable public key identity:
ServiceID = PublicKey

Services advertise reachability through the DHT and can rotate their underlying node infrastructure without changing their ServiceID.

---

## 6. Trust Model

### 6.1 What BGP-X assumes

- The adversary may control any subset of relay nodes
- The adversary may observe all public internet traffic (global passive adversary)
- The adversary may operate malicious entry or exit nodes
- The adversary does NOT compromise the client's device

### 6.2 Privacy guarantees under this model

With a path of N hops:

- No single relay can link sender to destination (split knowledge)
- A global adversary can attempt traffic correlation — this is the primary residual risk
- Longer paths + cover traffic + batching mitigate correlation attacks

### 6.3 What BGP-X does NOT guarantee

- Protection against endpoint compromise (client or server device)
- Protection against timing correlation by a global adversary watching both ends simultaneously
- Anonymity if the client's application layer leaks identity (cookies, browser fingerprint, etc.)

---

## 7. Threat Model Summary

| Adversary | BGP-X Protection |
|---|---|
| ISP (passive) | ✅ Sees only encrypted traffic to first relay |
| ISP (active MITM) | ✅ Cannot break onion encryption |
| Single malicious relay | ✅ Cannot link source to destination |
| Multiple colluding relays (minority) | ✅ Cannot link with path diversity |
| Multiple colluding relays (majority) | ⚠️ Partial — depends on path selection |
| Global passive adversary | ⚠️ Correlation risk — mitigated by cover traffic |
| Malicious exit node | ⚠️ Sees destination + plaintext if no inner HTTPS |
| Compromised client device | ❌ Out of scope |

---

## 8. Comparison to Existing Systems

### 8.1 vs. VPN

A VPN routes all traffic through a single trusted provider. The VPN provider sees source, destination, and content (unless HTTPS). BGP-X distributes trust across N hops with no single party able to see both ends.

### 8.2 vs. Tor

| Dimension | Tor | BGP-X |
|---|---|---|
| Directory | Centralized authorities | Decentralized DHT |
| Path length | Fixed 3 hops | Configurable N hops |
| Transport | TCP | UDP + reliability layer |
| Network layer | App-layer proxy | Full IP overlay |
| Firmware support | No | Yes |
| Multiplexing | No (one circuit = one stream) | Yes (N streams per path) |
| Congestion control | Inherited from TCP | Native |
| Cover traffic | Off by default | Pluggable module |

### 8.3 vs. I2P (Invisible Internet Project)

I2P uses garlic routing (multiple messages bundled) and is primarily designed for internal services. BGP-X is designed for full internet access with gateway exit nodes, and integrates at the network layer rather than the application layer.

### 8.4 vs. SCION

SCION (Scalability, Control, and Isolation On Next-generation networks) provides path-aware, source-selected routing at the infrastructure level, requiring ISP adoption. BGP-X achieves similar path control as an overlay without any infrastructure changes.

---

## 9. Non-Goals

The following are explicitly out of scope for BGP-X:

- Replacing or modifying BGP itself
- Providing a DNS replacement (resolution layer is a separate concern and can be swapped)
- Guaranteeing anonymity against an adversary who compromises the client device
- Providing content moderation or abuse reporting infrastructure
- Being a cryptocurrency or incentive layer (economic models are a separate layer)

---

## 10. Extension Points

BGP-X is designed for composability. The following are explicitly pluggable:

- **Resolution layer** — DHT is default; can be replaced with alternative naming systems
- **Path selection algorithm** — default is reputation-weighted latency-optimized; fully replaceable
- **Cover traffic module** — off by default; pluggable at the data plane
- **Reputation system** — default uses signed uptime + performance data; can be extended
- **Crypto primitives** — algorithm suite is versioned; future suites can be introduced without breaking the protocol

---

## 11. Design Decisions Log

| Decision | Rationale |
|---|---|
| UDP as primary transport | Lower overhead, works better with multi-hop latency, enables custom reliability layer |
| Client-selected paths | Removes network-level adversary from path decisions; increases sender control |
| BLAKE3 for hashing | High performance, modern design, suitable for embedded/firmware use |
| Ed25519 for signatures | Compact, fast, well-audited |
| X25519 for key exchange | State-of-the-art ECDH, widely supported |
| ChaCha20-Poly1305 for symmetric encryption | Hardware-accelerated on most platforms; superior to AES on devices without AES-NI |
| No built-in incentive layer | Keeps protocol simple; economic layer is a separate concern |
| Signed exit policies | Makes gateway behavior auditable and enforceable without trusting operators |

---

## 12. Version History

See [CHANGELOG.md](./CHANGELOG.md).
