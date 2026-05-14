# BGP-X Compared to Existing Privacy and Mesh Networks

**Version**: 0.1.0-draft

This document provides a detailed technical comparison of BGP-X against the most relevant existing systems.

---

## 1. Summary Comparison Table

| Property | Tor | VPN | I2P | cjdns | Yggdrasil | Reticulum | SCION | BGP-X |
|---|---|---|---|---|---|---|---|---|
| Multi-hop routing | ✅ Fixed 3 | ❌ 1 hop | ✅ Variable | ❌ None | ❌ None | ❌ None | ✅ Path-aware | ✅ N-hop unlimited |
| Onion routing | ✅ Yes | ❌ No | ✅ Garlic | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Router-level operation | ❌ No | ✅ Yes | ❌ No | Partial | Partial | ❌ No | ❌ No | ✅ Yes |
| Decentralized discovery | ❌ Directory auths | ❌ Provider | ✅ Netdb | ✅ Partial | ✅ Yes | ✅ Yes | ⚠️ ISD model | ✅ Unified DHT |
| Client-selected paths | ⚠️ Partial | ❌ None | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Full |
| Clearnet exit model | ✅ Exit nodes | ✅ All traffic | ⚠️ Outproxies | ❌ No | ❌ No | ❌ No | ⚠️ Gateway-dep | ✅ Signed gateways |
| Mesh transport native | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Partial | ✅ Yes | ❌ No | ✅ Yes (WiFi/LoRa/BLE) |
| LoRa transport | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| Pool-based trust domains | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Double-exit architecture | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Cross-domain routing | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| .bgpx / .onion native services | ✅ .onion | ❌ No | ✅ .i2p | ❌ No | ❌ No | ❌ No | ❌ No | ✅ .bgpx |
| Configurable path length | ❌ Hardcoded 3 | ❌ N/A | ⚠️ Limited | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| UDP-native transport | ❌ TCP only | ⚠️ Varies | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| Network-layer (TUN) | ❌ SOCKS5 proxy | ✅ Yes | ⚠️ Limited | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes |
| Stream multiplexing | ❌ Per-circuit | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Native |
| ECH at exit | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Router firmware support | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Hardware ecosystem | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Embedded | ❌ No | ✅ Full ecosystem |
| No central authority | ❌ Directory auths | ❌ Provider | ✅ Yes | ⚠️ Partial | ✅ Yes | ✅ Yes | ⚠️ ISD model | ✅ Yes |
| Forward secrecy | ✅ Yes | ⚠️ Impl-dep | ⚠️ Partial | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| Requires ISP cooperation | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes | ❌ No |
| ISP-free operation | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | ✅ Mesh modes |
| HTTP/2 native | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes (for .bgpx) |
| Satellite clarification | N/A | N/A | N/A | N/A | N/A | N/A | N/A | ✅ Sat internet = clearnet |

---

## 2. Hardware Ecosystem Comparison

| System | Dedicated Hardware | Router Firmware | Mesh Hardware | Hardware Cost |
|---|---|---|---|---|
| Tor | ❌ No | ❌ (complex) | ❌ No | N/A |
| VPN | ❌ No | ⚠️ Limited | ❌ No | Varies |
| I2P | ❌ No | ❌ No | ❌ No | N/A |
| cjdns | ❌ No | ⚠️ Partial | ❌ No | N/A |
| Reticulum | ❌ No | ❌ No | ⚠️ ESP32 | $20-40 (ESP32) |
| BGP-X | ✅ Router v1, Node v1, Gateway v1, Client Node, Adapter | ✅ OpenWrt native | ✅ LoRa + WiFi | $25-150 |

**BGP-X Hardware Products**:

| Product | Target User | Key Characteristics |
|---|---|---|
| **BGP-X Router v1** | End user | All-in-one; replaces home router; WiFi + LoRa + BLE; outdoor IP67 option; satellite WAN via USB |
| **BGP-X Node v1** | Community contributor | Compact outdoor node; solar-powered option; runs complete bgpx-node daemon; domain bridge capable |
| **BGP-X Gateway v1** | Provider | High-throughput exit node; SFP+ uplink; 1 GB RAM; datacenter deployment |
| **BGP-X Client Node** | Individual endpoint | Low-cost (LILYGO T3 based); battery-powered; personal connectivity; no routing for others |
| **BGP-X Adapter/Dongle** | USB LoRa modem | USB-powered; acts as LoRa interface for existing computers |

**Key clarification**: BGP-X Node v1 runs the **complete** bgpx-node daemon — the same software as BGP-X Router v1. The difference is form factor and deployment context, not capability. It is NOT a "lite" or partial device.

No other system offers dedicated hardware with native LoRa + WiFi mesh integration at the router level.

---

## 3. BGP-X vs. Tor

### Where They Agree

Both BGP-X and Tor are built on the same foundational insight: **no single node should see both the origin and the destination of a connection**. Both use layered encryption (onion routing) to enforce this. Both provide forward secrecy. Both are open-source and designed to resist surveillance.

### Where BGP-X Goes Further

#### 3.1 Decentralized Discovery (No Directory Authorities)

**Tor**: Uses a small set of hard-coded directory authorities (currently ~9) that collectively publish a signed consensus document describing all active relays. This creates structural centralization:

- The directory authorities are known, fixed targets for legal coercion, DDoS, and subversion
- If a majority of directory authorities are compromised, the relay list can be manipulated
- Directory authorities have geographic and jurisdictional concentration

**BGP-X**: Uses a **fully decentralized Unified DHT** spanning all routing domains. Every node publishes a self-signed advertisement. There is no consensus document and no directory authority. A client that discovers even one DHT node can find all others through the DHT's routing algorithm. There is no single entity to coerce, subpoena, or DDoS to degrade the network's anonymity set.

#### 3.2 Variable Path Length (N-Hop Unlimited)

**Tor**: All circuits are exactly 3 hops. This is a deliberate design choice to balance anonymity and latency. However, it means all users have identical path structure, which can aid traffic analysis. There is no mechanism for a high-risk user to use a 5-hop path without modifying Tor internals.

**BGP-X**: Path length is a first-class configuration parameter. Default is 4 hops. **No global enforcement maximum.** Applications choose based on threat model. High-security use cases use 6-7 hops with geographic diversity enforcement. N-hop unlimited applies across all routing domains in any combination.

#### 3.3 Pool-Based Trust

**Tor**: No equivalent concept. All relays come from one directory-authority-controlled pool. Users cannot specify that entry and exit nodes must come from different trust domains.

**BGP-X**: Named pools with curator signatures enable trust isolation. Multi-segment paths use different pools per segment. **Double-exit architecture** uses two independent exit pools in sequence — no single exit operator sees both path origin and destination.

#### 3.4 Transport Layer

**Tor**: TCP only between relays. Cannot natively carry UDP traffic (DNS queries, VoIP, gaming). TCP connections between relays carry identifiable fingerprints.

**BGP-X**: **UDP native.** Carries all IP protocols — TCP, UDP, ICMP. Custom reliability layer for streams needing ordered delivery. No TCP fingerprinting of the relay protocol. Mesh transport (LoRa, WiFi 802.11s, BLE) in addition to UDP.

#### 3.5 Network Integration

**Tor**: SOCKS5 proxy. Applications must be explicitly configured. System daemons, command-line tools, and SOCKS5-unaware apps require complex workarounds (iptables, torsocks, Whonix).

**BGP-X**: **TUN interface.** The operating system routes IP traffic into this interface. Any application — including those with no network privacy awareness — routes through BGP-X transparently. Same model as a VPN, but with multi-hop privacy properties.

#### 3.6 ECH / SNI Protection

**Tor**: No exit-level ECH support. TLS ClientHello SNI is visible to exit nodes and network observers.

**BGP-X**: Exit nodes query DNS HTTPS records for ECH configuration. When available, TLS ClientHello uses ECH, hiding the domain name from SNI. Even the exit node cannot see the domain name from the ClientHello.

#### 3.7 Mesh Transport

**Tor**: Internet-only. No radio or mesh transport. Requires ISP connectivity.

**BGP-X**: WiFi 802.11s, LoRa, Bluetooth BLE, Ethernet P2P. Communities can operate BGP-X mesh networks with **zero ISP involvement** for intra-mesh traffic. Coverage gaps between separated mesh islands are bridged via internet relay pools transparently and privately.

#### 3.8 Cross-Domain Routing

**Tor**: Single routing domain (internet). Cannot route to mesh networks, LoRa communities, or non-internet destinations.

**BGP-X**: Three equal routing domains (clearnet, overlay, mesh). Any-to-any routing. A clearnet client with no mesh hardware can reach services on a LoRa mesh island via domain bridge nodes.

#### 3.9 .bgpx Native Services

**Tor**: .onion addresses (56 characters base32). No human-readable name resolution.

**BGP-X**: .bgpx addresses (64 hex characters) with **Name Registry DHT** for human-readable short names (e.g., `community-news.bgpx`). Self-authenticating via Ed25519 public key. Accessible from clearnet, mesh, and overlay without any special client hardware.

#### 3.10 Hardware Ecosystem

**Tor**: Software only. No dedicated hardware.

**BGP-X**: Full hardware ecosystem: BGP-X Router v1, Node v1, Gateway v1, Client Node, Adapter/Dongle. Router-level deployment is a first-class target.

---

## 4. BGP-X vs. VPN

### The Fundamental VPN Problem

A VPN provides one hop of trust displacement. Instead of your ISP seeing your traffic, your VPN provider sees it. The VPN provider knows:

- Your real IP address
- Every destination you connect to
- Timing and volume of all your traffic
- Plaintext content (for non-HTTPS connections)

If the VPN provider is compromised, subpoenaed, or malicious, all of that data is available to the adversary. No amount of "no-log policy" changes the fundamental architecture.

### BGP-X's Approach

BGP-X distributes trust across **N independent nodes**. No single node has the complete picture. To reconstruct the source-destination link, an adversary must control multiple nodes simultaneously **and** they must be in the same path **and** they must be both entry and exit of that path. Pool-based operator diversity enforcement prevents this.

BGP-X node operators have no business relationship with users. No payment data, no account, no email linking a user to their BGP-X activity.

### Pool-Based Exit Control

VPNs offer no equivalent to BGP-X's pool-based routing. With BGP-X private pools, you can route exit traffic through infrastructure you control — with the path segments before the exit coming from the public pool (so even your own exit node doesn't see the origin). **Double-exit architecture** is not possible with a VPN.

### When a VPN Is Still Appropriate

VPNs are appropriate when:
- You fully trust the VPN provider
- Your threat model is ISP surveillance only
- You need consistent exit IP (e.g., for geo-restriction bypass)
- Latency is more important than strong privacy guarantees

BGP-X is appropriate when:
- You need privacy not dependent on trusting any single provider
- Your threat model includes legal coercion of service providers
- You are building infrastructure that must be resilient to targeted observation
- You need network-layer privacy for all applications, not just browsers
- You want ISP-free operation for mesh communities

---

## 5. BGP-X vs. I2P

### Similarities

Both BGP-X and I2P use multi-hop routing with anonymity properties. Both support native services (BGP-X native services, I2P eepsites). Both are fully decentralized.

### Differences

**Clearnet access**: I2P is primarily an internal network; clearnet outproxies are limited and not a primary use case. BGP-X is designed for full clearnet access via signed gateways as a primary use case.

**Transport**: I2P uses NTCP2 (TCP) and SSU2 (UDP). BGP-X uses UDP natively with custom reliability layer; mesh transports (LoRa, WiFi) for ISP-free operation.

**Pool trust**: I2P has no equivalent to BGP-X's pool-based trust domains and double-exit architecture.

**ECH**: I2P has no exit-level ECH support.

**Hardware ecosystem**: I2P requires a JVM (high resource requirements). BGP-X is implemented in Rust and designed for embedded/router deployment.

**Cross-domain routing**: I2P operates within a single logical network. BGP-X explicitly routes across multiple routing domains (clearnet ↔ overlay ↔ mesh).

**.bgpx vs .i2p**: Both have native service addressing. BGP-X adds human-readable Name Registry DHT and cross-domain accessibility (clearnet clients can reach mesh island .bgpx services).

---

## 6. BGP-X vs. cjdns / Yggdrasil

### What cjdns / Yggdrasil Do

cjdns (Hyperboria) and Yggdrasil provide public key addressing for overlay networks. Like BGP-X, they use cryptographic identities and work over existing internet infrastructure or local networks.

### Key Differences

**No onion routing**: cjdns/Yggdrasil use point-to-point encrypted links between adjacent nodes. No multi-hop anonymity — any node on the path can see both source and destination.

**No anonymity**: cjdns is designed for connectivity (a mesh internet), not privacy. The design does not attempt to hide communication patterns.

**No clearnet exit model**: cjdns is a separate overlay network. It does not provide access to clearnet internet destinations via exit nodes.

**No pools**: No trust tier mechanism.

**No hardware ecosystem**: Software only.

**BGP-X position**: BGP-X builds on cjdns's insight that public key addressing is a better model than IP addressing, and adds multi-hop onion routing, clearnet exit, pool trust domains, cross-domain routing, and hardware targets on top.

---

## 7. BGP-X vs. Reticulum

### What Reticulum Does

Reticulum Network Stack provides cryptographic mesh networking designed for low-bandwidth links. It works across multiple transports (LoRa, WiFi, serial) and is designed for resilient communication.

### Key Differences

**No onion routing**: Reticulum uses point-to-point encrypted links. Each node on a path can observe message routing. No anonymity from relays.

**No clearnet exit model**: Reticulum is for peer-to-peer communication within the network; not designed for clearnet internet access via exit nodes.

**No pools**: No trust tier mechanism.

**No cross-domain routing**: Operates within the mesh; no bridging to other routing domains as distinct entities.

**BGP-X position**: BGP-X validates Reticulum's approach to multi-transport mesh (especially LoRa for long-range low-bandwidth use), and adds the missing onion routing, clearnet exit, pool trust domains, cross-domain routing, unified DHT, and hardware ecosystem.

---

## 8. BGP-X vs. SCION

### What SCION Does

SCION (Scalability, Control, and Isolation On Next-generation networks) provides path-aware networking at the infrastructure level. It requires ISP adoption. Offers strong PKI-based trust, path selection control, and AS isolation.

### The Fundamental Difference

**SCION requires ISP cooperation.** It is a next-generation internet architecture that ISPs must deploy. Currently limited to academic and research networks.

**BGP-X requires no infrastructure changes.** It runs on top of the existing internet as an overlay. Any user with an internet connection can use BGP-X today. Any operator with a server can run a BGP-X node today.

### Complementarity

BGP-X and SCION are not competitors. If SCION becomes widely deployed, BGP-X can use SCION paths as its transport layer (Level 0 in BGP-X's stack), gaining SCION's path-integrity properties while retaining BGP-X's privacy overlay properties above it.

---

## 9. Integration With Existing Community Networks

### NYC Mesh / Freifunk / LibreMesh

These communities run OpenWrt routers — the same hardware BGP-X targets.

**Integration path**:
1. Install BGP-X as an OpenWrt package on existing routers:
   ```bash
   opkg update
   opkg install bgpx-node bgpx-cli bgpx-luci
   ```
2. Configure island_id matching the community's identifier:
   ```toml
   [[routing_domains]]
   domain_type = "mesh"
   island_id = "nyc-mesh-community"
   transports = ["wifi-mesh"]
   ```
3. Existing mesh routing continues unchanged
4. BGP-X adds privacy overlay on top of existing connectivity
5. Deploy one or two bridge nodes (BGP-X Router v1 or Node v1 with WAN) to connect community to global BGP-X network

**What changes for community members**: Traffic is now privately routed through BGP-X overlay. The mesh itself is unchanged.

**What community gains**:
- Privacy for all member traffic
- Federation with other BGP-X communities worldwide
- Coverage gap bridging via internet relay pools

### Meshtastic Communities

Meshtastic devices (ESP32/nRF52840) cannot run full bgpx-node daemon (too constrained). Integration via adaptation layer:

**Integration path**:
1. Add a BGP-X Router v1 or Raspberry Pi running bgpx-node near existing Meshtastic infrastructure
2. Connect Meshtastic device via USB
3. Flash BGP-X modem firmware to Meshtastic device (replaces Meshtastic routing with raw LoRa modem mode)
4. BGP-X node uses Meshtastic device as LoRa radio
5. BGP-X handles all routing, encryption, and DHT

**Hardware reuse**: Same Meshtastic antennas and radios. Same physical infrastructure.

**What Meshtastic community gains**:
- Full onion encryption over existing LoRa infrastructure
- Clearnet access via BGP-X gateway nodes
- Cross-island reach via unified DHT
- Access from clearnet BGP-X users without LoRa hardware

### Reticulum Users

Reticulum provides multi-transport mesh but lacks onion routing privacy model.

**Integration path**:
- BGP-X deploys on same Linux hardware running Reticulum
- Both daemons operate independently (different protocols)
- BGP-X provides the missing privacy layer
- Future: BGP-X transport driver for Reticulum's transport abstraction (interop research area)

---

## 10. Coverage Gap Bridging — Visual

### The Problem

Two communities each have functional mesh networks but cannot directly connect — physically separated (different cities, countries, too far for radio).

With existing systems: no connection. Communities are isolated.

### The BGP-X Solution

```
Community A (Lima)                    Community B (Bogotá)
mesh:lima-district-1                  mesh:bogota-district-1
         │                                      │
    [Bridge A]                           [Bridge B]
    (gateway with                         (gateway with
     Starlink WAN)                         fiber WAN)
         │                                      │
         └────────── BGP-X Clearnet ───────────┘
                     Overlay
                (internet relays)
```

**Path example**:
```
Mesh client in Lima
    → LoRa mesh hops
    → Bridge A (domain transition: mesh → clearnet)
    → Clearnet overlay relays (4 hops)
    → Bridge B (domain transition: clearnet → mesh)
    → WiFi mesh hops
    → Service in Bogotá
```

**Privacy**: All traffic onion-encrypted throughout. Internet relays see only encrypted BGP-X packets. Bridges see only adjacent domain peers.

**What user experiences**: Connecting to a peer in the other community works like connecting locally. No manual configuration. No awareness of internet bridge.

---

## 11. Property Deep Dives

### 11.1 Timing Correlation Resistance

All low-latency anonymity networks share the fundamental challenge of timing correlation. BGP-X mitigations:

- **Variable path length**: More hops = more timing disruption.
- **Geographic diversity**: Physical distance creates timing ambiguity.
- **Stream multiplexing**: Multiple streams share path; per-stream timing is ambiguous.
- **Cover traffic**: COVER packets use session_key; externally identical to RELAY; disrupts timing fingerprints.
- **KEEPALIVE randomization**: ±5 seconds; prevents keepalive timing fingerprinting.

**Residual risk**: A global passive adversary with simultaneous observation of entry and exit can attempt statistical correlation. This is acknowledged as an unsolved problem for all low-latency anonymous networks.

### 11.2 Pool Trust Model

BGP-X's pool system has no equivalent in any compared system. Key properties:

- **Trust is NOT transitive across pools** — each node is individually verified regardless of pool membership.
- Pool curators sign membership records; curator keys are separate from node identity keys.
- Pool curator key rotation uses dual-signature records (old key + new key both sign); 24hr acceptance window.
- Emergency rotation (compromise_confirmed) uses 2hr window.
- Private pools enable fully controlled exit infrastructure with zero public membership disclosure.
- Cross-pool diversity enforcement prevents single operator from controlling entry + exit across segments.

### 11.3 ECH / SNI Protection

Standard TLS ClientHello includes Server Name Indication (SNI) — the domain name in plaintext, visible to any network observer and to the exit node itself.

**BGP-X Solution**: Exit nodes support ECH (Encrypted Client Hello) when the destination publishes ECH configuration in its DNS HTTPS record. The exit node queries for this configuration via DoH (with DNSSEC validation and ECS stripping). When ECH is available, the TLS ClientHello is constructed with the inner ClientHello (containing the real SNI) encrypted to the server's ECH public key.

**Result**: Exit node sees only the outer ClientHello with a generic SNI; real domain is encrypted. Network observers between exit and destination cannot see the domain name.

This is available in no other compared system at the exit node level.

### 11.4 Exit Node Trust

**Tor**: Any volunteer can run an exit node. No vetting, no signed exit policy at the protocol level. Exit policies are advisory.

**BGP-X**: Exit nodes publish **signed exit policies** specifying what traffic they will forward, logging policy (none/metadata/full), operator contact, and jurisdiction. Exit policies include DNS resolver configuration (DoH required), DNSSEC validation, ECS stripping, and ECH capability. Clients filter gateways by policy before path construction. Policy violations are reportable and affect reputation.

**VPNs**: Operator publishes policy; no cryptographic enforcement at the protocol level.

### 11.5 Mesh and ISP-Free Operation

Neither Tor, VPN, I2P, SCION, nor any other compared system provides ISP-free operation via mesh transport.

**BGP-X**: Mesh modes enable communities to operate fully without ISP connectivity for intra-mesh traffic. Coverage gaps between separated mesh islands are bridged via internet relay pools (transparent to users, privately via BGP-X onion encryption). The pool-based DHT creates a unified global discovery layer spanning both mesh and internet nodes.

This enables BGP-X to serve communities without infrastructure — rural communities, disaster recovery scenarios, regions with censored internet, activist networks.

---

## 12. What BGP-X Innovates

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
11. **.bgpx native services** accessible from clearnet, mesh, and overlay
12. **Name Registry DHT** — decentralized short name resolution
13. **HTTP/2 native transport** — optimized for LoRa latency with multiplexed streams
14. **Hardware ecosystem** — Router, Node, Gateway, Client Node, Adapter as unified platform
15. **Satellite as clearnet** — explicit architectural clarity: commercial satellite internet is clearnet domain 0x00000001; domain 0x00000005 reserved for future BGP-X-native satellite network

---

## 13. What BGP-X Builds On

BGP-X integrates concepts from multiple prior systems:

- **cjdns**: inspired public key addressing and NodeID-as-address model
- **Yggdrasil**: informed DHT routing design
- **Reticulum**: validated multi-transport approach for low-bandwidth mesh
- **Tor**: foundational onion routing and circuit model
- **I2P**: garlic routing and native service model
- **Kademlia DHT**: node discovery algorithm
- **Meshtastic**: demonstrated LoRa mesh viability for community networks

BGP-X's innovation is **unification**: combining all of these into one coherent system with inter-protocol domain routing, hardware targets, pool-based trust domains, router-level deployment, and no protocol-level restrictions on path length, domain count, or domain ordering.

---

## 14. Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal:

- If a node declares a jurisdiction: geo plausibility scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply
- Nodes are NOT required to declare jurisdiction
- Declaring jurisdiction is an opt-in privacy/convenience tradeoff

Satellite-connected nodes are **exempt** from geo plausibility scoring because satellite terminal IPs may be assigned from distant ground stations.

---

## 15. HTTP/2 for .bgpx Services

BGP-X native services (.bgpx addresses) use **HTTP/2 over BGP-X streams**.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at exit nodes when connecting to HTTP/3 clearnet servers — standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 16. Satellite Internet Clarification

**Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) are clearnet.**

They provide BGP-routed IP connectivity. From BGP-X's perspective:
- **Domain**: Clearnet (type 0x00000001) — same as fiber, cellular, or cable
- **Latency class**: Satellite-specific (LEO: 20-60ms, GEO: 600ms+)
- **Regulatory treatment**: Same as any other internet WAN from a BGP-X protocol perspective

**Domain type 0x00000005 (bgpx-satellite)** is **RESERVED** for a future BGP-X-native satellite network where satellites themselves run BGP-X relay software and communicate via inter-satellite links. This is NOT current commercial satellite internet. Domain type 0x00000005 is reserved and not active in current deployments.
