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

---

### Architecture Added

- Router-centric architecture: BGP-X daemon as single routing stack; SDK and config client as daemon clients
- Six deployment modes: Dual-Stack Router, BGP-X Only Router, Standalone Device, Mesh Node, Gateway Node (generalized as Domain Bridge Node), Broadcast Amplifier
- Three equal entry points: clearnet, BGP-X overlay, and mesh islands as first-class routing domains with no priority ordering or domain restriction
- Inter-Protocol Domain Router layer: cross-domain path construction, domain bridge node selection, N-hop unlimited routing
- Three local interfaces: TUN (bgpx0), SDK Socket, Control Socket
- Application type taxonomy: Standard (transparent), Native SDK, Config Client
- BGP Replacement Spectrum: 6 levels from overlay-only to maximum independence
- Routing Policy Engine: per-flow decision making with domain-aware rule types
- BGP and BGP-X coexistence on dual-stack router: one WAN, two routing planes
- Domain bridge node model: generalization of gateway to any multi-domain node

---

### Protocol Added

- Full system architecture (ARCHITECTURE.md)
- Protocol specification (/protocol/protocol_spec.md) — includes N-hop unlimited policy statement, three equal entry points, unified DHT model
- Packet format (/protocol/packet_format.md) — includes:
  - path_id (8-byte return routing field) in onion layer header
  - New hop types: DOMAIN_BRIDGE (0x06), MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09)
  - Domain ID wire format: type (4B BE) + instance hash (4B BE) = 8 bytes
  - DOMAIN_ADVERTISE message (0x1A): node announces routing domain endpoints and bridge capability
  - MESH_ISLAND_ADVERTISE message (0x1B): mesh island announces existence, bridge nodes, and services
  - PATH_QUALITY_REPORT extended to 20 bytes (16 bytes payload + 4-byte domain_id prefix) — was 16 bytes in draft
  - MESH_BEACON extended with bridge capability flag and served domain IDs
  - POOL_KEY_ROTATION message (0x1C)
  - N-hop unlimited: no hop counter in common header; no protocol mechanism drops packets after N hops
- Handshake specification (/protocol/handshake.md) — domain-agnostic: identical HANDSHAKE_INIT/RESP/DONE format for all routing domains; HANDSHAKE_DONE carries path_id for cross-domain path table initialization at bridge nodes; cross-domain session establishment via relay chain
- Routing domains specification (/protocol/routing_domains.md) — NEW: authoritative domain type definitions, domain ID wire format, domain bridge node definition, cross-domain onion layer construction
- Path construction (/protocol/path_construction.md) — domain-aware, multi-segment, cross-domain, any-to-any with N-hop unlimited
- DHT Pools specification (/protocol/pool_spec.md) — includes domain-scoped pools and domain-bridge pools
- Pluggable transport specification (/protocol/pluggable_transport.md)
- Path quality reporting specification (/protocol/path_quality_reporting.md) — includes domain_id field; domain-calibrated latency buckets
- Mesh transport specification (/protocol/mesh_transport.md) — mesh as first-class routing domain; unified DHT for mesh (replaces two-DHT model); inter-island routing
- Pool curator key rotation specification (/protocol/pool_curator_key_rotation.md)
- Error handling (/protocol/error_handling.md) — includes new error codes 0x001E-0x0024 for cross-domain errors
- Protocol versioning (/protocol/versioning.md) — includes extension flags 10-12 for domain bridge, cross-domain routing, mesh island routing

---

### Control Plane Added

- Node discovery via unified Kademlia DHT (/control-plane/discovery.md) — unified single DHT replacing two-DHT model; domain bridge discovery; mesh island discovery; domain-filtered DHT queries
- Node advertisement with all new fields (/control-plane/node_advertisement.md) — routing_domains array replaces endpoints + mesh_endpoints; bridge_capable and bridges fields; island_memberships; new extension flags domain_bridge, cross_domain_routing, mesh_island_routing
- Pool-aware cross-domain routing algorithm (/control-plane/routing_algorithm.md) — domain-aware scoring, bridge node scoring function, cross-domain diversity enforcement across entire path
- Local-primary reputation system (/control-plane/reputation_system.md) — domain-aware event tagging, per-domain reputation tracking for bridge nodes, domain bridge failure events
- Geographic plausibility scoring (/control-plane/geo_plausibility.md) — domain-specific thresholds: clearnet vs WiFi mesh vs LoRa vs satellite; bridge node independent per-domain scoring
- Full Control API (/control-plane/control_api.md) — includes domain management methods: list_domains, list_domain_bridges, get_island_status, list_islands, test_domain_connectivity, discover_bridge_nodes; updated events including cross-domain events

---

### Data Plane Added

- Forwarding pipeline (/data-plane/forwarding.md) — includes DOMAIN_BRIDGE (0x06), MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09) hop dispatch; cross-domain path_id table (separate from single-domain path table); domain transport selector; return path routing across domain boundaries; Concurrency Model with sharded hash maps, receiver threads, worker threads, sender threads
- Encryption layers (/data-plane/encryption_layers.md) — domain-agnostic crypto section confirming identical suite in all domains; cross-domain onion construction example; no re-encryption at domain boundaries; COVER domain-agnostic (session_key in all domains)
- Congestion control (/data-plane/congestion_control.md) — domain-calibrated baselines; domain-level congestion identification via domain_id in PATH_QUALITY_REPORT; per-domain segment rebuild; multi-stream bandwidth sharing with WFQ and stream priority table; path quality scoring formula; LoRa duty cycle as hard cap; cover traffic domain-aware minimum rates
- Stream multiplexing (/data-plane/multiplexing.md) — cross-domain stream continuity (stream_id unchanged across domain boundaries); PAUSED stream state for offline islands and LoRa duty cycle; store-and-forward vs. PAUSED state distinction

---

### Design Decisions Finalized

**Cryptographic suite (unchanged, domain-agnostic):**
- X25519 for key exchange
- Ed25519 for signatures
- ChaCha20-Poly1305 for symmetric encryption (all domains, no domain-specific cipher)
- BLAKE3 for all hashing
- HKDF-SHA256 for KDF (identical salts and info strings in all domains)
- No separate cover_key — COVER uses session_key; externally indistinguishable from RELAY in all domains

**Architecture decisions:**
- Unified DHT spanning all routing domains (replaces two-DHT model with gateway synchronization)
- N-hop unlimited: no protocol-level maximum on path hops, domain count, or domain traversal count
- Three equal entry points: clearnet, overlay, mesh — no priority, no required ordering, no domain restriction
- DOMAIN_BRIDGE (0x06) as a first-class hop type in the onion layer
- Domain-agnostic handshake: one HANDSHAKE_INIT/RESP/DONE protocol for all routing domains and transports
- No re-encryption at domain boundaries: bridge node forwards inner layers without modification
- Cross-domain path_id table at bridge nodes: separate from single-domain path table, enables return routing across domain boundaries
- Domain ID wire format: type (4B BE) + instance hash (4B BE) = 8 bytes
- Routing_domains array in node advertisement: replaces dual endpoints + mesh_endpoints model
- Bridge capability declared in advertisement with bridges array
- Mesh islands as DHT entities: MESH_ISLAND_ADVERTISE; discoverable from clearnet via unified DHT
- Any-to-any connectivity: protocol imposes no domain ordering or entry point privilege

**Retained decisions from draft:**
- DHT-based decentralized discovery (no directory authorities)
- Client-selected paths (source routing)
- Minimum 3-hop, default 4-hop paths (no global maximum enforcement)
- DHT Pools with pool curator Ed25519 keys and dual-signature key rotation
- path_id (8 bytes, CSPRNG) for return traffic routing
- Bidirectional sessions: two independent sequence number spaces (inbound/outbound) per session
- Geographic Plausibility Scoring: RTT-based (no external database), domain-specific thresholds
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
- Meshtastic adaptation layer: ESP32/nRF52840 as LoRa radio modem for BGP-X bridge nodes

---

### Removed/Replaced from Design

- Guard mode (replaced by DHT Pools for trust isolation)
- Global maximum path length enforcement (N-hop unlimited replaces all enforcement)
- Separate cover_key derivation (COVER uses session_key)
- Two-DHT model (internet DHT + mesh DHT with gateway sync) — replaced by unified DHT spanning all domains
- Debug/trace log levels from production (valid: error, warn, info)

---

### New Error Codes

- 0x001E ERR_DOMAIN_NOT_FOUND
- 0x001F ERR_DOMAIN_BRIDGE_UNAVAILABLE
- 0x0020 ERR_MESH_ISLAND_UNREACHABLE
- 0x0021 ERR_NO_CROSS_DOMAIN_PATH
- 0x0022 ERR_DOMAIN_POLICY_DENIED
- 0x0023 ERR_DOMAIN_SCOPE_MISMATCH
- 0x0024 ERR_DOMAIN_BRIDGE_POOL_INVALID

---

### New Extension Flags

- Bit 10: DOMAIN_BRIDGE_SUPPORT
- Bit 11: CROSS_DOMAIN_PATH_CONSTRUCTION
- Bit 12: MESH_ISLAND_ROUTING

---

### Acknowledged Prior Art

- cjdns (public key addressing)
- Yggdrasil Network (DHT routing)
- Reticulum Network Stack (multi-transport mesh)
- Tor (onion routing)
- I2P (anonymous services)
- Meshtastic (LoRa mesh hardware)
- Kademlia (DHT algorithm)
