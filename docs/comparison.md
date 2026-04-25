# BGP-X Compared to Existing Privacy Networks

This document provides a detailed technical comparison of BGP-X against the four most relevant existing systems: Tor, VPN, I2P, and SCION.

---

## Summary Table

| Property | BGP-X | Tor | VPN | I2P | SCION |
|---|---|---|---|---|---|
| Multi-hop routing | ✅ N hops | ✅ Fixed 3 | ❌ 1 hop | ✅ Variable | ✅ Path-aware |
| Decentralized discovery | ✅ DHT | ❌ Directory authorities | ❌ Single provider | ✅ Netdb | ✅ ISD-based |
| Client-selected paths | ✅ Full | ⚠️ Partial (guard pinning) | ❌ None | ✅ Yes | ✅ Yes |
| UDP-native transport | ✅ Yes | ❌ TCP only | ⚠️ Protocol-dependent | ✅ Yes | ✅ Yes |
| Network-layer integration | ✅ TUN interface | ⚠️ SOCKS5 proxy | ✅ TUN interface | ⚠️ Limited | ✅ Yes |
| Stream multiplexing | ✅ Native | ❌ One circuit per stream | ✅ Yes | ✅ Yes | ✅ Yes |
| Configurable path length | ✅ Yes | ❌ Hardcoded 3 | ❌ N/A | ⚠️ Limited | ✅ Yes |
| Router firmware support | ✅ Yes | ❌ No | ✅ Yes | ❌ No | ❌ No |
| Clearnet access | ✅ Via gateways | ✅ Via exit nodes | ✅ All traffic | ⚠️ Limited | ⚠️ Gateway-dependent |
| Native service hosting | ✅ Yes | ✅ Hidden services | ❌ No | ✅ Yes | ✅ Yes |
| Cover traffic support | ✅ Pluggable | ❌ No | ❌ No | ⚠️ Partial | ❌ No |
| No central authority | ✅ Yes | ❌ Directory authorities | ❌ Provider | ✅ Yes | ⚠️ ISD trust model |
| Forward secrecy | ✅ Yes | ✅ Yes | ⚠️ Depends on impl | ⚠️ Partial | ✅ Yes |
| Requires ISP cooperation | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Resistance to global passive adversary | ⚠️ Partial (cover traffic helps) | ⚠️ Partial | ❌ ISP sees all | ⚠️ Partial | ⚠️ Partial |

---

## BGP-X vs. Tor

### Where they agree

Both BGP-X and Tor are built on the same foundational insight: **no single node should be able to see both the origin and destination of a connection**. Both use layered encryption (onion routing) to enforce this. Both provide forward secrecy. Both are open-source and designed to resist surveillance.

### Where BGP-X goes further

#### 1. Decentralized discovery (no directory authorities)

**Tor**: Uses a small set of hard-coded directory authorities (currently ~9) that collectively publish a signed consensus document describing all active relays. This creates a structural centralization risk:

- The directory authorities are known, fixed targets for legal coercion, DDoS, and subversion
- If a majority of directory authorities are compromised, the relay list can be manipulated
- Directory authorities have geographic and jurisdictional concentration

**BGP-X**: Uses a Kademlia-style DHT for node advertisement. Every node publishes a self-signed advertisement. There is no consensus document and no directory authority. A client that discovers even one DHT node can find all others through the DHT's routing algorithm. There is no single entity to coerce, subpoena, or DDoS in order to degrade the network's anonymity set.

#### 2. Variable path length

**Tor**: All circuits are exactly 3 hops. This is a deliberate design choice to balance anonymity and latency. However, it also means all users have identical path structure, which can aid traffic analysis. There is no mechanism for a high-risk user to use a 5-hop path without modifying Tor internals.

**BGP-X**: Path length is a first-class configuration parameter. The default is 4 hops. High-risk use cases can use 5, 6, or 7 hops. The path selection API exposes length, geographic diversity constraints, and trust thresholds. This allows the same protocol to serve both general-purpose browsing and high-security communications without protocol modification.

#### 3. UDP-native transport

**Tor**: Tor wraps everything in TCP connections between relays. This means Tor cannot natively carry UDP traffic (DNS queries, VoIP, gaming). UDP-over-Tor is not supported by the base protocol. Additionally, TCP connections between relays carry identifiable fingerprints.

**BGP-X**: The relay protocol runs over UDP natively. BGP-X carries any IP traffic — TCP, UDP, ICMP, or custom protocols. For application streams that require ordered delivery, BGP-X provides a reliability layer. For fire-and-forget UDP applications, no overhead is added.

#### 4. Network-layer integration

**Tor**: Tor Browser and the Tor daemon provide SOCKS5 proxy interfaces. Applications must be explicitly configured to use SOCKS5. Applications that do not support SOCKS5 (e.g., system daemons, non-browser applications) require complex workarounds (iptables rules, torsocks wrapper, Whonix-style OS integration).

**BGP-X**: The BGP-X client exposes a TUN interface. The operating system routes IP traffic into this interface the same way it routes traffic to a physical network adapter. Any application — including those with no network privacy awareness — routes through BGP-X transparently. This is the same model as a VPN client, extended with the privacy properties of onion routing.

#### 5. Stream multiplexing

**Tor**: Each TCP stream creates or reuses a circuit. A typical browser session (loading a single web page) may involve dozens of TCP connections, each potentially requiring its own circuit setup or sharing a circuit in ways that can correlate streams.

**BGP-X**: Multiple streams are multiplexed over a single path using a stream ID in the packet header. Path setup is amortized across all streams in a session. This reduces overhead and eliminates the inter-stream correlation risk that arises when streams share a Tor circuit.

#### 6. Router firmware integration

**Tor**: Tor is a user-space daemon designed for individual devices. Running Tor at the router level requires significant custom configuration and is not officially supported on commodity router firmware.

**BGP-X**: BGP-X is designed with router firmware as a first-class deployment target. The node daemon is implemented to run within the memory and CPU constraints of commodity OpenWrt-capable routers. When BGP-X runs at the router level, all devices on the network segment are protected without per-device configuration — the same user experience as a VPN router, with genuine multi-hop privacy.

---

## BGP-X vs. VPN

### The fundamental VPN problem

A VPN provides one hop of trust displacement. Instead of your ISP seeing your traffic, your VPN provider sees it. The VPN provider knows:

- Your real IP address
- Every destination you connect to
- Timing and volume of all your traffic
- Plaintext content (for non-HTTPS connections)

If the VPN provider is compromised, subpoenaed, or malicious, all of that data is available to the adversary.

### BGP-X's approach

BGP-X distributes trust across N independent nodes. No single node has the complete picture. To reconstruct the source-destination link, an adversary must control multiple nodes simultaneously **and** they must be in the same path **and** they must be both the entry and exit of that path.

Additionally, BGP-X node operators do not have a business relationship with users. There is no payment data, no account, no email address linking a user to their BGP-X activity.

### When a VPN is still appropriate

VPNs are appropriate when:

- You trust the VPN provider and want a simple, low-latency solution
- Your threat model is ISP surveillance only (not an adversary who can subpoena VPN logs)
- You need consistent exit IP (e.g., for geo-restriction bypass)
- Latency is more important than strong privacy guarantees

BGP-X is appropriate when:

- You need privacy that does not depend on trusting any single provider
- Your threat model includes legal coercion of service providers
- You are building infrastructure that must be resilient to targeted observation
- You need network-layer privacy for all applications, not just browsers

---

## BGP-X vs. I2P

### Similarities

Both BGP-X and I2P use garlic/onion routing and are fully decentralized. Both support native services (BGP-X native services, I2P eepsites). Both are designed to resist traffic analysis.

### Differences

**Clearnet access**: I2P is primarily designed as an internal network. Clearnet access via I2P outproxies is limited and not a primary use case. BGP-X is designed from the ground up for full clearnet access via gateways, with internal BGP-X native services as an additional capability.

**Transport**: I2P uses NTCP2 (TCP-based) and SSU2 (UDP-based) transports. BGP-X uses UDP natively with a custom reliability layer tuned for multi-hop overlay routing.

**Network integration**: I2P exposes a SOCKS proxy and HTTP proxy. BGP-X exposes a TUN interface for full network-layer integration.

**Path model**: I2P uses unidirectional tunnels (separate inbound and outbound paths). BGP-X uses bidirectional paths with separate return routing. The BGP-X model is simpler to reason about for general internet traffic.

**Router firmware**: I2P requires a JVM and has significant resource requirements. BGP-X is implemented in Rust and designed for embedded/router deployment.

---

## BGP-X vs. SCION

### What SCION does

SCION (Scalability, Control, and Isolation On Next-generation networks) is a next-generation internet architecture that provides:

- Path-aware networking (senders can choose network paths)
- Isolation between routing domains (ISDs)
- Strong PKI-based trust

SCION is a genuine architectural replacement for parts of BGP, designed to be deployed at the ISP and infrastructure level.

### The fundamental difference

**SCION requires ISP cooperation.** It is not an overlay. To use SCION, your ISP must run SCION infrastructure. Today, SCION is deployed in limited academic and research environments. It is not available to general internet users.

**BGP-X requires no infrastructure changes.** It runs on top of the existing internet, using BGP as dumb transport. Any user with an internet connection can use BGP-X today (once the reference implementation exists). Any operator with a server can run a BGP-X node today.

### Complementarity

BGP-X and SCION are not competitors. If SCION becomes widely deployed, BGP-X can use SCION paths as its transport layer, gaining SCION's path-integrity properties at Layer 0 while retaining BGP-X's privacy properties at the overlay layer. This is a natural future integration point.

---

## Property Deep Dives

### Resistance to traffic correlation

Traffic correlation is the most challenging attack against any low-latency anonymity network. An adversary who can observe traffic entering the overlay (at the entry node) and traffic exiting the overlay (at the exit node) can potentially correlate them based on timing and volume patterns.

**Tor**: Vulnerable to correlation by a global passive adversary. Tor's design accepts this as a fundamental limitation of low-latency networks. Research has demonstrated practical correlation attacks under certain conditions.

**BGP-X**: Has the same fundamental vulnerability as Tor regarding timing correlation, because both are low-latency networks. BGP-X mitigates this through:

- Variable path length (longer paths increase the correlation difficulty)
- Geographic diversity requirements (reduces the probability that a single adversary observes both ends)
- Pluggable cover traffic module (adds synthetic traffic to disrupt timing fingerprints)
- Stream multiplexing (multiple streams on one path create ambiguity about which stream generates which packets)

Cover traffic is the most effective mitigation but comes at a bandwidth cost. It is off by default and can be enabled for high-risk sessions.

### Forward secrecy

Both BGP-X and Tor provide forward secrecy at the session level. Session keys are ephemeral — derived from a Diffie-Hellman exchange and destroyed after the session. Compromise of a node's long-term static key does not decrypt past sessions.

VPNs vary widely in forward secrecy implementation. Many commercial VPNs do not implement forward secrecy correctly.

### Exit node trust

**Tor**: Any volunteer can run an exit node. There is no vetting, signing, or exit policy enforcement at the protocol level. Exit nodes are expected to publish an exit policy (what ports and destinations they will forward), but this is advisory.

**BGP-X**: Exit nodes must publish a signed exit policy before they appear in the DHT as exit-capable nodes. The exit policy specifies exactly what traffic the gateway will forward and what logging (if any) is performed. Clients can filter gateways by policy before path construction. Gateways that violate their published policy are detectable and reportable through the reputation system.

This does not eliminate the need to trust gateway operators — but it makes their commitments explicit, verifiable, and machine-readable.
