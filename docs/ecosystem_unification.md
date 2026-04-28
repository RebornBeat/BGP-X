# BGP-X: Ecosystem Unification Through Domain Routing

**Version**: 0.1.0-draft

---

## 1. The Fragmentation Problem

The current privacy and mesh networking ecosystem is fragmented:

- **Tor** provides internet anonymity but cannot reach mesh island services
- **Reticulum** provides mesh networking but has no onion-routing privacy model
- **NYC Mesh / Freifunk** provide community internet access but isolated from other communities
- **Meshtastic** provides LoRa mesh communication but disconnected from internet services
- **cjdns / Yggdrasil** provide encrypted routing but no source anonymity

Each system works in isolation. A Tor user cannot reach a Meshtastic-connected service. A Freifunk community in Berlin cannot communicate privately with one in Lima. A Reticulum mesh island cannot route through the Tor/I2P overlay for additional privacy.

BGP-X's unification is the **routing domain model** that makes all of these first-class participants in one network.

---

## 2. The Unification: Routing Domains

Every BGP-X routing domain is a first-class participant in a single unified topology:

1. **No domain is secondary**: clearnet, overlay, and mesh are equals
2. **Any domain can originate a path**: clearnet client, mesh client, satellite client — same routing algorithm
3. **Any domain can be a destination**: services in any domain reachable from any other
4. **Any combination** of domains can be intermediate hops — no ordering enforced
5. **N-hop unlimited across all domains** — no protocol maximum at any level
6. **One unified DHT**: all domains, one discovery layer, one consistent view

This is ecosystem unification — not "these systems coexist" but "these systems are one system."

---

## 3. What Each Prior System Contributes

BGP-X inherits and extends:

- **cjdns**: public key addressing → BGP-X NodeID model
- **Yggdrasil**: DHT-based routing → BGP-X unified Kademlia DHT
- **Reticulum**: multi-transport mesh → BGP-X mesh transport abstraction
- **Tor**: onion routing → BGP-X onion encryption (domain-agnostic)
- **I2P**: garlic routing and native service model → BGP-X SDK native services
- **Meshtastic**: LoRa community mesh → BGP-X mesh transport + adaptation layer
- **NYC Mesh / Freifunk**: community WiFi mesh → BGP-X OpenWrt deployment

**BGP-X adds**: the domain routing layer that connects all of these into one coherent topology. The DOMAIN_BRIDGE hop type. The unified DHT spanning all domains. The domain bridge node model. The cross-domain path_id return routing. The N-hop unlimited protocol design.

---

## 4. How BGP-X Works With Existing Systems

### NYC Mesh / Freifunk / LibreMesh Integration

BGP-X deploys as an OpenWrt package on community mesh hardware. The community's existing WiFi mesh continues to provide local connectivity. BGP-X adds:
- Onion routing for metadata privacy
- Cross-domain connectivity to other BGP-X mesh islands worldwide
- Clearnet exit via pool-selected exit nodes
- Unified DHT for service discovery across communities

When a community installs BGP-X with a bridge node, the coverage gap between communities is bridged via the clearnet overlay transparently. Community A in Lima and Community B in Bogotá can communicate privately through BGP-X paths that traverse clearnet between their gateways.

### Meshtastic Integration

Meshtastic hardware (ESP32 + LoRa radio) serves as a LoRa radio modem for BGP-X nodes via an adaptation layer. The Meshtastic routing protocol is replaced by BGP-X's protocol. The result: LoRa radio infrastructure running a full privacy overlay, discoverable from the unified DHT, accessible from clearnet BGP-X clients without any LoRa hardware.

### Inter-Community Routing

Previously: communities using different mesh systems (Meshtastic, NYC Mesh, Freifunk) could not communicate privately with each other.

With BGP-X domain routing: each community runs BGP-X with its own `island_id`. Domain bridge nodes connect each community to the BGP-X clearnet overlay. Through the unified DHT, communities discover each other. Cross-domain paths connect them through the overlay.

The coverage gap between communities — previously an absolute barrier — becomes a clearnet overlay segment in a cross-domain path.

---

## 5. The Coverage Gap Bridging Solution

```
Community A (Lima)           [Clearnet Gap]           Community B (Bogotá)
mesh:lima-district-1    ─────────────────────    mesh:bogota-district-1
      │                                                      │
   Bridge A                BGP-X overlay                 Bridge B
   (gateway)            (internet-connected             (gateway)
                           BGP-X relays)

Cross-domain path:
  mesh:lima → Bridge A → clearnet overlay → Bridge B → mesh:bogota
```

The unified DHT makes both islands discoverable from each other via their bridge nodes. The path is constructed automatically by the routing algorithm.

---

## 6. Honest Competitive Positioning

**BGP-X IS** the first to:
- Unify onion routing + mesh transport + clearnet exit + decentralized unified DHT + pool trust domains + hardware ecosystem into one coherent system
- Enable any-to-any cross-domain routing (clearnet ↔ mesh ↔ satellite in any combination)
- Enable clearnet clients to reach mesh island services without any special hardware
- Implement N-hop unlimited path construction with no protocol maximum
- Operate a unified DHT spanning all routing domains

**BGP-X is NOT** the first:
- Mesh network (Reticulum, Meshtastic predate it)
- Anonymous communication system (Tor, I2P, cjdns predate it)
- To use public key addressing (cjdns pioneered this)
- To support LoRa transport (Reticulum, Meshtastic predate it)
- To provide onion routing (Tor, I2P predate it)

BGP-X's value is in synthesis and unification, not in any single primitive.
