# BGP-X

**BGP-X** is a **router-level privacy overlay network** that unifies multiple transport protocols and routing domains into a single, cryptographically enforced privacy plane.

It runs directly on your router (or as a dedicated node) and protects **all connected devices** without per-device configuration. It is **not** a per-application tool, **not** a VPN you connect to, and **not** a Tor fork.

BGP-X uses the existing internet (BGP-routed) as a **dumb transport substrate** while building a parallel, identity-based, multi-hop overlay with onion encryption, decentralized discovery, and native mesh support.

---

## Why BGP-X Exists

Modern networks are fragmented:

- **BGP-routed internet** — reachability-focused, metadata-leaky
- **Mesh networks** (WiFi 802.11s, LoRa, Bluetooth, etc.) — local, often no internet
- **Tor** — strong anonymity, but TCP-only, application-layer proxy, centralized directory
- **VPNs** — single-provider trust shift
- **cjdns / Yggdrasil / Reticulum** — excellent pieces, but incomplete unification

BGP-X solves the unification problem:

> **You should be able to communicate across any combination of protocols, networks, and infrastructures — such that no single party can link your identity to your destination — while retaining production-grade performance and router-level deployment.**

---

## What BGP-X Is

BGP-X is a **multi-protocol, router-level privacy overlay** with these core properties:

- **Router-centric**: Runs as the routing stack on your router; protects every device on the LAN transparently
- **Identity-based addressing**: Nodes and services identified by cryptographic public keys, not IP addresses
- **Client-selected paths**: Sender chooses the relay path (including cross-protocol segments)
- **Layered encryption**: Onion-encrypted packets; each relay decrypts only its layer
- **DHT Pools**: Trust-segmented discovery with public, curated, private, and ephemeral pools
- **Native mesh support**: WiFi 802.11s, LoRa, Bluetooth, Ethernet P2P, satellite — operates without ISP
- **Gateway interoperability**: Audited exit nodes bridge mesh ↔ internet; double-exit architecture possible
- **ECH at exits**: Hides domain name from SNI (when destination supports ECH)
- **Pluggable transport**: Obfuscation layer for DPI resistance
- **Geographic plausibility scoring**: RTT-based verification as reputation signal
- **No central directory**: Fully decentralized DHT + signed pool advertisements

---

## Core Unification (Our Selling Point)

**Tor operates on the BGP-routed internet.**

**BGP-X operates *between* protocol levels** — routing between:
- BGP-routed internet
- Mesh networks (WiFi, LoRa, Bluetooth, etc.)
- Satellite links
- Cellular
- Private networks

**Pools** enable this unification:
- Public pools for open participation
- Curated pools for trusted operators
- Private pools for your own infrastructure
- Ephemeral pools for one-off high-security sessions

This allows mesh communities to connect to each other and to the internet through audited gateways while preserving strong unlinkability.

---

## Deployment Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Dual-Stack Router** | BGP + BGP-X coexist on one router | Most home/small business |
| **BGP-X Only Router** | All traffic through overlay | Maximum privacy |
| **Standalone Device** | Single-device protection | Laptop, phone, server |
| **Mesh Node** | No ISP; pure mesh transport | Off-grid, community networks |
| **Gateway Node** | Mesh ↔ Internet bridge | Connect mesh islands to clearnet |
| **Broadcast Amplifier** | Range extension only; no routing | Extend coverage in remote areas |

---

## How BGP-X Differs from Tor

| Property | Tor | BGP-X |
|---|---|---|
| Architecture | Fixed 3-hop circuit | Configurable multi-segment paths across protocols |
| Discovery | Centralized directory authorities | Decentralized DHT + signed pools |
| Transport | TCP only | Any IP + native mesh (WiFi, LoRa, BLE, satellite) |
| Deployment | Application proxy | Router-level (protects all devices) |
| Exit model | Volunteer exits | Audited, signed exit nodes with ECH |
| Multiplexing | One circuit per stream | Native multiplexing over single path |
| Cover traffic | Optional | Pluggable, uses session_key (indistinguishable from RELAY) |
| Mesh support | No | Native |
| ECH at exit | Partial | Yes (when destination supports) |

---

## Architecture in One Diagram

```
[LAN Devices] ──► [BGP-X Router Daemon]
                     │
          ┌──────────┼──────────┐
          │   Routing Policy Engine   │
          └──────────┼──────────┘
                     │
          ┌──────────┼──────────┐
          │   Overlay Routing Layer   │ (onion + pools)
          └──────────┼──────────┘
                     │
   Mesh Transport ───┼─── Internet Transport (BGP-routed)
          │          │          │
     [Mesh Island]  [Gateway]  [Clearnet]
```

---

## Repository Structure

```
/bgp-x
├── README.md
├── ARCHITECTURE.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── LICENSE
├── SECURITY.md
├── /docs
│   ├── getting_started.md
│   ├── architecture_overview.md
│   ├── deployment_architecture.md
│   ├── routing_policy.md
│   ├── bgp_bgpx_coexistence.md
│   ├── application_guide.md
│   ├── mesh_architecture.md
│   ├── ecosystem_unification.md
│   ├── regulatory_framework.md
│   └── faq.md
├── /protocol
│   ├── protocol_spec.md
│   ├── packet_format.md
│   ├── handshake.md
│   ├── path_construction.md
│   ├── pool_spec.md
│   ├── pluggable_transport.md
│   ├── path_quality_reporting.md
│   ├── mesh_transport.md
│   ├── pool_curator_key_rotation.md
│   ├── error_handling.md
│   └── versioning.md
├── /control-plane
│   ├── discovery.md
│   ├── node_advertisement.md
│   ├── routing_algorithm.md
│   ├── geo_plausibility.md
│   ├── reputation_system.md
│   └── control_api.md
├── /data-plane
│   ├── forwarding.md
│   ├── encryption_layers.md
│   ├── congestion_control.md
│   └── multiplexing.md
├── /node
│   ├── node.md
│   └── api.md
├── /gateway
│   ├── gateway_spec.md
│   ├── exit_node.md
│   ├── entry_node.md
│   └── bgp_interop.md
├── /firmware
│   └── firmware.md
├── /client
│   └── control_client.md
├── /sdk
│   └── sdk_spec.md
├── /security
│   ├── threat_model.md
│   ├── attack_vectors.md
│   ├── crypto_spec.md
│   └── audit_plan.md
├── /simulation
│   ├── network_sim.md
│   ├── attack_sim.md
│   └── performance.md
├── /deployment
│   ├── bootstrap_nodes.md
│   ├── node_setup.md
│   ├── scaling.md
│   ├── mesh_deployment.md
│   ├── mast_tower_deployment.md
│   ├── solar_deployment.md
│   ├── aerial_deployment.md
│   ├── vehicle_deployment.md
│   ├── maritime_deployment.md
│   ├── underground_deployment.md
│   └── satellite_gateway_deployment.md
├── /production
│   ├── sop.md
│   ├── node_certification.md
│   └── release_process.md
├── /legal
│   ├── liability.md
│   └── privacy_policy.md
└── /hardware
    ├── README.md
    ├── node_spec.md
    ├── gateway_spec.md
    ├── amplifier_spec.md
    ├── compatible_hardware.md
    ├── meshtastic_adapter.md
    └── manufacturing.md
```

---

## Status

BGP-X is in the **pre-implementation specification phase**.

All documentation in this repository represents a complete system design intended to guide implementation. No production code has been written yet.

Phase plan:

- [x] System architecture (including router-centric model, pools, mesh, ECH)
- [x] Protocol specification (including path_id, pool support, mesh transport, ECH)
- [x] Security model (including pool threats, mesh threats)
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
