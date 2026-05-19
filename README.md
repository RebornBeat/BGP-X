# BGP-X

**BGP-X** is a router-level privacy overlay network and inter-protocol domain router. It runs on your network router and protects all connected devices transparently — or on a standalone device, or embedded directly in an application.

BGP-X routes traffic across routing domains: clearnet (BGP-routed internet, including all satellite internet services), BGP-X overlay, and mesh islands (WiFi 802.11s, LoRa, Bluetooth). Any domain can reach any other domain through domain bridge nodes. There is no required domain ordering. Paths may traverse any combination of domains, any number of times, with no protocol-level maximum on total hops.

BGP-X does not replace the Border Gateway Protocol. It uses the public internet as one of several transport substrates while implementing a parallel, privacy-first routing plane that is identity-based, multi-hop by default, cryptographically enforced at every layer, and capable of routing between multiple network domains without restriction.

---

## Why BGP-X Exists

The internet's routing layer (BGP) was built for reachability, not privacy. Every packet you send leaks:

- **Your real IP** (your network identity)
- **Your destination IP** (who you're talking to)
- **Traffic patterns** (when, how much, what timing)

HTTPS solves content privacy. VPNs shift trust to a single provider. Tor provides unlinkability but with centralized directory authorities, fixed path length, TCP-only transport, and application-proxy architecture that limits what can be built on top of it.

BGP-X is built to solve a different problem:

> **You should be able to communicate across the internet such that no single party — not your ISP, not your government, not the server you're talking to — can link you to your destination, without sacrificing the ability to build production-grade networked systems.**

---

## What BGP-X Is

BGP-X is a **router-level privacy overlay network and inter-protocol domain router** with the following design properties:

- **Router-centric**: runs on your network router and protects all connected devices — no per-device configuration required
- **Identity-based addressing**: nodes and services identified by cryptographic public keys, not IP addresses
- **Three equal entry points**: clearnet internet, BGP-X overlay, and mesh islands — any entry point reaches any destination
- **Cross-domain routing**: route through any combination of clearnet, overlay, and mesh domains in any order, any number of times
- **Client-selected paths**: the sender constructs the complete relay path before transmitting
- **N-hop unlimited**: no protocol-level maximum on path length, domain count, or segment count
- **Layered onion encryption**: each relay decrypts only its own layer; no relay sees both source and destination
- **No central directory**: decentralized DHT-based node discovery with signed peer advertisements — one unified DHT spanning all routing domains
- **DHT Pools**: configurable trust segments — public, curated, private, semi-private — enabling double-exit architecture and jurisdiction-specific routing
- **Domain bridge nodes**: nodes with endpoints in multiple routing domains enable any-to-any cross-domain connectivity
- **Mesh island routing**: named mesh island networks are first-class routing domains — addressable, discoverable, and reachable from clearnet without any special client hardware
- **Gateway interoperability**: designated exit nodes bridge the overlay to the existing internet
- **Forward secrecy**: ephemeral session keys ensure past sessions cannot be decrypted
- **Encrypted Client Hello (ECH)**: exit nodes support ECH to hide the destination domain name from SNI when the destination publishes ECH configuration
- **Pluggable transport**: obfuscates BGP-X traffic patterns to resist ISP-level blocking
- **Mesh transport native**: operates without any ISP connection via WiFi 802.11s, LoRa radio, Bluetooth, and other transports
- **Cover traffic**: COVER packets are cryptographically indistinguishable from RELAY packets to external observers — both use the session key
- **Geographic plausibility scoring**: RTT-based verification of node region claims (no external database required) — OPTIONAL feature; jurisdiction declaration is opt-in
- **path_id return routing**: 8-byte random path identifier enables return traffic routing without leaking path composition
- **.bgpx native addressing**: self-authenticating service addresses with human-readable short names via Name Registry DHT
- **HTTP/2 native transport**: BGP-X native services use HTTP/2 over BGP-X streams for efficient multiplexing on high-latency paths

---

## Three Equal Entry Points

BGP-X provides three equal first-class entry points to the network:

| Entry Point | Description | Hardware |
|---|---|---|
| **Clearnet** | BGP-routed internet (fiber, cellular, satellite internet — all are clearnet) | Any internet-connected device |
| **BGP-X Overlay** | Onion-encrypted relay network | Any BGP-X node |
| **Mesh Islands** | Community radio networks (WiFi mesh, LoRa, BLE) | BGP-X Router v1, BGP-X Node v1 |

No entry point is primary. A mesh-only user in a remote village and a fiber-connected user in a city participate in the same network. A clearnet client with no mesh hardware can reach services hosted in mesh islands via domain bridge nodes.

---

## Routing Domains

BGP-X treats three network classes as equal first-class citizens:

```
┌─────────────────┐         ┌───────────────────┐         ┌──────────────────┐
│  CLEARNET/BGP   │◄───────►│   BGP-X OVERLAY   │◄───────►│   MESH ISLANDS   │
│                 │  Domain │                   │  Domain │                  │
│ Standard TCP/IP │  Bridge │ Onion-encrypted   │  Bridge │ WiFi mesh        │
│ BGP routing     │ Nodes   │ ChaCha20-Poly1305 │ Nodes   │ LoRa radio       │
│ Client device   │◄───────►│ DHT P2P discovery │◄───────►│ BLE              │
│ Server on inet  │         │ Multi-hop paths   │         │ Ethernet P2P     │
└─────────────────┘         └───────────────────┘         └──────────────────┘
         ▲                          ▲  ▲                           ▲
         │                          │  │                           │
         └─────── Any combination, any order, unlimited hops ──────┘
```

A clearnet user with no BGP-X router can reach a mesh island service. A mesh user with no ISP can reach clearnet via a gateway. Two islands connect through the overlay. Three domains in sequence: valid. Fifteen hops across four domains: valid. The protocol imposes no ordering, no count limit.

---

## .bgpx Domain System

BGP-X provides a native service addressing system equivalent to Tor's .onion:

- **Self-authenticating addresses**: `a3f2b9c1....bgpx` — the address IS the service's Ed25519 public key
- **Human-readable names**: `community-news.bgpx` via BGP-X Name Registry DHT
- **Cross-domain access**: clearnet users can reach mesh island services via domain bridge nodes
- **Latency-adaptive**: BGP-X Browser handles LoRa-class paths with batched resource loading and progressive rendering

`.bgpx` services use HTTP/2 over BGP-X streams. HTTP/2 multiplexing reduces round-trips on high-latency paths. HTTP/3 is used only at exit nodes connecting to HTTP/3 clearnet servers — not within the BGP-X overlay.

---

## HTTP/2 for .bgpx Services

BGP-X native services (.bgpx addresses) use **HTTP/2 over BGP-X streams**.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at exit nodes when connecting to HTTP/3 clearnet servers — standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths: HTTP/2 multiplexing is critical. Each round-trip costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## Deployment Modes

| Mode | Description | Hardware | ISP Required |
|---|---|---|---|
| Dual-Stack Router | BGP + BGP-X coexist; routing policy decides per flow | BGP-X Router v1 | Yes |
| BGP-X Only Router | All LAN traffic through overlay; no bypass for LAN devices | BGP-X Router v1 | Yes |
| Standalone Device | BGP-X daemon on laptop/desktop/server | Any computer | Yes |
| Mesh Relay | No ISP; mesh transport only | BGP-X Node v1 or Router v1 | No |
| Community Gateway | Bridges mesh island to clearnet | BGP-X Node v1 or Router v1 | Yes (at gateway) |
| Range Extension | Coverage extension with full BGP-X encryption | BGP-X Node v1 in Range Extension mode | No |

---

## Application Types

| Type | Description | Configuration Required |
|---|---|---|
| Standard Application | Any app that doesn't use the SDK; transparently protected via router | None — just connect to the network |
| BGP-X Native Application | Uses the SDK to control paths, pools, domains, ECH; can register as native services | SDK integration |
| Configuration Client | bgpx-cli and GUI tools; connects to daemon via Control Socket | Socket access |

---

## Hardware Ecosystem

BGP-X provides purpose-built hardware for every participant tier:

| Hardware | Target User | Key Characteristics |
|---|---|---|
| **BGP-X Router v1** | End user | All-in-one; replaces home router; WiFi + LoRa + BLE; outdoor IP67 option; satellite WAN via USB |
| **BGP-X Node v1** | Community contributor | Compact outdoor node; solar-powered option; runs complete bgpx-node daemon; domain bridge capable |
| **BGP-X Gateway v1** | Provider | High-throughput exit node; SFP+ uplink; 1 GB RAM; datacenter deployment |
| **BGP-X Client Node** | Individual endpoint | Low-cost (LILYGO T3 based); battery-powered; personal connectivity; no routing for others |
| **BGP-X Adapter/Dongle** | USB LoRa modem | USB-powered; acts as LoRa interface for existing computers |
| **BGP-X OpenWrt Package** | Existing hardware | Software-only for compatible third-party routers |

**Key clarification**: BGP-X Node v1 runs the **complete** bgpx-node daemon — the same software as BGP-X Router v1. The difference is form factor and deployment context, not capability. It is NOT a "lite" or partial device.

---

## BGP and BGP-X Relationship

BGP-X uses the BGP-routed internet as one transport layer among several. BGP-X packets are ordinary UDP datagrams from BGP's perspective — opaque, encrypted, indistinguishable from any other UDP traffic. BGP-X does not participate in BGP peering, does not announce routes, and does not require any ISP or infrastructure changes.

For mesh deployments (Modes 4-6), BGP-X operates with zero BGP involvement for intra-mesh traffic. Coverage gaps between mesh islands are bridged via internet relay pools transparently and privately.

---

## BGP Replacement Spectrum

BGP-X enables a spectrum of BGP independence depending on deployment:

- **Level 0**: Pure overlay — BGP as full transport, BGP-X adds privacy only
- **Level 1**: Intra-mesh routing — 100% BGP-free for local mesh traffic
- **Level 2**: Mesh + clearnet via gateway — mesh users have zero direct BGP exposure
- **Level 3**: Multi-island bridging — mesh islands connected through internet relay pools; users never directly exposed to BGP
- **Level 4**: Dense multi-mode hybrid — all modes simultaneously, variable BGP dependence by position
- **Level 5**: Wide-area mesh with aerial nodes — near-total BGP independence for intra-network
- **Level 6**: Maximum — BGP-X replaces ISP routing decisions; honest limit: cannot replace physical infrastructure

**Honest assessment**: BGP-X replaces ISP dependency and routing decisions, but cannot replace physical infrastructure (fiber, towers, spectrum).

---

## Prior Art

BGP-X builds on significant prior work. It does not claim to be the first in any individual dimension:

- **cjdns / Hyperboria**: public key addressing for overlay networks
- **Yggdrasil Network**: DHT-based routing with public key identities
- **Reticulum Network Stack**: multi-transport mesh networking designed for low-bandwidth links
- **Tor**: multi-hop onion routing with clearnet exit nodes
- **I2P**: garlic routing for anonymous services
- **Meshtastic**: LoRa mesh hardware and protocol

**What BGP-X uniquely combines**: multi-hop onion routing + mesh transport native + clearnet exit model + decentralized unified DHT + pool-based trust domains + hardware targets + cross-domain inter-protocol routing + N-hop unlimited path construction — unified into one coherent system.

---

## How BGP-X Differs from Tor

| Property | Tor | BGP-X |
|---|---|---|
| Architecture | Fixed 3-hop circuit | Variable N-hop path, client-configurable, no maximum |
| Directory model | Centralized directory authorities | Decentralized unified DHT spanning all routing domains |
| Transport | TCP streams | UDP-native with reliability layer |
| Identity model | Single anonymous identity per circuit | Cryptographic identity; rotatable per session |
| Path length | Fixed 3 hops | Configurable; no global enforcement maximum |
| Domain routing | None | Cross-domain: clearnet ↔ overlay ↔ mesh in any order |
| Entry points | Internet only | Clearnet, BGP-X overlay, or mesh — any is first-class |
| Congestion control | None (TCP passthrough) | Native BGP-X congestion and backpressure |
| Multiplexing | Single stream per circuit | Native stream multiplexing over single path |
| Exit model | Any volunteer relay can exit | Designated, signed, policy-publishing exit nodes |
| Network layer | Application proxy (SOCKS5) | Full IP overlay via TUN interface |
| Cover traffic | Optional, not default | Pluggable; uses session_key; externally indistinguishable |
| Firmware support | No | Yes — router firmware targets |
| Path trust model | Assumes 1 in 3 hops honest | Configurable; reputation-weighted selection |
| DHT Pools | No | Yes — multi-pool paths, double-exit, trust tiers |
| ECH at exit | No | Yes — hides SNI when destination supports it |
| Mesh transport | No | Yes — WiFi mesh, LoRa, Bluetooth |
| Hardware ecosystem | No | Yes — Router v1, Node v1, Gateway v1, Client Node, Adapter |
| Cross-domain routing | No | Yes — any combination of network domains |
| .bgpx native services | No | Yes — self-authenticating addresses with Name Registry |
| HTTP/2 native | No | Yes — for .bgpx services |
| N-hop unlimited | No (fixed 3) | Yes — no protocol maximum |
| Satellite clarification | N/A | Yes — satellite internet = clearnet domain |

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

## Architecture in One Diagram

```
[All Devices on LAN]
         ↓
[BGP-X Router Daemon]
  Routing Policy Engine decides per-flow:
  ├── BGP-X overlay → Entry → Relay → Relay → Exit → Clearnet
  ├── BGP-X cross-domain → Entry → [clearnet relays] → Bridge → [mesh relays] → Mesh Service
  └── Standard routing → Direct WAN → Clearnet
         ↓
[Three Interfaces]
  ├── TUN (bgpx0): transparent IP capture
  ├── SDK Socket: BGP-X native apps
  └── Control Socket: bgpx-cli and management tools
```

---

## Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal:

- If a node declares a jurisdiction: geo plausibility scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply
- Nodes are NOT required to declare jurisdiction
- Declaring jurisdiction is an opt-in privacy/convenience tradeoff

Satellite-connected nodes are **exempt** from geo plausibility scoring because satellite terminal IPs may be assigned from distant ground stations.

---

## Known Limitations

BGP-X is explicit about what it does not guarantee:

- **Timing correlation**: all low-latency anonymous networks share this fundamental residual risk; cover traffic raises attacker cost but does not eliminate it
- **ECH requires destination support**: destinations must publish ECH configuration in DNS HTTPS records
- **Geographic plausibility is RTT-signal-based**: not cryptographic proof of node location
- **ISP can observe BGP-X participation**: your ISP sees encrypted UDP/7474 traffic to BGP-X entry nodes; pluggable transport mitigates this
- **Application-layer identity leakage**: BGP-X cannot prevent an app from revealing your identity via cookies, logins, or fingerprinting
- **Compromised device**: BGP-X cannot protect traffic from malware on your device
- **Cross-domain bridge availability**: if no bridge node exists between two domains, cross-domain connectivity is not possible for that pair
- **No legal protection**: BGP-X does not make illegal activities legal
- **LoRa paths have high latency**: LoRa paths introduce 500ms-5s latency per round-trip. Standard browsers feel slow. BGP-X Browser handles this gracefully with batched loading and progressive rendering
- **Satellite latency**: GEO satellite paths add 600ms+ RTT. Suitable for non-interactive traffic; challenging for real-time applications
- **No post-quantum cryptography (v1)**: Current version uses classical cryptography. Post-quantum upgrade planned for future version

---

## What BGP-X Is Not

- ❌ A replacement for BGP (the Border Gateway Protocol)
- ❌ A VPN (VPNs shift trust to a single provider; BGP-X distributes it)
- ❌ A Tor fork (fundamentally different architecture, protocol, and threat model)
- ❌ An anonymizing proxy (operates at the network layer, not application layer)
- ❌ A silver bullet (see Known Limitations above)
- ❌ A per-application tool (BGP-X runs on the router and protects all devices)
- ❌ The first mesh network (builds on significant prior art)

---

## Repository Structure

```
/bgp-x
├── README.md                   — This file
├── ARCHITECTURE.md             — Full system architecture
├── CHANGELOG.md                — Version history
├── CONTRIBUTING.md             — Contribution guidelines
├── CODE_OF_CONDUCT.md          — Community standards
├── LICENSE                     — MIT (code)
├── LICENSE-HARDWARE            — CERN-OHL-S v2 (hardware)
├── LICENSE-DOCS                — CC BY 4.0 (documentation)
├── SECURITY.md                 — Vulnerability reporting
├── /docs                       — User-facing documentation
│   └── /user_guide             — Setup guides for each deployment mode
├── /protocol                   — Protocol specification
│   └── routing_domains.md      — Cross-domain routing specification
├── /control-plane              — Routing logic and node management
├── /data-plane                 — Packet forwarding and encryption
├── /node                       — Node daemon specification
├── /gateway                    — BGP interoperability layer
├── /firmware                   — Router firmware targets
├── /client                     — Configuration client specification
├── /sdk                        — Developer SDK specification
├── /security                   — Security model and auditing
├── /simulation                 — Network and attack simulation
├── /deployment                 — Deployment and operations guides
├── /production                 — SOPs, certification, release
├── /hardware                   — Hardware specifications
├── /application                — BGP-X Browser, native app framework, SDK integration
└── /legal                      — Liability and privacy documentation
```

---

## Status

BGP-X is in **pre-implementation specification phase**.

All documentation represents a complete system design for v1 implementation. No features are deferred to future versions.

Phase plan:

- [x] System architecture
- [x] Protocol specification
- [x] Security model
- [x] Mesh architecture
- [x] Cross-domain routing (inter-protocol domain router model)
- [x] Hardware specification
- [x] Application layer (BGP-X Browser, native apps)
- [ ] Reference implementation (Rust)
- [ ] Testnet
- [ ] Mainnet

---

## License

- **Code**: MIT License. See [LICENSE](./LICENSE).
- **Hardware**: CERN-OHL-S v2. See [LICENSE-HARDWARE](./LICENSE-HARDWARE).
- **Documentation**: CC BY 4.0. See [LICENSE-DOCS](./LICENSE-DOCS).

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## Security

To report a vulnerability, see [SECURITY.md](./SECURITY.md).
