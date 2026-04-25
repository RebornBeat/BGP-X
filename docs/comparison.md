# BGP-X Compared to Existing Privacy and Mesh Networks

This document provides a detailed technical comparison of BGP-X against the most relevant existing systems.

---

## Summary Table

| Property | BGP-X | Tor | VPN | I2P | cjdns | Reticulum | SCION |
|---|---|---|---|---|---|---|---|
| Multi-hop routing | ✅ N hops | ✅ Fixed 3 | ❌ 1 hop | ✅ Variable | ❌ None | ❌ None | ✅ Path-aware |
| Onion routing | ✅ Yes | ✅ Yes | ❌ No | ✅ Garlic | ❌ No | ❌ No | ❌ No |
| Decentralized discovery | ✅ DHT | ❌ Directory authorities | ❌ Single provider | ✅ Netdb | ✅ Partial | ✅ Yes | ⚠️ ISD model |
| Clearnet exit model | ✅ Signed gateways | ✅ Exit nodes | ✅ All traffic | ⚠️ Outproxies | ❌ No | ❌ No | ⚠️ Gateway-dep |
| Mesh transport native | ✅ Yes | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ❌ No |
| LoRa transport | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes | ❌ No |
| Pool-based trust domains | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No |
| Double-exit architecture | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No |
| Configurable path length | ✅ Yes | ❌ Hardcoded 3 | ❌ N/A | ⚠️ Limited | ❌ No | ❌ No | ✅ Yes |
| UDP-native transport | ✅ Yes | ❌ TCP only | ⚠️ Protocol-dep | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| Network-layer (TUN) | ✅ Yes | ❌ SOCKS5 proxy | ✅ Yes | ⚠️ Limited | ✅ Yes | ❌ No | ✅ Yes |
| Stream multiplexing | ✅ Native | ❌ One circuit/stream | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| ECH at exit | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No |
| Router firmware support | ✅ Yes | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| Hardware ecosystem | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No |
| No central authority | ✅ Yes | ❌ Directory auths | ❌ Provider | ✅ Yes | ⚠️ Partial | ✅ Yes | ⚠️ ISD model |
| Forward secrecy | ✅ Yes | ✅ Yes | ⚠️ Impl-dep | ⚠️ Partial | ❌ No | ✅ Yes | ✅ Yes |
| Requires ISP cooperation | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| ISP-free operation | ✅ Mesh modes | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ❌ No |

---

## BGP-X vs. Tor

### Where they agree

Both BGP-X and Tor are built on the same foundational insight: no single node should see both origin and destination. Both use layered onion encryption. Both provide forward secrecy. Both are open-source.

### Decentralized Discovery

**Tor**: Hard-coded directory authorities (~9 servers) publish a signed consensus of all relays. Known fixed targets for legal coercion, DDoS, and subversion. Geographic and jurisdictional concentration.

**BGP-X**: Fully decentralized DHT. No consensus document, no directory authority. Clients verify all advertisements cryptographically. Even if every bootstrap node fails, the DHT continues operating via existing routing tables.

### Path Length

**Tor**: Fixed 3 hops. All users have identical path structure — aids traffic analysis. No mechanism for a high-risk user to use more hops without modifying Tor internals.

**BGP-X**: Default 4 hops. No global enforcement maximum. Applications choose based on threat model. High-security use cases use 6-7 hops with geographic diversity enforcement.

### Pool-Based Trust

**Tor**: No equivalent concept. All relays come from one directory-authority-controlled pool.

**BGP-X**: Named pools with curator signatures enable trust isolation. Multi-segment paths use different pools per segment. Double-exit architecture uses two independent exit pools in sequence — no single exit operator sees both path origin and destination.

### Transport Layer

**Tor**: TCP only. Cannot natively carry UDP traffic. Inherits TCP fingerprinting.

**BGP-X**: UDP native. Carries all IP protocols. Custom reliability layer for streams that need ordered delivery. No TCP fingerprinting of the relay protocol.

### Network Integration

**Tor**: SOCKS5 proxy. Applications must explicitly support and be configured for SOCKS5. System daemons, command-line tools, and SOCKS5-unaware apps require complex workarounds.

**BGP-X**: TUN interface. Any application — including SOCKS5-unaware — routes transparently through the overlay. Same model as a VPN, but with multi-hop privacy properties.

### ECH

**Tor**: No exit-level ECH support. TLS ClientHello SNI visible to exit nodes and network observers.

**BGP-X**: Exit nodes query DNS HTTPS records for ECH configuration. When available, the TLS ClientHello uses ECH, hiding the domain name from SNI. Even the exit node cannot see the domain name from the ClientHello.

### Mesh Transport

**Tor**: Internet-only. No radio or mesh transport. Requires ISP connectivity.

**BGP-X**: WiFi 802.11s, LoRa, Bluetooth BLE, Ethernet P2P. Communities can operate BGP-X mesh networks with zero ISP involvement for intra-mesh traffic.

---

## BGP-X vs. VPN

### The Fundamental VPN Problem

A VPN provides one hop of trust displacement. The VPN provider knows: your real IP, every destination, timing and volume, and plaintext content for non-HTTPS connections.

If the VPN provider is compromised, subpoenaed, or malicious, all of that data is available to the adversary. No amount of "no-log policy" changes the fundamental architecture.

### BGP-X's Approach

BGP-X distributes trust across N independent nodes. No single node has the complete picture. To reconstruct the source-destination link, an adversary must control both entry and exit of the same path simultaneously — prevented by pool-based operator diversity enforcement.

Additionally, BGP-X node operators have no business relationship with users. No payment data, no account, no email linking a user to their BGP-X activity.

### Pool-Based Exit Control

VPNs offer no equivalent to BGP-X's pool-based routing. With BGP-X private pools, you can route exit traffic through infrastructure you control — with the path segments before the exit coming from the public pool (so even your own exit node doesn't see the origin). VPNs cannot provide this architecture.

### When a VPN Is Still Appropriate

VPNs are appropriate when: you fully trust the VPN provider; your threat model is ISP surveillance only; you need consistent exit IP; latency is more important than strong privacy guarantees.

BGP-X is appropriate when: you need privacy not dependent on trusting any single provider; your threat model includes legal coercion; you need network-layer privacy for all applications; you want ISP-free operation for mesh communities.

---

## BGP-X vs. I2P

### Similarities

Both BGP-X and I2P use multi-hop routing with anonymity properties. Both support native services (BGP-X native services, I2P eepsites). Both are fully decentralized.

### Differences

**Clearnet access**: I2P is primarily an internal network; clearnet outproxies are limited. BGP-X is designed for full clearnet access via signed gateways as a primary use case.

**Transport**: I2P uses NTCP2 (TCP) and SSU2 (UDP). BGP-X uses UDP natively with custom reliability layer; mesh transports (LoRa, WiFi) for ISP-free operation.

**Pool trust**: I2P has no equivalent to BGP-X's pool-based trust domains and double-exit architecture.

**ECH**: I2P has no exit-level ECH support.

**Hardware ecosystem**: I2P requires a JVM (high resource requirements). BGP-X is implemented in Rust and designed for embedded/router deployment.

---

## BGP-X vs. cjdns

### What cjdns Does

cjdns (Hyperboria) provides public key addressing for overlay networks. Like BGP-X, it uses cryptographic identities and works over existing internet infrastructure or local networks.

### Key Differences

**No onion routing**: cjdns uses point-to-point encrypted links between adjacent nodes. No multi-hop anonymity — any node on the path can see both source and destination.

**No anonymity**: cjdns is designed for connectivity (a mesh internet), not privacy. The design does not attempt to hide communication patterns.

**No clearnet exit model**: cjdns is a separate overlay network. It does not provide access to clearnet internet destinations via exit nodes.

**No pools**: no trust tier mechanism.

**No hardware ecosystem**: cjdns is software only.

**BGP-X position**: BGP-X builds on cjdns's insight that public key addressing is a better model than IP addressing, and adds multi-hop onion routing, clearnet exit, pool trust domains, and hardware targets on top.

---

## BGP-X vs. Reticulum

### What Reticulum Does

Reticulum Network Stack provides cryptographic mesh networking designed for low-bandwidth links. It works across multiple transports (LoRa, WiFi, serial, etc.) and is designed for resilient communication.

### Key Differences

**No onion routing**: Reticulum uses point-to-point encrypted links. Each node on a path can observe message routing.

**No anonymity**: communication source and destination visible to relaying nodes.

**No clearnet exit model**: Reticulum is for peer-to-peer communication within the network; not designed for clearnet internet access via exit nodes.

**No pools**: no trust tier mechanism.

**BGP-X position**: BGP-X validates Reticulum's approach to multi-transport mesh (especially LoRa for long-range low-bandwidth use), and adds the missing onion routing, clearnet exit, pool trust domains, and hardware ecosystem.

---

## BGP-X vs. SCION

### What SCION Does

SCION (Scalability, Control, and Isolation On Next-generation networks) provides path-aware networking at the infrastructure level, requiring ISP adoption. Strong PKI-based trust, path selection control, AS isolation.

### The Fundamental Difference

**SCION requires ISP cooperation.** It is a next-generation internet architecture that ISPs must deploy. Currently limited to academic and research networks.

**BGP-X requires no infrastructure changes.** It runs on top of the existing internet as an overlay. Any user with an internet connection can use BGP-X today (once implementation is complete).

### Complementarity

BGP-X and SCION are not competitors. If SCION becomes widely deployed, BGP-X can use SCION paths as its transport layer (Level 0 in BGP-X's stack), gaining SCION's path-integrity properties while retaining BGP-X's privacy overlay properties above it.

---

## Property Deep Dives

### Timing Correlation Resistance

All low-latency anonymity networks share the fundamental challenge of timing correlation. BGP-X mitigations:

- Variable path length (more hops = more timing disruption)
- Geographic diversity (physical distance creates timing ambiguity)
- Stream multiplexing (multiple streams share path; per-stream timing is ambiguous)
- Cover traffic (COVER packets use session_key; externally identical to RELAY; disrupt timing fingerprints)
- KEEPALIVE randomization (±5 seconds; prevents keepalive timing fingerprinting)

Residual risk: a global passive adversary with simultaneous observation of entry and exit can attempt statistical correlation. This is acknowledged as an unsolved problem for all low-latency anonymous networks.

### Pool Trust Model

BGP-X's pool system has no equivalent in any compared system. Key properties:

- Trust is NOT transitive across pools — each node is individually verified regardless of pool membership
- Pool curators sign membership records; curator keys are separate from node identity keys
- Pool curator key rotation uses dual-signature records (old key + new key both sign); 24hr acceptance window
- Emergency rotation (compromise_confirmed) uses 2hr window
- Private pools enable fully controlled exit infrastructure with zero public membership disclosure
- Cross-pool diversity enforcement prevents single operator from controlling entry + exit across segments

### ECH / SNI Protection

Standard TLS ClientHello includes the Server Name Indication (SNI) — the domain name in plaintext, visible to any network observer and to the exit node itself.

BGP-X exit nodes support ECH (Encrypted Client Hello) when the destination publishes ECH configuration in its DNS HTTPS record. The exit node queries for this configuration via DoH (with DNSSEC validation and ECS stripping). When ECH is available, the TLS ClientHello is constructed with the inner ClientHello (containing the real SNI) encrypted to the server's ECH public key.

Result: exit node sees only the outer ClientHello with a generic SNI; real domain is encrypted. Network observers between exit and destination cannot see the domain name.

This is available in no other compared system at the exit node level.

### Exit Node Trust

**Tor**: Any volunteer can run an exit node. No vetting, no signed exit policy at the protocol level.

**BGP-X**: Exit nodes publish signed exit policies specifying what traffic they will forward, logging policy (none/metadata/full), operator contact, and jurisdiction. Exit policies include DNS resolver configuration (DoH required), DNSSEC validation, ECS stripping, and ECH capability. Clients filter gateways by policy before path construction. Policy violations are reportable and affect reputation.

**VPNs**: Operator publishes policy; no cryptographic enforcement at the protocol level.

### Mesh and ISP-Free Operation

Neither Tor, VPN, I2P, SCION, nor any other compared system provides ISP-free operation via mesh transport.

BGP-X mesh modes enable communities to operate fully without ISP connectivity for intra-mesh traffic. Coverage gaps between separated mesh islands are bridged via internet relay pools (transparent to users, privately via BGP-X onion encryption). The pool-based DHT creates a unified global discovery layer spanning both mesh and internet nodes.

This is the unique capability that enables BGP-X to serve communities without infrastructure — rural communities, disaster recovery scenarios, regions with censored internet, activist networks.
