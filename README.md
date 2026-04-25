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
