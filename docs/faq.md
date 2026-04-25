# BGP-X Frequently Asked Questions

---

## General

### What is BGP-X?

BGP-X is a router-level privacy overlay network. It runs on your network router and protects all connected devices transparently — routing their traffic through a chain of encrypted relay nodes such that no single party can see both where traffic comes from and where it is going.

It is not a replacement for BGP (Border Gateway Protocol) — it uses the existing internet as a transport layer and builds a private routing layer on top of it.

---

### Is BGP-X a VPN?

No. A VPN routes your traffic through a single server operated by a single provider. That provider sees your source IP, your destination, and all unencrypted content.

BGP-X routes traffic through a chain of multiple independent nodes selected from a decentralized DHT. No single node sees both where you are and where you are going. There is no single provider to trust.

---

### Is BGP-X a Tor fork?

No. BGP-X shares the concept of multi-hop onion routing with Tor but differs fundamentally:

- **Discovery**: BGP-X uses a fully decentralized DHT; Tor uses centralized directory authorities
- **Path length**: BGP-X supports configurable paths (no global enforcement maximum); Tor uses fixed 3-hop circuits
- **Transport**: BGP-X uses UDP native with reliability layer; Tor uses TCP only
- **Network integration**: BGP-X operates at IP layer via TUN interface; Tor is a SOCKS5 proxy
- **Multiplexing**: BGP-X multiplexes multiple streams per path; Tor creates one circuit per connection
- **Pools**: BGP-X has pool-based trust domains with double-exit architecture; Tor has no equivalent
- **ECH**: BGP-X exit nodes support ECH; Tor does not
- **Mesh transport**: BGP-X operates without ISP via radio mesh; Tor is internet-only
- **Hardware ecosystem**: BGP-X has a unified hardware platform; Tor has no hardware

---

### Does BGP-X replace BGP?

No. BGP-X uses the BGP-routed internet as a transport layer. It does not participate in BGP peering, does not announce routes, and requires no ISP cooperation.

For mesh deployments (no ISP), BGP-X provides an alternative to BGP routing for intra-community traffic. Coverage gaps between mesh islands are bridged via internet relay pools. BGP is still used as physical transport where internet connectivity exists.

---

### Is BGP-X the first mesh network?

No. BGP-X builds on significant prior art:

- **cjdns / Hyperboria**: public key addressing for overlay networks
- **Yggdrasil Network**: DHT-based routing with cryptographic identities
- **Reticulum Network Stack**: multi-transport mesh for low-bandwidth links
- **Meshtastic**: demonstrated LoRa mesh hardware and community adoption

**What BGP-X uniquely provides**: the combination of multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust domains + hardware targets in one coherent system.

---

### Is BGP-X ready to use?

BGP-X is currently in **pre-implementation specification phase**. All documentation describes the complete system design. Reference implementation in Rust is the next phase.

---

## Architecture

### How does BGP-X work on a router?

The BGP-X daemon runs alongside your router's standard network stack. When a packet arrives from a LAN device, the routing policy engine decides whether it routes through the BGP-X overlay or directly to the internet. BGP-X traffic enters the TUN interface (bgpx0), gets onion-encrypted by the daemon, and exits as encrypted UDP to the first relay node. All of this is transparent to the LAN devices.

---

### What are the three interfaces the daemon exposes?

**TUN (bgpx0)**: virtual network interface. Traffic routed here is captured for overlay routing. Applications need no modification.

**SDK Socket** (`/var/run/bgpx/sdk.sock`): BGP-X native applications connect here to access overlay capabilities explicitly (path constraints, service registration, pool selection, ECH requirements).

**Control Socket** (`/var/run/bgpx/control.sock`): bgpx-cli and GUI tools connect here for management (status, statistics, routing policy, pools, reputation).

---

### What is a DHT Pool?

A pool is a named, signed collection of BGP-X nodes grouped by trust level or operational purpose.

Instead of using one flat pool of all relays, you can structure paths across multiple pools:

- **Public pool**: open DHT, any node can join
- **Curated pool**: a curator signs membership; elevated trust signal
- **Private pool**: your own nodes; membership not published; maximum trust
- **Semi-private pool**: discoverable but restricted join

Example: "Use relays from the default public pool, but exit through my private exit nodes."

Pools enable the **double-exit architecture**: two independent exit pools in sequence, so no single exit node sees both the path origin and the destination.

---

### Can I use my own servers as BGP-X exit nodes and route through them privately?

Yes. Create a private pool containing your servers. Configure your path constraints to use that pool as the exit segment. Your servers will only be used by clients who know the pool configuration (since it's private). Path segments before your servers come from the public pool, so even your exit servers don't see the origin.

---

### What does path_id do?

path_id is an 8-byte random identifier assigned to each path at construction time. It is included in every onion layer of that path.

When a relay node decrypts its onion layer, it records `path_id → predecessor_address` in memory. When return traffic arrives referencing that path_id, the relay forwards it toward the predecessor without decrypting it.

This enables bidirectional sessions without leaking path composition to intermediate relays. Relays see path_id but not the actual path structure or node identities of other hops.

---

### What is a Broadcast Amplifier?

Mode 6. A simple device with a single radio that receives BGP-X mesh packets and rebroadcasts them. No routing intelligence, no DHT participation, no encryption or decryption. Just physics — extending radio range.

Uses: extending LoRa mesh coverage over hills or obstacles; bridging short gaps between mesh segments.

Hardware: minimal (STM32H7 microcontroller + LoRa radio + PoE power), under 800mW, outdoor weatherproof.

---

### Can I use more than 4 hops?

Yes. BGP-X has no global enforcement maximum. The default is 4 hops. You can configure any value. The daemon logs a soft warning above 10 hops but does not enforce a limit.

Higher hops means more latency and higher anonymity. Choose based on your threat model.

---

### What is Geographic Plausibility Scoring?

BGP-X cannot verify that a node's claimed region/country is accurate (there's no authority to check against). Instead, BGP-X measures the round-trip time (RTT) to each node via KEEPALIVE timing and compares it to the expected range for the node's claimed region.

A node claiming to be in Europe that responds in 400ms from a European client is suspicious (EU-EU should be 8-120ms). This discrepancy generates a reputation penalty (GEO_SUSPICIOUS or GEO_IMPLAUSIBLE events).

This is not a hard exclusion — it's a reputation signal. Nodes with persistent geographic implausibility accumulate reputation penalties over time.

Mesh nodes have separate thresholds since LoRa hops can be 100ms-5s by design.

---

## Technical

### How does onion encryption work?

For a 4-hop path (entry → relay 1 → relay 2 → exit), the client wraps the packet in 4 layers:

```
Layer 4 (outermost): encrypted for Entry
  Layer 3: encrypted for Relay 1
    Layer 2: encrypted for Relay 2
      Layer 1 (innermost): encrypted for Exit
        [destination + payload]
```

Each layer also includes the path_id (8 bytes). The entry decrypts layer 4, sees next hop is Relay 1 and path_id, stores path_id → client, forwards the still-encrypted inner layers. And so on.

Each layer is encrypted using a session key derived from a Diffie-Hellman key exchange between the client and that specific node.

---

### Does BGP-X hide the domain name I'm connecting to?

Partially. BGP-X hides your IP from the destination and hides the destination from your ISP. The exit node must connect to the destination via TLS, which normally includes the Server Name Indication (SNI) — the domain name in plaintext.

**Encrypted Client Hello (ECH)** hides the domain name from SNI when:
- The destination publishes an ECH configuration in its DNS HTTPS record
- The exit node is ECH-capable

With ECH: the exit node (and network observers between exit and destination) see only the destination IP, not the domain name.

Without ECH: the exit node sees the domain name in the TLS ClientHello SNI.

---

### What cryptographic algorithms does BGP-X use?

| Function | Algorithm |
|---|---|
| Signatures | Ed25519 |
| Key exchange | X25519 |
| Symmetric encryption | ChaCha20-Poly1305 |
| Hashing | BLAKE3 |
| KDF | HKDF-SHA256 |

COVER packets use the same session_key as RELAY packets — they are externally indistinguishable to any observer.

No separate cover_key is derived.

---

### Can I use Meshtastic hardware with BGP-X?

Yes, via an adaptation layer. Meshtastic devices (ESP32 or nRF52840 + LoRa radio) are too constrained to run the full BGP-X daemon. However, they can serve as **LoRa radio modems** for a BGP-X router (Raspberry Pi, GL.iNet router, etc.) connected via USB.

The BGP-X daemon sends and receives raw LoRa packets via the Meshtastic device's serial interface. A BGP-X-specific firmware for the ESP32/nRF52840 removes the Meshtastic protocol and exposes raw LoRa mode.

See `/hardware/meshtastic_adapter.md` for details.

---

### What is the BGP Replacement Spectrum?

A framework describing how much BGP-X can replace ISP-level BGP routing depending on deployment:

- **Level 0**: pure overlay, BGP unchanged — BGP-X adds privacy only
- **Level 1**: intra-mesh routing — 100% BGP-free for local mesh traffic
- **Level 2**: mesh + gateway — mesh users have zero direct BGP exposure
- **Level 3**: multi-island bridging — islands connected via internet relay pools; individuals never directly exposed
- **Level 4**: dense multi-mode hybrid — variable by position
- **Level 5**: aerial/satellite mesh — near-total BGP independence
- **Level 6**: maximum — BGP-X replaces ISP routing decisions; honest limit: cannot replace physical infrastructure

---

## Operations

### How does a LAN device access the BGP-X router daemon's SDK socket?

Option A — SSH socket forwarding (recommended):
```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@router
```
Then in your application:
```rust
let client = Client::connect(Path::new("/tmp/bgpx-router.sock")).await?;
```

Option B — Router exposes TCP endpoint (configure in daemon):
```toml
[sdk_api]
tcp_listen = "192.168.1.1:7475"
tcp_auth_required = true
```

Option C — If your app doesn't need path control or service registration, just run it as a standard application. The router transparently routes its traffic through the overlay with no SDK required.

---

### What should I do if I find a malicious node?

Report it through the BGP-X reputation system. Nodes can be flagged in the DHT via signed blacklist records. The reputation system aggregates these as advisory signals (not automatic exclusions by default).

Blacklist a node locally via:
```bash
bgpx-cli nodes blacklist <node_id> --reason "suspected MITM"
```

Enable global blacklist enforcement (opt-in):
```toml
[reputation]
enforce_global_blacklist = true
```

For malicious exit gateways violating their exit policy, include specific behavior in the report (TLS interception, traffic modification). See `SECURITY.md` for the vulnerability reporting process.

---

### Can I use BGP-X with my existing VPN?

Yes. They can be layered:
```
BGP-X overlay → gateway → VPN → internet
```
Or:
```
BGP-X overlay → gateway → internet (BGP-X alone)
```

If both are active, the ISP sees VPN traffic. The VPN provider sees encrypted BGP-X traffic. The BGP-X entry node sees the VPN provider's IP. This provides additional protection against entry node observation but adds latency.
