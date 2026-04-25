# BGP-X Architecture Overview

This document provides a structured narrative overview of the BGP-X system architecture, suitable for engineers evaluating the system or preparing to implement it. For the full formal specification, see `ARCHITECTURE.md`.

---

## The Core Problem

The internet was designed to connect machines, not to protect the identity of the people using those machines.

Every packet sent over the public internet carries a source IP address and a destination IP address. Both are visible to every network device that handles the packet — your router, your ISP, any ISP along the path, and the destination's ISP.

HTTPS protects the *content* of the packet. It does not protect the metadata: who is talking to whom, when, and how often. That metadata — often called traffic analysis data — can be as sensitive as the content itself.

Existing solutions address parts of this problem:

- **VPNs** — route all traffic through a single trusted server. The VPN provider sees everything the ISP previously saw. Trust is shifted, not eliminated.
- **Tor** — distributes trust across 3 fixed hops. Strong privacy model, but centralized directory authorities, fixed path length, TCP-only, and application-proxy architecture limit what can be built on top of it.
- **I2P** — strong internal network properties but primarily designed for internal services, not clearnet access. Complex to use for general internet traffic.

BGP-X is designed to solve the complete problem: routing *any* IP traffic — not just browser traffic, not just TCP streams — through a multi-hop overlay network with cryptographically enforced privacy at every layer, without any centralized coordination point.

---

## Layers

BGP-X is organized into five layers. Each layer has a clearly defined responsibility and communicates with adjacent layers through defined interfaces.

### Layer 0: Transport

The public internet — UDP packets routed via BGP between BGP-X nodes. This layer is unchanged. BGP-X does not modify, participate in, or depend on BGP beyond using it as packet transport.

BGP-X uses UDP at this layer because:

- Lower per-packet overhead than TCP
- No kernel-level connection state between BGP-X nodes
- Enables a custom reliability layer tuned to overlay routing requirements
- Better performance characteristics for multi-hop paths with variable latency

### Layer 1: Overlay Routing (Data Plane)

BGP-X nodes exchange onion-encrypted packets. Each packet carries:

- A routing header (encrypted for the next hop only)
- A payload (encrypted for each subsequent hop in onion layers)
- An authentication tag (integrity protection)

At each hop, the node:

1. Authenticates the packet (verifies the auth tag)
2. Decrypts its layer of the onion
3. Extracts the next hop address
4. Forwards the remaining (still-encrypted) packet

No node can decrypt more than one layer. No node sees more than its immediate predecessor and immediate successor.

### Layer 2: Control Plane

The control plane manages how nodes learn about each other and how paths are constructed.

BGP-X uses a **Distributed Hash Table (DHT)** for node discovery. Nodes publish signed advertisements to the DHT. Clients query the DHT to discover available nodes and their properties.

There are no directory servers, no central authorities, and no single point of failure in the control plane.

Path construction is **client-side**. The client selects nodes from the DHT, verifies their signatures, applies path selection constraints (geographic diversity, latency budget, trust thresholds), and constructs the path before sending a single packet.

### Layer 3: Identity and Resolution

BGP-X separates identity from location.

**Node identity** is a long-term Ed25519 keypair. The NodeID is the BLAKE3 hash of the public key. This identity does not change as the node moves between IP addresses or data centers.

**Client identity** is ephemeral by default. A fresh keypair is generated per connection. Long-term client identity is supported for authenticated services.

**Service identity** allows servers and applications to maintain a stable public key identity while changing their underlying infrastructure. Services register reachability records in the DHT pointing to one or more entry nodes.

**Resolution** maps human-readable service names to ServiceIDs. BGP-X does not define a mandatory resolution system — the DHT provides a lowest-common-denominator resolution layer, and operators can layer additional naming systems on top.

### Layer 4: Gateway (BGP Interoperability)

Exit nodes bridge the BGP-X overlay to the public internet. They are the only nodes that touch both worlds.

A gateway node:

- Receives onion-encrypted packets from the overlay
- Decrypts the final onion layer
- Sees the destination address (clearnet IP or domain)
- Sends the traffic to the destination via the public internet
- Receives responses and re-encrypts them for the return path

Gateway nodes operate under a published, signed exit policy. The exit policy specifies:

- What traffic the gateway will forward (protocol, port, destination class)
- Whether any logging is performed (and what)
- Operator identity and contact information
- Jurisdictional information

Clients can filter available gateways by exit policy before path selection. This allows applications to enforce their own exit requirements (e.g., "only use gateways in jurisdiction X with no-log policy").

---

## Packet Lifecycle

### Outbound (client to destination)

1. **Path selection**: Client queries DHT, selects N nodes, verifies signatures
2. **Key exchange**: Client performs X25519 key exchange with each selected node
3. **Onion construction**: Client wraps packet in N encryption layers, one per hop
4. **Transmission**: Client sends outermost-encrypted packet to entry node via UDP
5. **Relay forwarding**: Each relay strips its layer, forwards to next hop
6. **Gateway delivery**: Exit node strips final layer, sends to destination
7. **Response**: Destination sends response to gateway
8. **Return path**: Gateway encrypts response, sends back through path in reverse
9. **Client receipt**: Client decrypts all layers, receives plaintext response

### Inbound (BGP-X native service)

For services hosted natively on BGP-X (not clearnet), no gateway is needed:

1. Client resolves ServiceID via DHT
2. Path terminates at the service's BGP-X entry node
3. Service decrypts final layer using its private key
4. End-to-end encryption is maintained throughout

---

## Key Exchange and Session Establishment

BGP-X uses a **Noise Protocol**-inspired key exchange:

1. Client generates an ephemeral X25519 keypair per session
2. Client fetches each node's static public key from their signed DHT advertisement
3. Client computes a shared secret with each node using X25519 ECDH
4. Session key is derived from the shared secret via HKDF-SHA256
5. Session key is used for ChaCha20-Poly1305 encryption of that hop's layer

Forward secrecy is provided because:

- Client ephemeral keys are destroyed after session establishment
- Compromise of a node's long-term static key does not decrypt past sessions

---

## Node Discovery (DHT)

BGP-X uses a Kademlia-style DHT for node advertisement storage and retrieval.

### What is stored

Signed node advertisement records. Each record is:

- Signed by the node's private key
- Verifiable by any party with the node's public key
- Timestamped and expires after a configurable TTL (default: 24 hours)
- Re-published by the node before expiry

### What is NOT stored

- Client identities
- Session records
- Traffic metadata

### Bootstrap

New nodes join the DHT by connecting to a set of well-known bootstrap nodes. Bootstrap nodes are not trusted authorities — they are simply known entry points into the DHT graph. Any node can serve as a bootstrap node.

A curated list of bootstrap nodes is distributed with the BGP-X software, but operators can configure custom bootstrap nodes.

---

## Trust Model

BGP-X operates under the following trust assumptions:

**Assumed adversary capabilities:**

- May operate any subset of BGP-X relay nodes (but not all)
- May observe all public internet traffic (global passive adversary)
- May attempt active attacks at individual hops (MITM at a relay)
- May correlate timing of traffic entering and exiting the overlay

**Privacy guarantees:**

- If fewer than all path hops are compromised, source and destination remain unlinked
- A global passive adversary can attempt traffic correlation but cannot decrypt content
- A global active adversary who controls entry and exit simultaneously may perform timing correlation — this is a known residual risk of all low-latency anonymity networks

**Not protected against:**

- Compromise of the client device itself
- Application-layer identity leakage (cookies, logins, fingerprinting)
- Long-term traffic correlation by a sufficiently powerful adversary

---

## Comparison to Tor Architecture

### Structural differences

**Directory model:**

Tor uses a set of hard-coded directory authorities — a small number of servers that publish the consensus list of relays. If a majority of directory authorities are compromised or coerced, the anonymity set is degraded.

BGP-X uses a fully decentralized DHT. There are no directory authorities. There is no consensus document. Each node advertises itself, and clients verify advertisements cryptographically.

**Path length:**

Tor uses exactly 3 hops (guard → middle → exit). This is a pragmatic balance between anonymity and latency.

BGP-X uses configurable path lengths. The minimum is 3; the default is 4; there is no maximum. Applications can enforce longer paths for higher-risk use cases without changing the protocol.

**Transport layer:**

Tor uses TCP streams. This inherits all TCP properties: connection state, ordering, head-of-line blocking, OS fingerprinting.

BGP-X uses UDP with a custom reliability layer. This provides: lower overhead, better multi-hop behavior, resistance to TCP fingerprinting, and a foundation for future transport optimizations.

**Network layer:**

Tor is primarily a SOCKS5 proxy. Applications that do not support SOCKS5 require additional configuration (e.g., iptables rules or a TUN interface).

BGP-X operates at the network layer natively. It provides a TUN interface out of the box. Any application — including those that are SOCKS5-unaware — can route through BGP-X transparently.

**Multiplexing:**

Tor creates one circuit per TCP connection. A single web page may require dozens of circuits.

BGP-X multiplexes multiple streams over a single path using a stream ID. A single path handles all connections to all destinations within a session window.

---

## Extension Architecture

BGP-X is designed with explicit extension points to support future capabilities without protocol fragmentation:

| Extension Point | Description | Default |
|---|---|---|
| Path selection algorithm | Scoring function for node selection | Reputation-weighted latency |
| Cover traffic module | Adds synthetic traffic to resist correlation | Disabled |
| Pluggable transport | Obfuscates BGP-X traffic patterns at Layer 0 | Disabled |
| Resolution layer | Naming system for service discovery | DHT |
| Reputation system | Node trust scoring | Uptime + performance |
| Economic layer | Incentive model for node operators | None (out of scope for base protocol) |

Extensions are negotiated at session establishment. A client and relay that both support an extension activate it; a relay that does not support an extension degrades gracefully to the base behavior.

---

## Component Dependency Map

```
Application / SDK
      │
      ▼
BGP-X Client Core
  ├── Path Selector ──────────► DHT (Node Discovery)
  ├── Key Manager
  ├── Onion Builder
  └── Stream Multiplexer
      │
      ▼ (UDP)
BGP-X Relay Network
  ├── Entry Node
  ├── Relay Nodes (1..N)
  └── Exit / Gateway Node
              │
              ▼ (public internet)
         Destination
```

---

## Implementation Notes (for implementors)

The reference implementation of BGP-X is written in Rust. The design prioritizes:

- **Memory safety** — no undefined behavior in the packet processing path
- **Zero-copy where possible** — packet buffers are passed by reference through the relay pipeline
- **Async I/O** — all network I/O is non-blocking via Tokio
- **Explicit error handling** — no panics in the relay path; all errors are handled and logged
- **Constant-time cryptography** — all cryptographic operations use constant-time implementations to resist timing side-channels

See the implementation specification in `/node/node.md` and the cryptographic specification in `/security/crypto_spec.md` for full detail.
