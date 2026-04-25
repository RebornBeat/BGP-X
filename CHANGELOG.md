# BGP-X Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
BGP-X uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Reference implementation in Rust
- BGP-X node daemon (bgpx-node)
- BGP-X client library (bgpx-client)
- BGP-X gateway daemon (bgpx-gateway)
- BGP-X CLI tooling
- Simulation harness
- Testnet launch

---

## [0.1.0] — Pre-implementation specification

### Added
- Full system architecture (ARCHITECTURE.md)
- Protocol specification (/protocol/protocol_spec.md)
- Packet format (/protocol/packet_format.md)
- Handshake specification (/protocol/handshake.md)
- Path construction specification (/protocol/path_construction.md)
- Error handling specification (/protocol/error_handling.md)
- Protocol versioning policy (/protocol/versioning.md)
- Control plane: discovery, advertisement, routing algorithm, reputation, API
- Data plane: forwarding, encryption layers, congestion control, multiplexing
- Node specification
- Gateway and BGP interop specification
- Client specification
- SDK specification
- Threat model and security specifications
- Simulation specifications
- Deployment and production SOPs
- Legal documentation

### Design decisions finalized
- UDP-native transport with custom reliability layer
- Ed25519 + X25519 + ChaCha20-Poly1305 cryptographic suite
- BLAKE3 for all hashing
- DHT-based decentralized discovery (no directory authorities)
- Client-selected paths (source routing)
- Minimum 3-hop, default 4-hop paths
- Signed, auditable exit node policy framework
