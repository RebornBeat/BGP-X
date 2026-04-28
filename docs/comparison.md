# BGP-X Compared to Other Systems

**Version**: 0.1.0-draft

---

## 1. Summary Comparison Table

| Property | Tor | I2P | cjdns | Yggdrasil | Reticulum | BGP-X |
|---|---|---|---|---|---|---|
| Onion routing | ✅ | ✅ Garlic | ❌ | ❌ | ❌ | ✅ |
| Router-level | ❌ | ❌ | Partial | Partial | ❌ | ✅ |
| Mesh/radio transport | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Cross-domain routing | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| N-hop unlimited | ❌ (fixed 3) | Partial | N/A | N/A | ❌ | ✅ |
| Pool-based trust | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Hardware target | None | None | Router | Router | Embedded | Router + hardware |
| ECH at exit | ❌ | N/A | N/A | N/A | N/A | ✅ |
| Cover traffic | Padding | Padding | ❌ | ❌ | ❌ | Full COVER msg |
| Unified DHT | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ (all domains) |

---

## 2. BGP-X vs. Tor

**Similarities**: onion routing, layered encryption, no central authority.

**BGP-X differences**:
- Runs at router level (all devices protected; Tor is per-application)
- Client-chosen paths with no fixed hop count (Tor uses fixed 3-hop circuits)
- Pool-based trust model for path selection (Tor uses Directory Authorities)
- Mesh radio transport in addition to internet (Tor: internet only)
- Cross-domain routing: clearnet ↔ overlay ↔ mesh in any combination
- N-hop unlimited (Tor: fixed 3 hops)
- COVER packets use session_key (Tor uses padding only)
- ECH at exit nodes (Tor: no ECH)

---

## 3. BGP-X vs. I2P

**Similarities**: distributed, no central authority, native service support.

**BGP-X differences**:
- Router-level protection (I2P is per-application)
- Clearnet exit capability with exit policy (I2P: internal network primarily)
- Mesh radio support (I2P: internet only)
- Cross-domain routing (I2P: single domain)
- Hardware targets (I2P: general-purpose only)

---

## 4. BGP-X vs. cjdns / Yggdrasil

**Similarities**: public key addressing, DHT routing.

**BGP-X differences**:
- Onion encryption for privacy (cjdns/Yggdrasil: encrypted but not anonymous)
- Source hidden from relays (cjdns: source visible to all hops)
- Pool-based trust model (cjdns: none)
- Mesh radio transport (cjdns/Yggdrasil: internet/Ethernet only)
- Cross-domain routing (neither supports cross-domain)

---

## 5. BGP-X vs. Reticulum

**Similarities**: multi-transport mesh support, LoRa/radio capability.

**BGP-X differences**:
- Onion encryption for metadata privacy (Reticulum: content encrypted but routing metadata visible)
- Router-level internet integration (Reticulum: mesh-focused)
- Cross-domain routing between internet and mesh (Reticulum: mesh only)
- Unified DHT (Reticulum: propagation-based)
- Pool-based path selection (Reticulum: no pools)

---

## 6. BGP-X Unique Properties

These properties do not exist in any prior system:

1. **Any-to-any cross-domain routing**: clearnet ↔ overlay ↔ mesh in any combination
2. **Clearnet clients reaching mesh island services** without any mesh hardware
3. **N-hop unlimited** with no protocol-level maximum at any level
4. **Three equal entry points** — no domain is secondary, none is privileged
5. **Unified DHT spanning all routing domains** — one discovery layer for all
6. **Domain bridge nodes** with standardized DOMAIN_BRIDGE hop type
7. **Pool-based trust with domain scoping** — pools restricted to specific routing domains
8. **Router-level + mesh-level** combined deployment
9. **ECH at exit** combined with mesh transport
10. **Domain-agnostic handshake** — one handshake protocol for all transports
