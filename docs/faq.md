# BGP-X Frequently Asked Questions

**Version**: 0.1.0-draft

---

## General

### What is BGP-X?

BGP-X is a router-level privacy overlay network and inter-protocol domain router. It runs on your network router or as a standalone daemon and protects all connected devices without requiring any application modification. It also enables cross-domain routing between clearnet, BGP-X overlay, and mesh island networks.

### How is BGP-X different from a VPN?

A VPN routes all your traffic through one provider's server — that provider sees both you and your destination. BGP-X uses onion routing: no single node sees both who you are and where you're going. BGP-X also has no central provider; it's a distributed network.

### How is BGP-X different from Tor?

BGP-X runs at the router level (all devices protected), supports mesh island radio networks (no ISP required), enables cross-domain routing between clearnet and mesh, has no fixed hop-count limit, uses pool-based trust for path selection, and targets dedicated hardware. Tor is per-application or per-device and internet-only with fixed 3-hop circuits.

### Does BGP-X protect application-layer data?

BGP-X protects network-level metadata: source IP, destination IP, and traffic volume are hidden from relays. Application content is protected by HTTPS/TLS when used. BGP-X does not protect against application-layer identity leakage (cookies, JavaScript, login sessions).

---

## Cross-Domain Routing

### Can a standard internet user reach a service inside a mesh island?

Yes. Install the BGP-X daemon on your device (no mesh hardware or BGP-X router required). The daemon discovers domain bridge nodes that connect clearnet to the mesh island, constructs a cross-domain path, and the bridge node handles all radio transmission on your behalf. You never send or receive any radio signals.

### Can a mesh island user (no ISP) reach clearnet services?

Yes, via a gateway node (domain bridge node with ISP connection). The gateway bridges the mesh island to the clearnet internet. Mesh users route clearnet traffic through the gateway's ISP connection. Individual mesh island members have zero direct ISP exposure.

### Can two mesh islands communicate?

Yes, two ways:

1. **Via clearnet overlay**: mesh island A → gateway A → clearnet relays → gateway B → mesh island B. Both islands need at least one internet-connected gateway node.

2. **Via direct island-to-island bridge**: a node with two radio interfaces (WiFi mesh for island A + LoRa for island B) bridges the islands directly. No clearnet involvement.

### Is there a limit on hops or routing domains?

No protocol-level limit. BGP-X imposes no maximum on total path hops, domain count, or domain traversal count. You can specify a path through clearnet → mesh island A → clearnet → mesh island B → clearnet exit. Practical limits are latency (more hops = more latency) and node availability.

### What is a domain bridge node?

A domain bridge node is any BGP-X node with endpoints in two or more routing domains. It receives traffic on one domain (e.g., clearnet UDP) and forwards it to the other domain (e.g., WiFi mesh radio). It decrypts only its own DOMAIN_BRIDGE onion layer and forwards the remaining encrypted payload unchanged. The bridge node is the single point of domain translation.

### Do cross-domain paths still maintain anonymity?

Yes. The split-knowledge guarantee is maintained across domain boundaries. The clearnet entry node knows the client's clearnet IP but not the mesh island destination. The domain bridge node knows which clearnet relay forwarded to it and which mesh relay received from it — but not the client's IP or the service identity. Mesh relays know only their immediate neighbor addresses. No single node in any domain can link source to destination.

### What happens when a mesh island has no internet connection?

The island operates in offline mode: intra-island BGP-X traffic continues normally; cross-domain traffic is impossible until a bridge node reconnects. When a bridge node comes back online, it publishes the island's advertisement to the unified DHT, and cross-domain paths become available again. Store-and-forward can be configured for applications that tolerate delayed delivery.

### Does BGP-X have a separate mesh DHT and internet DHT?

No. BGP-X operates one unified DHT spanning all routing domains. All nodes participate in the same Kademlia key space. Domain bridge nodes (including gateway nodes) serve as the physical infrastructure for DHT storage accessible from the internet. Mesh-only nodes access the unified DHT via their bridge nodes' caches.

### How does a clearnet client discover a mesh island service?

1. BGP-X daemon queries unified DHT for the service record (keyed by ServiceID)
2. Service record indicates it's registered in `mesh:target-island`
3. Daemon queries DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
4. Daemon constructs cross-domain path via discovered bridge nodes
5. Connection established transparently

---

## Privacy and Security

### What can my ISP see if I use BGP-X?

Your ISP sees encrypted UDP traffic to the BGP-X entry node's IP address. They cannot determine your destination, the content of your traffic, or which service you're accessing. They may determine that you're using BGP-X.

### Can a relay node compromise my privacy?

A single relay node cannot link your source IP to your destination — it sees only its immediate predecessor and successor addresses. Multiple colluding relay nodes are constrained by ASN, country, and operator diversity requirements. If both the entry and exit nodes are controlled by the same adversary, correlation is possible — pool-based routing and operator diversity reduce this probability.

### Is BGP-X post-quantum secure?

No. BGP-X v1 uses classical cryptography (X25519, Ed25519, ChaCha20-Poly1305). Post-quantum resistance is not in scope for v1.

---

## Technical

### What ports does BGP-X use?

Default: **7474/UDP**. Configurable. Pluggable transport available to obfuscate port and protocol fingerprint.

### Does BGP-X support IPv6?

Yes. Implementations SHOULD support IPv6. Dual-stack (IPv4 + IPv6) is RECOMMENDED.

### Can I run BGP-X on a Raspberry Pi?

Yes. BGP-X runs on ARM64 (Raspberry Pi 4, 3) and ARMv7 (Raspberry Pi 2). Minimum 256 MB RAM for relay-only; 512 MB for domain bridge.

### What happens to traffic that doesn't match any routing policy rule?

Configurable default action. Default: route via BGP-X overlay. Can be changed to `standard` (bypass) or `block` per operator preference.
