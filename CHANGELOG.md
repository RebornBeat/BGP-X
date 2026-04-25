# BGP-X Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
BGP-X uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Reference implementation in Rust
- BGP-X node daemon (bgpx-node)
- BGP-X configuration client (bgpx-cli)
- BGP-X gateway daemon
- BGP-X mesh firmware
- BGP-X SDK (Rust primary, Python/TypeScript/Go bindings)
- Simulation harness
- Testnet launch

---

## [0.1.0] — Pre-implementation specification

### Scope

All features listed below are included in the v1 specification. Nothing is deferred to future versions. This is the complete v1 design.

### Architecture Added

- Router-centric architecture: BGP-X daemon as single routing stack; SDK and config client as daemon clients
- Six deployment modes: Dual-Stack Router, BGP-X Only Router, Standalone Device, Mesh Node, Gateway Node, Broadcast Amplifier
- Three local interfaces: TUN (bgpx0), SDK Socket, Control Socket
- Application type taxonomy: Standard (transparent), Native SDK, Config Client
- BGP Replacement Spectrum: 6 levels from overlay-only to maximum independence
- Routing Policy Engine: per-flow decision making with rule types (device, application, destination, protocol/port, pool-specific)
- BGP and BGP-X coexistence on dual-stack router: one WAN, two routing planes

### Protocol Added

- Full system architecture (ARCHITECTURE.md)
- Protocol specification (/protocol/protocol_spec.md)
- Packet format (/protocol/packet_format.md) — includes path_id (8-byte return routing field) in onion layer header
- Handshake specification (/protocol/handshake.md) — explicit two-part HANDSHAKE_INIT (cleartext: client ephemeral key; encrypted: all other fields)
- Path construction (/protocol/path_construction.md) — pool-aware, multi-segment
- DHT Pools specification (/protocol/pool_spec.md)
- Pluggable transport specification (/protocol/pluggable_transport.md)
- Path quality reporting specification (/protocol/path_quality_reporting.md)
- Mesh transport specification (/protocol/mesh_transport.md)
- Pool curator key rotation specification (/protocol/pool_curator_key_rotation.md)
- Error handling (/protocol/error_handling.md)
- Protocol versioning (/protocol/versioning.md)

### Control Plane Added

- Node discovery via Kademlia DHT (/control-plane/discovery.md)
- Node advertisement with all new fields (/control-plane/node_advertisement.md)
- Pool-aware routing algorithm (/control-plane/routing_algorithm.md) — 5-component scoring including geographic plausibility
- Local-primary reputation system (/control-plane/reputation_system.md)
- Geographic plausibility scoring (/control-plane/geo_plausibility.md)
- Full Control API (/control-plane/control_api.md)

### Data Plane Added

- Forwarding pipeline (/data-plane/forwarding.md) — path_id → predecessor routing for return traffic
- Encryption layers (/data-plane/encryption_layers.md) — bidirectional session model; cover traffic uses session_key (no separate cover_key); ECH at exit nodes
- Congestion control (/data-plane/congestion_control.md)
- Stream multiplexing (/data-plane/multiplexing.md) — Stream IDs: client uses odd, service uses even

### Design Decisions Finalized

- UDP-native transport with custom reliability layer
- Ed25519 + X25519 + ChaCha20-Poly1305 cryptographic suite
- BLAKE3 for all hashing
- DHT-based decentralized discovery (no directory authorities)
- Client-selected paths (source routing)
- Minimum 3-hop, default 4-hop paths (no global maximum enforcement)
- DHT Pools with pool curator Ed25519 keys and dual-signature key rotation
- path_id (8 bytes, CSPRNG) for return traffic routing
- Bidirectional sessions: two independent sequence number spaces (inbound/outbound) per session
- COVER uses session_key — externally indistinguishable from RELAY
- Geographic Plausibility Scoring: RTT-based (no external database)
- Local-first reputation: global blacklists advisory, opt-in enforcement
- ECH at exit nodes when destination publishes HTTPS DNS record
- Pluggable transport: obfs4-style; built-in + external subprocess interface
- Session re-handshake at 24 hours for long-lived sessions
- Advertisement withdrawal: signed NODE_WITHDRAW message; 48hr natural expiry as fallback
- Exit policy versioning: policy_version field; clients re-fetch on version change
- Pool curator key rotation: dual-signature (old + new key); 24hr window standard; 2hr for compromise_confirmed
- DNS at exit nodes: DoH default, DNSSEC validation, ECS stripping mandatory
- Local blacklist expiry: 30-day default; configurable; cryptographic violations permanent
- Mesh transport: WiFi 802.11s, LoRa, Bluetooth BLE, Ethernet P2P
- Hardware: unified BGP-X Node platform (single PCB, multiple carrier/enclosure variants)

### Acknowledged Prior Art

- cjdns (public key addressing)
- Yggdrasil Network (DHT routing)
- Reticulum Network Stack (multi-transport mesh)
- Tor (onion routing)
- I2P (anonymous services)
- Meshtastic (LoRa mesh hardware)
- Kademlia (DHT algorithm)

### Removed from Design

- Guard mode (replaced by DHT Pools for trust isolation without observable long-term entry relationships)
- Global maximum path length enforcement (defaults remain configurable)
- Separate cover_key derivation (COVER uses session_key)
- Debug/trace log levels from production (valid: error, warn, info)
