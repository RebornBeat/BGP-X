# BGP-X: Ecosystem Unification

This document explains the fragmented mesh networking ecosystem, why existing systems have failed to reach widespread adoption, and how BGP-X provides the unifying architecture.

---

## The Fragmentation Problem

The mesh networking community has built many valuable systems over the past two decades. Each solves part of the problem. None has achieved widespread adoption because each is isolated in its capabilities.

When a community wants to deploy private mesh networking, they face an impossible choice:

```
Want LoRa range?                    → Meshtastic (no privacy, no clearnet)
Want community WiFi?                → NYC Mesh (no privacy, no mesh federation)
Want anonymity?                     → Tor (no mesh transport, internet-only)
Want privacy over mesh?             → Reticulum (no onion routing, no clearnet)
Want public key addressing?         → cjdns (no anonymity, no clearnet exit)
Want ISP-free operation?            → Meshtastic or Reticulum (limited capabilities)
Want full internet access?          → VPN (single trust point, no mesh)
```

No existing system provides all of these properties simultaneously. Every community must choose and sacrifice.

---

## What Each System Provides (and Doesn't)

### Capability Matrix

| Feature | Meshtastic | NYC Mesh | Tor | I2P | cjdns | Reticulum | Helium | BGP-X |
|---|---|---|---|---|---|---|---|---|
| LoRa transport | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| WiFi mesh | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ |
| Onion routing | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Clearnet exit | ❌ | ✅ | ✅ | ⚠️ | ❌ | ❌ | ❌ | ✅ |
| Anonymity | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| No ISP needed | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ |
| Public key addressing | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Decentralized discovery | ✅ | ❌ | ❌ | ✅ | ⚠️ | ✅ | ❌ | ✅ |
| Pool trust domains | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Router firmware | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Hardware ecosystem | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Inter-network routing | ❌ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ | ❌ | ✅ |
| ECH at exit | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## BGP-X's Unique Position

BGP-X does not claim to be the first to implement any individual feature. It claims to be the first to **unify all of these features in a single coherent system with hardware targets**.

**What BGP-X innovates**:
- Pool-based trust domains with double-exit architecture
- Clearnet exit model with signed exit policies, ECH, DoH, DNSSEC
- Unified DHT spanning mesh and internet transports
- Router-level deployment (all devices protected transparently)
- Hardware platform with unified core and deployment-specific variants

**What BGP-X builds on**:
- cjdns's insight that public key addressing is superior to IP addressing
- Reticulum's validation of multi-transport mesh for low-bandwidth links
- Tor's multi-hop onion routing model and circuit design
- Meshtastic's demonstration of LoRa mesh hardware viability
- NYC Mesh / Freifunk's demonstration of community WiFi mesh deployment
- Kademlia DHT for decentralized discovery
- Noise Protocol for the handshake model

---

## How BGP-X Works With Existing Systems

BGP-X does not replace existing community deployments. It can layer on top of them:

### NYC Mesh / Freifunk / LibreMesh Integration

These communities run OpenWrt routers — the same hardware BGP-X targets.

**Integration path**:
1. Install BGP-X as an OpenWrt package (no hardware changes)
2. Existing mesh routing continues unchanged
3. BGP-X adds a privacy overlay on top of existing connectivity
4. Community members' traffic is now onion-encrypted through the overlay
5. DHT pool configuration enables federation with other BGP-X communities

**What changes for community members**: traffic is now privately routed. The mesh itself is unchanged — BGP-X runs as additional software on the same routers.

**What community gets**: privacy protection for all member traffic; federation with other BGP-X communities; coverage gap bridging via internet relay pools.

### Meshtastic Integration

Meshtastic devices (ESP32/nRF52840) cannot run the full BGP-X daemon (too constrained). Integration via adaptation layer:

**Integration path**:
1. Add a Raspberry Pi or GL.iNet router near existing Meshtastic infrastructure
2. Connect Meshtastic device to router via USB
3. BGP-X adaptation layer uses Meshtastic device as raw LoRa radio modem
4. BGP-X handles all routing, encryption, and DHT via LoRa radio
5. Meshtastic protocol is bypassed for BGP-X traffic

**Hardware reuse**: same Meshtastic antennas and radios. New firmware for ESP32 that exposes raw LoRa mode for BGP-X. Meshtastic protocol can still coexist for devices not running BGP-X.

### Helium Infrastructure Reuse

Helium hotspots contain LoRaWAN radio hardware that could be repurposed. With modified firmware:

1. Helium hotspot hardware (RAK concentrator + gateway SBC) runs BGP-X daemon
2. LoRa radio serves BGP-X mesh transport instead of (or in addition to) LoRaWAN
3. Helium coverage becomes BGP-X mesh coverage

Note: Helium's blockchain-based incentive model is separate from BGP-X. Any economic incentive layer for BGP-X node operators is out of scope for the base protocol.

---

## The Coverage Gap Bridging Solution

The most significant architectural capability BGP-X provides that no existing system offers: **automatic, transparent bridging between geographically separated mesh islands via internet relay pools**.

### The Problem

Two communities each have functional mesh networks but cannot directly connect because they're physically separated (different cities, different countries, too far for radio).

With existing systems: no connection. The communities are isolated.

### The BGP-X Solution

Each community deploys a gateway node (Mode 5) connecting their mesh to the internet. Each community configures a pool for their mesh nodes.

```
Community A (Lima) → [Pool: mesh-lima] → Gateway Lima → Internet relay pool
                                                                │
Community B (Bogotá) ← [Pool: mesh-bogota] ← Gateway Bogotá ←─┘
```

Gateway DHT cross-domain sync makes each community's nodes appear in the other's DHT. Path construction automatically uses the three-segment path across all pools.

**What the user experiences**: connecting to a peer in the other community works just like connecting to a local mesh node. No manual configuration. No awareness of the internet bridge.

**Privacy**: all traffic is onion-encrypted throughout. The internet relay nodes see only encrypted BGP-X traffic. The gateways see which segment they're connecting (but not the full path).

---

## Honest Competitive Positioning

BGP-X should not overclaim. The honest positioning:

**BGP-X IS**:
- The first system to unify: onion routing + mesh transport + clearnet exit + decentralized DHT + pool trust domains + hardware ecosystem
- The first to enable transparent coverage gap bridging between separated mesh islands
- The first with router-level deployment (all devices protected without per-device configuration)
- The first with ECH at exit nodes for domain name privacy
- The first with pool-based trust domains enabling double-exit architecture

**BGP-X is NOT**:
- The first mesh network
- The first anonymous communication system
- The first to use public key addressing
- The first to support LoRa transport
- The first to provide onion routing

**BGP-X builds on** decades of prior work by the mesh networking, anonymity, and cryptographic networking communities. The goal is to complete the picture that prior systems have each partially drawn.
