# BGP-X

**BGP-X** is a privacy-preserving, identity-routed overlay routing network built on top of the existing internet infrastructure.

BGP-X does not replace the Border Gateway Protocol. It transcends it — using the public internet as a dumb transport substrate while implementing a parallel, privacy-first routing plane that is identity-based, multi-hop by default, and cryptographically enforced at every layer.

---

## Why BGP-X Exists

The internet's routing layer (BGP) was built for reachability, not privacy. Every packet you send leaks:

- **Your real IP** (your network identity)
- **Your destination IP** (who you're talking to)
- **Traffic patterns** (when, how much, what timing)

HTTPS solves *content* privacy. VPNs shift trust to a single provider. Tor provides unlinkability but with significant latency, limited bandwidth, and a centralized directory authority model.

BGP-X is built to solve a different problem:

> **You should be able to communicate across the internet such that no single party — not your ISP, not your government, not the server you're talking to — can link you to your destination, without sacrificing the ability to build production-grade networked systems.**

---

## What BGP-X Is

BGP-X is a **multi-hop, onion-routed overlay network** with the following design properties:

- **Identity-based addressing** — nodes and services are identified by cryptographic public keys, not IP addresses
- **Client-selected paths** — the sender (not the network) chooses the relay path
- **Layered encryption** — onion-encrypted packets; each relay decrypts only its own layer
- **No central directory** — distributed node discovery via DHT and signed peer advertisements
- **Gateway interoperability** — controlled, auditable exit nodes bridge the overlay to the existing internet
- **Forward secrecy** — ephemeral session keys ensure past sessions cannot be decrypted after the fact
- **Optional identity rotation** — clients can rotate keys per session, per connection, or per application policy

---

## How BGP-X Differs from Tor

| Property | Tor | BGP-X |
|---|---|---|
| Architecture | Fixed 3-hop circuit | Variable N-hop path, client-configurable |
| Directory model | Centralized directory authorities | Decentralized DHT + signed advertisements |
| Transport | TCP streams | UDP-native with reliability layer |
| Identity model | Single anonymous identity per circuit | Cryptographic identity; rotatable per session |
| Latency profile | Optimized for low latency | Configurable: low-latency or high-anonymity modes |
| Congestion control | None (TCP passthrough) | Native BGP-X congestion and backpressure model |
| Multiplexing | Single stream per circuit | Native stream multiplexing over a single path |
| Exit model | Any relay can be an exit | Designated, audited, signed exit nodes |
| Interoperability | Browser-focused | Full network layer — supports any IP traffic |
| BGP awareness | None | Designed to interact cleanly with public internet at gateways |
| Cover traffic | Optional (not default) | Pluggable cover traffic module |
| Firmware support | No | Yes — router firmware targets (OpenWrt, etc.) |
| Path trust model | Assumes 1 in 3 hops is honest | Configurable trust thresholds; reputation-weighted path selection |

---

## How BGP-X Surpasses Tor

### 1. No central directory authority

Tor's directory authorities are a known attack surface. BGP-X uses a fully decentralized DHT-based node discovery system with signed, verifiable node advertisements. There is no single server that, if compromised, degrades the anonymity of the entire network.

### 2. Protocol-level, not application-level

Tor is primarily a browser/SOCKS proxy. BGP-X is a full network layer. Any application, daemon, or system service can route through BGP-X without modification.

### 3. Configurable path length and trust model

Tor hardcodes 3-hop circuits as a balance between anonymity and latency. BGP-X exposes path length, hop selection policy, and relay trust thresholds as first-class configuration. High-risk use cases can extend to 5+ hops with geographic diversity enforcement.

### 4. Native multiplexing

Tor creates one circuit per destination. BGP-X multiplexes multiple streams over a single path, dramatically improving efficiency for multi-connection workloads (APIs, browsers, services).

### 5. Router firmware integration

BGP-X is designed to run on commodity routers. This means all traffic from all devices on a network segment routes through BGP-X without per-device configuration — the same model as a VPN gateway, but with genuine unlinkability.

### 6. Gateway architecture for full internet reach

BGP-X exits to the public internet through designated gateway nodes that are explicitly audited and signed. This makes exit policy transparent, accountable, and configurable — unlike Tor exit nodes which are operated by volunteers with no formal accountability structure.

---

## What BGP-X Is Not

- ❌ A replacement for BGP (the Border Gateway Protocol)
- ❌ A VPN (VPNs shift trust to a single provider; BGP-X distributes it)
- ❌ A Tor fork (fundamentally different architecture, protocol, and threat model)
- ❌ An anonymizing proxy (operates at the network layer, not application layer)
- ❌ A silver bullet (see Threat Model for limitations)

---

## Architecture in One Diagram

[Application]
      ↓
[BGP-X Client SDK]
      ↓
[BGP-X Overlay Network]
  Entry → Relay → Relay → Exit
      ↓
[Gateway: BGP-X → Public IP Internet]
      ↓
[Destination Server]



The public internet's BGP routing is used only as a packet transport. All routing logic, identity, and privacy is handled within the BGP-X overlay.

---

## Repository Structure

/bgp-x
├── README.md               — This file
├── ARCHITECTURE.md         — Full system architecture
├── CHANGELOG.md            — Version history
├── CONTRIBUTING.md         — Contribution guidelines
├── CODE_OF_CONDUCT.md      — Community standards
├── LICENSE                 — Apache 2.0
├── SECURITY.md             — Vulnerability reporting
├── /docs                   — User-facing documentation
├── /protocol               — Protocol specification
├── /control-plane          — Routing logic and node management
├── /data-plane             — Packet forwarding and encryption
├── /node                   — Node daemon specification
├── /gateway                — BGP interoperability layer
├── /firmware               — Router firmware targets
├── /client                 — Client application specification
├── /sdk                    — Developer SDK specification
├── /security               — Security model and auditing
├── /simulation             — Network and attack simulation
├── /deployment             — Deployment and operations guides
├── /production             — SOPs, certification, release
└── /legal                  — Liability and privacy documentation


---

## Status

BGP-X is in **pre-implementation specification phase**.

All documentation in this repository represents a complete system design intended to guide implementation. No production code has been written yet.

Phase plan:

- [x] System architecture
- [x] Protocol specification
- [x] Security model
- [ ] Reference implementation (Rust)
- [ ] Testnet
- [ ] Mainnet

---

## License

Apache License 2.0. See [LICENSE](./LICENSE).

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## Security

To report a vulnerability, see [SECURITY.md](./SECURITY.md).
