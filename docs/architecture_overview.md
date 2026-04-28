# BGP-X Architecture Overview

**Version**: 0.1.0-draft

---

## 1. What BGP-X Is

BGP-X is a **router-level privacy overlay network and inter-protocol domain router**. It runs on the network router or as a standalone daemon, protecting all connected devices transparently. Applications require no modification.

BGP-X provides:
- **Onion-encrypted routing**: no single node sees both endpoints of a connection
- **Client-selected paths**: the sender constructs the complete path before transmission
- **Cross-domain routing**: paths may span clearnet, overlay, and mesh islands in any combination
- **N-hop unlimited**: no protocol maximum on path length, domain count, or segment count
- **Three equal entry points**: clearnet, BGP-X overlay, and mesh islands are first-class citizens

---

## 2. Three Equal Entry Points

```
CLEARNET                  BGP-X OVERLAY             MESH ISLANDS
(BGP-routed internet)     (onion-encrypted)          (radio transport)
        │                         │                         │
        └─────────────────────────┴─────────────────────────┘
                    Any combination, any order, unlimited hops
                         All reach all. None is secondary.
```

A clearnet client with a BGP-X daemon (no special hardware) can reach a service inside a mesh island via a domain bridge node that handles the radio transmission. A mesh island client can reach clearnet via a gateway. A path can traverse clearnet → mesh → clearnet. The protocol enforces no entry point restriction and no domain ordering.

---

## 3. System Layers

```
┌────────────────────────────────────────────┐
│             Application Layer              │
│    Standard apps + SDK-aware apps          │
├────────────────────────────────────────────┤
│          Routing Policy Engine             │
│    Domain-aware per-flow routing rules     │
├────────────────────────────────────────────┤
│     Inter-Protocol Domain Router           │
│    Cross-domain path construction          │
│    Domain bridge node selection            │
│    N-hop unlimited routing                 │
├────────────────────────────────────────────┤
│         Overlay Routing Layer              │
│    Onion encryption, unified DHT           │
│    Pools, reputation, geo-plausibility     │
├────────────────────────────────────────────┤
│            Gateway Layer                   │
│    Clearnet exit, domain bridges           │
│    ECH support, exit policy enforcement    │
├────────────────────────────────────────────┤
│           Transport Layer                  │
│    UDP/IP, WiFi 802.11s, LoRa, BLE         │
│    Pluggable transport for obfuscation     │
└────────────────────────────────────────────┘
```

---

## 4. Router-Centric Design

BGP-X runs on the home or office router. All connected devices are protected without per-device configuration. The router installs a TUN interface (bgpx0) and captures traffic via routing policy rules.

Applications see a standard network interface. A standard browser, application, or IoT device behind a BGP-X router is protected without any modification.

BGP-X native applications can use the SDK Socket to request specific path properties including cross-domain routing.

---

## 5. Packet Lifecycle (Clearnet)

1. App sends packet → OS routing table → bgpx0 TUN
2. BGP-X daemon reads packet
3. Routing policy matches rule → BGP-X overlay
4. Client selects 4-hop path from default pool
5. Client performs X25519 handshake with each hop
6. Client constructs 4-layer ChaCha20-Poly1305 onion
7. Outer layer sent to entry node via UDP
8. Each relay decrypts one layer, forwards via path_id routing
9. Exit node delivers to clearnet destination
10. Response returns via path_id chain back to client

---

## 6. Packet Lifecycle (Cross-Domain)

1. App sends packet → bgpx0 TUN
2. Routing policy matches cross-domain rule
3. Client discovers bridge nodes for required domain transition
4. Client selects path spanning clearnet + mesh segments
5. Domain-agnostic handshake with all hops including bridge node
6. Client constructs onion with DOMAIN_BRIDGE hop at bridge position
7. Bridge node receives clearnet UDP, decrypts DOMAIN_BRIDGE layer
8. Bridge node forwards remaining onion via mesh radio
9. Mesh relays forward to service
10. Return path: service → mesh relays → bridge (cross-domain path_id lookup) → clearnet relays → client

---

## 7. N-Hop Unlimited

BGP-X imposes no protocol maximum on path length. The common header has no hop counter. No mechanism drops a packet after N hops. The only minimum is 3 hops.

Applications choose path length based on their threat model and latency tolerance. A 20-hop path across 4 routing domains is as valid as a 3-hop single-domain path.

---

## 8. Unified DHT

BGP-X operates one unified Kademlia DHT spanning all routing domains. All nodes — clearnet, mesh island, satellite — participate in the same key space. There is no separate mesh DHT. Domain bridge nodes serve as DHT storage infrastructure accessible from the internet for mesh-only nodes.

Record types: node advertisements, pool advertisements, domain bridge records, mesh island records, key rotation records.

---

## 9. Comparison to Tor

| Property | Tor | BGP-X |
|---|---|---|
| Deployment | Per-application or per-device | Router-level (all devices) |
| Path length | Fixed 3 (circuit) | Unlimited (client-chosen) |
| Entry points | Internet only | Clearnet, overlay, mesh |
| Cross-domain routing | No | Yes — any combination |
| Mesh/radio transport | No | Yes — WiFi, LoRa, BLE |
| N-hop unlimited | No (fixed 3) | Yes |
| Pool-based trust | No | Yes |
| DHT | Distributed directory | Unified Kademlia DHT |
| Hardware target | General-purpose | Router + embedded hardware |
| ECH at exit | No | Yes |
| Cover traffic | Padding only | COVER uses session_key |

---

## 10. Extension Architecture

All extensions are v1 — nothing deferred.

| Extension | Default | Notes |
|---|---|---|
| Cover traffic | Disabled | Enable for enhanced timing resistance |
| Pluggable transport | Disabled | Enable for ISP censorship resistance |
| Stream multiplexing | Enabled | Multiple streams per path |
| Path quality reporting | Enabled | Includes domain ID |
| ECH at exit | Enabled when capable | Hides domain at exit |
| Geographic plausibility | Enabled | Domain-specific thresholds |
| Pool support | Enabled | Including domain-scoped pools |
| Mesh transport | Enabled in mesh firmware | WiFi, LoRa, BLE |
| Cross-domain routing | Enabled | Requires bridge nodes |
| Domain bridge support | Enabled when configured | Requires 2+ routing_domains |
| Mesh island routing | Enabled in mesh firmware | Island DHT registration |
