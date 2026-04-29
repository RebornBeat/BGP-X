# BGP-X Frequently Asked Questions

**Version**: 0.1.0-draft

---

## General

### What is BGP-X?

BGP-X is a router-level privacy overlay network and inter-protocol domain router. It runs on your network router or as a standalone daemon and protects all connected devices without requiring any application modification. It also enables cross-domain routing between clearnet, BGP-X overlay, and mesh island networks.

### Is BGP-X a VPN?

No. A VPN routes all your traffic through a single server operated by a single provider — that provider sees your source IP, every destination, timing, volume, and plaintext content for non-HTTPS connections. If the VPN provider is compromised, subpoenaed, or malicious, all that data is available to the adversary.

BGP-X distributes trust across N independent nodes. No single node has the complete picture. Entry nodes see your IP but not the destination. Exit nodes see the destination but not your IP. No single entity controls both.

BGP-X and a VPN can be layered if desired: routing BGP-X through a VPN hides your entry node connection from your ISP, but adds latency and reduces the anonymity advantage over using BGP-X alone.

### Can I use BGP-X with my existing VPN?

Yes. They can be layered in two configurations:

**BGP-X inside VPN** (BGP-X traffic goes through VPN):
```
Your device → VPN → Internet → BGP-X entry → relays → exit → Destination
```
Your ISP sees VPN traffic. VPN provider sees encrypted BGP-X packets (cannot identify entry nodes). BGP-X entry node sees VPN provider's IP, not your real IP. This provides additional protection against entry node observation but adds latency.

**BGP-X outside VPN** (BGP-X carries VPN traffic):
```
Your device → BGP-X entry → relays → BGP-X exit → VPN server → Destination
```
Less common. VPN server acts as the final clearnet hop after BGP-X exit.

When in doubt: BGP-X alone provides strong protection without VPN overhead. Add VPN only if your specific threat model requires hiding BGP-X participation from your ISP.

### How is BGP-X different from Tor?

BGP-X differs from Tor in these key ways:

- **Router-level deployment**: BGP-X runs on your router and protects all devices without per-device configuration. Tor requires per-application configuration (SOCKS5 proxy).
- **No fixed hop count**: BGP-X paths are configurable (default 4, no maximum). Tor always uses 3 hops.
- **Mesh transport**: BGP-X operates on WiFi 802.11s, LoRa, and Bluetooth mesh without ISP connectivity. Tor requires internet.
- **Cross-domain routing**: BGP-X clearnet clients can reach mesh island services without special hardware. Tor is internet-only.
- **Pool-based trust**: BGP-X has configurable pool trust tiers. Tor uses Directory Authorities.
- **ECH at exit**: BGP-X exit nodes support ECH to hide domain names from SNI. Tor does not.
- **N-hop unlimited**: BGP-X has no protocol maximum on path length. Tor is fixed at 3 hops.
- **COVER traffic**: BGP-X COVER packets use session_key and are externally identical to RELAY. Tor uses padding.

### Is BGP-X ready to use?

BGP-X is currently in pre-implementation specification phase. All documentation describes the complete system design. Reference implementation in Rust is the next phase.

---

## Architecture

### How does BGP-X work on a router?

The BGP-X daemon runs alongside the router's standard network stack. When a packet arrives from a LAN device, the routing policy engine decides whether it routes through the BGP-X overlay or directly to the internet. BGP-X overlay traffic enters the TUN interface (bgpx0), gets onion-encrypted by the daemon, and exits as encrypted UDP to the first relay node.

### What are the three equal entry points?

**Clearnet**: your standard ISP connection. BGP-X daemon routes traffic through the onion overlay via standard internet.

**BGP-X Overlay**: the onion-encrypted routing layer itself. Paths may stay within the overlay for additional hops before exiting.

**Mesh Islands**: community radio networks (WiFi 802.11s, LoRa, Bluetooth). No ISP required within the island. Cross-domain access to/from clearnet via bridge nodes.

All three are equal — no entry point is privileged, required, or more primary than the others.

### What is a DHT Pool?

A pool is a named, signed collection of BGP-X nodes grouped by trust level or operational purpose. Example: use relays from the default public pool for the first 3 hops, but exit through your own private servers for the last hop. The private exit servers only see traffic from the public pool relay — not from you directly.

Types: Public, Curated, Private, Semi-private, Ephemeral, Domain-scoped (restricted to nodes in a specific routing domain), Domain-bridge (composed of bridge nodes for a specific domain pair).

### What is a domain bridge node?

A domain bridge node is any BGP-X node with endpoints in two or more routing domains. It receives traffic on one domain (e.g., clearnet UDP) and forwards it to another domain (e.g., WiFi mesh radio). It decrypts only its own DOMAIN_BRIDGE onion layer and forwards the remaining encrypted inner layers without modification. The bridge node cannot decrypt inner layers.

### Can a clearnet user with no mesh hardware reach a mesh island service?

Yes. Install the BGP-X daemon on your device. The daemon discovers domain bridge nodes that connect clearnet to the target mesh island via the unified DHT. It constructs a cross-domain path including the bridge node. The bridge node receives your encrypted traffic via clearnet UDP and transmits it into the mesh island via its radio. You never send or receive any radio signals. The mesh island service never learns your clearnet IP.

### Is there a hop count limit?

No. BGP-X imposes no protocol-level maximum on total path hops, domain segment count, or domain traversal count. The common header has no hop counter. No mechanism rejects a packet after N hops.

Practical defaults: 4 hops for single-domain clearnet; 7 hops for two-domain cross-domain; 10 hops for three domains. A 20-hop path across 4 domains is as valid as a 4-hop single-domain path. The daemon logs a soft informational message above 15 hops (high latency expected) but does not enforce any limit.

### What is the unified DHT?

BGP-X operates one unified Kademlia DHT spanning all routing domains. All nodes — clearnet, mesh island, satellite — participate in the same key space. There is no separate mesh DHT and no synchronization between separate DHTs (the previous two-DHT model was replaced). Domain bridge nodes serve as the physical DHT storage infrastructure accessible from the internet for mesh-only nodes. Mesh-only nodes access the unified DHT via their bridge nodes' caches.

---

## Technical

### How does onion encryption work?

For a 4-hop path (entry → relay 1 → relay 2 → exit), the client wraps the packet in 4 layers:

```
Layer 4 (outermost): encrypted for Entry
  Layer 3: encrypted for Relay 1
    Layer 2: encrypted for Relay 2
      Layer 1 (innermost): encrypted for Exit
        [destination + payload + path_id]
```

Each layer also includes the path_id (8 bytes). The entry decrypts layer 4, sees next hop is Relay 1 and path_id, stores path_id → client, forwards the still-encrypted inner layers. And so on.

For cross-domain paths, domain bridge hops use hop_type DOMAIN_BRIDGE (0x06). The bridge node decrypts its layer, reads the target domain instruction, and forwards remaining inner layers via the target domain's transport without re-encrypting them.

### Does BGP-X hide the domain name I'm connecting to?

Partially. BGP-X hides your IP from the destination and hides the destination from your ISP. The exit node must connect via TLS, which normally includes SNI — the domain name in plaintext.

**Encrypted Client Hello (ECH)** hides the domain when the destination publishes ECH configuration in its DNS HTTPS record and the exit node is ECH-capable. With ECH: network observers between exit and destination see only the destination IP, not the domain name.

Without ECH: the exit node sees the domain in TLS ClientHello SNI.

### What cryptographic algorithms does BGP-X use?

| Function | Algorithm |
|---|---|
| Key exchange | X25519 |
| Signatures | Ed25519 |
| Symmetric encryption | ChaCha20-Poly1305 |
| Hash | BLAKE3 |
| KDF | HKDF-SHA256 |

COVER packets use the same session_key as RELAY — externally indistinguishable. Identical crypto in all routing domains (clearnet, mesh, satellite). No domain-specific key derivation.

### What happens if a mesh island has no internet connection?

The island operates in offline mode. Intra-island BGP-X traffic continues normally — full onion encryption and DHT within the island. Cross-domain traffic is impossible until a bridge node reconnects. When a bridge node comes back online, it publishes the island's advertisement to the unified DHT, and cross-domain paths become available again.

If `store_and_forward = true` is configured, cross-domain stream open requests are queued and delivered when the island reconnects.

### Can I use Meshtastic hardware with BGP-X?

Yes, via the adaptation layer. Meshtastic devices (ESP32 or nRF52840 + LoRa radio) are too constrained to run the full BGP-X daemon. However, they serve as LoRa radio modems for a BGP-X router (Raspberry Pi, GL.iNet router, etc.) connected via USB. The BGP-X adaptation firmware removes the Meshtastic routing protocol and exposes raw LoRa mode. When combined with a BGP-X router, the Meshtastic device enables the router to act as a domain bridge node connecting clearnet to a LoRa mesh island.

---

## Operations

### What should I do if I find a malicious node?

Report it through the BGP-X reputation system:

```bash
bgpx-cli nodes blacklist <node_id> --reason "suspected MITM"
```

Enable global blacklist enforcement (opt-in) to consider community reports:

```toml
[reputation]
enforce_global_blacklist = true
```

For malicious exit gateways violating their exit policy, include specific behavior (TLS interception, traffic modification) in the report. See SECURITY.md for the vulnerability reporting process.

### How does a LAN device access the SDK socket on the router?

Option A (SSH socket forwarding, recommended):

```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@router
```

Option B (router exposes TCP endpoint):

```toml
[sdk_api]
tcp_listen = "192.168.1.1:7475"
tcp_auth_required = true
```

Option C: If your application doesn't need path control, just run it normally. The router transparently routes its traffic through the overlay with no SDK required.

### What does geographic plausibility scoring protect against?

Node operators can lie about their geographic location in their advertisements (claiming to be in Germany when actually in Singapore). Geographic plausibility scoring uses round-trip time measurements to detect this: a node claiming EU that responds in 400ms from an EU client is suspicious (EU-EU should be under 240ms round-trip). Detected implausibility generates reputation penalties reducing selection probability.

This is a signal, not a hard exclusion — and it's domain-specific. LoRa mesh nodes expected to have high latency are not flagged for it. Satellite nodes are entirely exempt.

---

## Security

### What are BGP-X's limitations?

- **Timing correlation**: all low-latency anonymity networks share this fundamental residual risk; cover traffic raises attacker cost but does not eliminate it
- **ECH requires destination support**: destinations must publish ECH configuration in DNS HTTPS records
- **Geographic plausibility is RTT-signal-based**: not cryptographic proof of location
- **Application-layer leakage**: BGP-X cannot prevent an app from revealing identity via cookies, logins, or fingerprinting
- **Compromised device**: BGP-X cannot protect traffic from malware on your device
- **No legal protection**: BGP-X does not make illegal activities legal
- **Not post-quantum**: v1 uses classical cryptography (X25519, Ed25519)
