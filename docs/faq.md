# BGP-X Frequently Asked Questions

**Version**: 0.1.0-draft

---

## General

### What is BGP-X?

BGP-X is a router-level privacy overlay network and inter-protocol domain router. It runs on your network router or as a standalone daemon and protects all connected devices without requiring any application modification. It also enables cross-domain routing between clearnet, BGP-X overlay, and mesh island networks.

It is not a replacement for BGP (Border Gateway Protocol) — it uses the existing internet as a transport layer and builds a private routing layer on top of it. For mesh deployments, BGP-X provides an alternative to BGP routing for intra-community traffic.

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

### Is BGP-X a Tor fork?

No. BGP-X shares the concept of multi-hop onion routing with Tor but differs fundamentally:

- **Discovery**: BGP-X uses a fully decentralized DHT; Tor uses centralized directory authorities
- **Path length**: BGP-X supports configurable N-hop paths with no global enforcement maximum; Tor uses fixed 3-hop circuits
- **Transport**: BGP-X uses UDP native with a reliability layer; Tor uses TCP only
- **Network integration**: BGP-X operates at the IP layer via TUN interface; Tor is a SOCKS5 proxy
- **Multiplexing**: BGP-X multiplexes multiple streams per path; Tor creates one circuit per connection
- **Mesh transport**: BGP-X operates on WiFi 802.11s, LoRa, and Bluetooth mesh without ISP connectivity; Tor is internet-only
- **Cross-domain routing**: BGP-X clearnet clients can reach mesh island services without special hardware; Tor is internet-only
- **Pool-based trust**: BGP-X has configurable pool trust tiers with double-exit architecture; Tor has no equivalent
- **ECH at exit**: BGP-X exit nodes support Encrypted Client Hello; Tor does not
- **Hardware ecosystem**: BGP-X has unified hardware platform (Router v1, Node v1, Gateway v1); Tor has no hardware
- **Router-level deployment**: BGP-X runs on your router protecting all devices; Tor requires per-application configuration
- **COVER traffic**: BGP-X COVER packets use session_key and are externally indistinguishable from RELAY; Tor uses padding

### Does BGP-X protect application-layer data?

BGP-X protects network-level metadata: source IP, destination IP, and traffic volume are hidden from relays. Application content is protected by HTTPS/TLS when used. BGP-X does not protect against application-layer identity leakage (cookies, JavaScript, login sessions).

### Does BGP-X replace DNS?

No. BGP-X includes a resolution layer (for mapping service names to BGP-X service identities), but this does not replace DNS for clearnet access. When you access a clearnet destination through a BGP-X gateway, the gateway resolves the domain name via its own DNS resolver before connecting to the destination server.

### Is BGP-X the first mesh network?

No. BGP-X builds on significant prior art:

- **cjdns / Hyperboria**: public key addressing for overlay networks
- **Yggdrasil Network**: DHT-based routing with cryptographic identities
- **Reticulum Network Stack**: multi-transport mesh for low-bandwidth links
- **Meshtastic**: demonstrated LoRa mesh hardware and community adoption

**What BGP-X uniquely provides**: the combination of multi-hop onion routing + mesh transport + clearnet exit + decentralized DHT + pool-based trust domains + hardware targets in one coherent system.

### Is BGP-X ready to use?

BGP-X is currently in pre-implementation specification phase. All documentation describes the complete system design. Reference implementation in Rust is the next phase. There is no production software to download or install yet.

### Why UDP instead of TCP?

TCP is a stateful, ordered, reliable transport. Those properties are useful for the application layer — but they add overhead and complexity at the relay layer.

BGP-X relays do not need ordered delivery or connection state. They need to:

1. Receive a packet
2. Decrypt one layer
3. Forward to the next hop

UDP is the right primitive for this. BGP-X implements a lightweight reliability layer above UDP for streams that require it (e.g., stream ordering for application data), but the relay path itself is connectionless and stateless.

---

## Hardware

### What hardware do I need as an end user?

**BGP-X Router v1** — one device, plugs in where your current home router is, replaces it entirely. All your LAN devices are automatically protected. No additional hardware needed for any BGP-X capability including WiFi mesh, LoRa, and domain bridge operation.

### What if I already have a compatible router?

Install the **BGP-X OpenWrt Package** on your existing GL.iNet, Banana Pi, or Raspberry Pi router. You get clearnet overlay protection immediately. Add a USB LoRa adapter (Meshtastic USB adapter or standalone USB LoRa module) for domain bridge operation.

### What is the BGP-X Node v1?

The Node v1 is for community mesh contributors — people who want to help expand BGP-X mesh coverage in their neighborhood without replacing their home router. Mount it on a rooftop or mast, power it via PoE or solar, and it adds relay capacity to the local mesh island. It can also be a community gateway (with WAN Ethernet) or operate in range-extension mode.

**Important**: BGP-X Node v1 runs the **complete** bgpx-node daemon — the same software as BGP-X Router v1. The difference is form factor and deployment context, not capability. It is NOT a "lite" or partial device.

### Does BGP-X have a simple LoRa range extender?

No — and intentionally so. A dumb LoRa repeater that relays unencrypted LoRa frames at the PHY level breaks end-to-end encryption. BGP-X Node v1 in Range Extension mode provides equivalent coverage benefit while maintaining full BGP-X onion encryption at every hop. Every forwarding device in the BGP-X network is a proper node running the full protocol.

**Retired concept**: The original BGP-X specification included a "BGP-X Amplifier v1" as a separate product category — a minimal LoRa repeater with no BGP-X routing intelligence. This concept has been retired. Use BGP-X Node v1 in Range Extension mode instead.

### What is the BGP-X Client Node?

A low-cost, battery-powered endpoint for connecting to BGP-X mesh networks. Does not route traffic for others. Based on LILYGO T3S3, T-Beam, Heltec V3, or RAK WisBlock hardware. For individual users, IoT sensors, and portable operation.

### What is the BGP-X Adapter/Dongle?

A USB LoRa modem for existing computers. The BGP-X daemon runs on the host computer; the adapter is just a radio interface.

### Can I connect Starlink to a BGP-X router?

Yes. The BGP-X Router v1 and Node v1 detect Starlink terminals via USB vendor ID and load the satellite transport driver automatically. Starlink provides the WAN (clearnet) connection. BGP-X overlay runs on top of it identically to fiber or cellular. A BGP-X Router v1 with Starlink WAN is a perfectly capable clearnet relay, domain bridge, and exit node — it just has satellite-class latency on its clearnet side.

### Is satellite internet a separate BGP-X domain?

No. Starlink, Iridium, Inmarsat, and all current commercial satellite services are clearnet domain (0x00000001). See docs/satellite_architecture.md for the complete explanation.

### What is the minimum hardware to run a relay node?

The relay node daemon is lightweight. Minimum:

- 1 CPU core
- 512 MB RAM
- 10 Mbps symmetric bandwidth (100+ Mbps recommended)
- Stable public IP address

The dominant resource constraint for relay nodes is network bandwidth, not CPU or RAM.

### Can I run BGP-X on a home internet connection?

Relay nodes can run on home connections, but with caveats:

- Many home ISPs use CG-NAT (carrier-grade NAT), which makes your node unreachable from the public internet without port forwarding.
- Home IP addresses are often dynamic, which affects node advertisement stability.
- Bandwidth limitations affect how much relay traffic you can handle.

Exit nodes / gateways should be run on dedicated, stable, public IP addresses — preferably in a data center or colocation facility.

### Can I run BGP-X on a Raspberry Pi?

Yes. BGP-X runs on ARM64 (Raspberry Pi 4, 3) and ARMv7 (Raspberry Pi 2). Minimum 256 MB RAM for relay-only; 512 MB for domain bridge.

### Will BGP-X work on my router?

BGP-X firmware support is in development for OpenWrt-compatible routers. When available, this allows all devices on your network to route through BGP-X without per-device configuration.

See `/firmware/firmware.md` for supported hardware targets and installation guidance.

### Can I use Meshtastic hardware with BGP-X?

Yes, via an adaptation layer. Meshtastic devices (ESP32 or nRF52840 + LoRa radio) are too constrained to run the full BGP-X daemon. However, they can serve as **LoRa radio modems** for a BGP-X router (Raspberry Pi, GL.iNet router, etc.) connected via USB.

The BGP-X daemon sends and receives raw LoRa packets via the Meshtastic device's serial interface. A BGP-X-specific firmware for the ESP32/nRF52840 removes the Meshtastic protocol and exposes raw LoRa mode.

When combined with a BGP-X router, the Meshtastic device enables the router to act as a domain bridge node connecting clearnet to a LoRa mesh island.

See `/hardware/meshtastic_adapter.md` for details.

---

## Architecture

### How does BGP-X work on a router?

The BGP-X daemon runs alongside the router's standard network stack. When a packet arrives from a LAN device, the routing policy engine decides whether it routes through the BGP-X overlay or directly to the internet. BGP-X overlay traffic enters the TUN interface (bgpx0), gets onion-encrypted by the daemon, and exits as encrypted UDP to the first relay node. All of this is transparent to the LAN devices.

### What are the three local interfaces?

**TUN (bgpx0)**: virtual network interface. Traffic routed here is captured for overlay routing. Applications need no modification. Handles all IP protocols transparently.

**SDK Socket** (`/var/run/bgpx/sdk.sock`): BGP-X native applications connect here to access overlay capabilities explicitly (path constraints, service registration, pool selection, ECH requirements). Also accessible as TCP endpoint for LAN devices connecting to router daemon.

**Control Socket** (`/var/run/bgpx/control.sock`): bgpx-cli and GUI tools connect here for management (status, statistics, routing policy, pools, reputation).

### What are the three equal entry points?

**Clearnet**: your standard ISP connection — fiber, cellular, or satellite internet (all of these are clearnet; BGP-X overlay runs on top).

**BGP-X Overlay**: the onion-encrypted routing layer. Paths may stay within the overlay for additional hops before exiting.

**Mesh Islands**: community radio networks (WiFi 802.11s, LoRa, Bluetooth). No ISP required within the island. Cross-domain access to/from clearnet via bridge nodes.

All three are equal — no entry point is privileged, required, or more primary than the others. A mesh-only user in a remote village and a fiber-connected user in a city participate in the same network.

### What is a .bgpx address?

A `.bgpx` address is a self-authenticating service identifier — equivalent to a Tor .onion address. The address is the hex encoding of the service's Ed25519 public key (64 hex characters). No certificate authority is required — the address IS the cryptographic identity.

Human-readable short names (`shortname.bgpx`) are available via the BGP-X Name Registry DHT.

BGP-X Browser handles `bgpx://` URLs natively. Standard apps (not BGP-X native) use clearnet URLs transparently routed through the overlay.

### Can I browse .bgpx sites over a LoRa path?

Yes. LoRa paths have high latency — hundreds of milliseconds to seconds per round-trip, comparable to slow Tor .onion browsing. BGP-X Browser adapts to this:

- Batches critical resources into fewer round-trips
- Renders skeleton layout immediately while waiting for full content
- Shows latency indicator so you know why loading is slower
- Never shows a blank white page — always shows partial content or loading state

It is functional and usable. It is not fast. The browser makes the latency visible and manages it gracefully, the same way Tor Browser handles slow .onion sites. Applications designed for BGP-X (setting `X-BGP-X-Latency-Class: lora` headers, using Brotli compression, setting high cache TTLs) work best on LoRa paths.

### Does BGP-X use HTTP/2 or HTTP/3?

For `.bgpx` services: **HTTP/2 over BGP-X streams**. BGP-X already provides reliable ordered delivery at the session layer, so HTTP/2's multiplexing is the right fit. HTTP/3's QUIC would add redundant reliability and congestion control.

For clearnet sites accessed via BGP-X exit: the exit node negotiates HTTP/3 with the destination if supported. This is standard HTTP/3 over TLS over the exit's clearnet connection — not over BGP-X streams.

For LoRa paths: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

### What is a DHT Pool?

A pool is a named, signed collection of BGP-X nodes grouped by trust level or operational purpose.

Instead of using one flat pool of all relays, you can structure paths across multiple pools:

- **Public pool**: open DHT, any node can join
- **Curated pool**: a curator signs membership; elevated trust signal
- **Private pool**: your own nodes; membership not published; maximum trust
- **Semi-private pool**: discoverable but restricted join
- **Domain-scoped pool**: restricted to nodes in a specific routing domain
- **Domain-bridge pool**: composed of bridge nodes for a specific domain pair

Example: "Use relays from the default public pool, but exit through my private exit nodes."

Pools enable the **double-exit architecture**: two independent exit pools in sequence, so no single exit node sees both the path origin and the destination.

### Can I use my own servers as BGP-X exit nodes and route through them privately?

Yes. Create a private pool containing your servers. Configure your path constraints to use that pool as the exit segment. Your servers will only be used by clients who know the pool configuration (since it's private). Path segments before your servers come from the public pool, so even your exit servers don't see the origin.

### What does path_id do?

path_id is an 8-byte random identifier assigned to each path at construction time. It is included in every onion layer of that path.

When a relay node decrypts its onion layer, it records `path_id → predecessor_address` in memory. When return traffic arrives referencing that path_id, the relay forwards it toward the predecessor without decrypting it.

This enables bidirectional sessions without leaking path composition to intermediate relays. Relays see path_id but not the actual path structure or node identities of other hops.

For cross-domain paths, domain bridge nodes additionally store `path_id → domain_a_predecessor` and `path_id → domain_b_predecessor` in their cross-domain path table. Return traffic is forwarded opaquely across domain boundaries — intermediate relays do NOT decrypt return traffic.

### What is a NodeID?

A NodeID is derived deterministically from a node's public key:

```
NodeID = BLAKE3(Ed25519_public_key)
```

NodeIDs are 32 bytes (256 bits). They are used as DHT keys for node advertisement lookup and as identifiers in path construction.

### What is the difference between an entry node and a relay node?

An entry node is the first hop in your path. It is the only node that sees your real IP address. It does not see your destination.

A relay node is a middle hop. It sees only its immediate predecessor and successor. It knows neither your IP nor your destination.

In practice, a single node daemon can serve both roles depending on how it is configured and how it is selected by clients.

### What is an exit node / gateway?

The gateway is the last hop in your path for clearnet traffic. It is the only node that sees the destination IP. It does not know your real IP address — it only knows the previous relay.

The gateway connects to the public internet and sends your traffic to the destination on your behalf. Responses from the destination are received by the gateway and relayed back through the path to you.

### What is a domain bridge node?

A domain bridge node is any BGP-X node with endpoints in two or more routing domains. It receives traffic on one domain (e.g., clearnet UDP) and forwards it to another domain (e.g., WiFi mesh radio). It decrypts only its own DOMAIN_BRIDGE onion layer and forwards the remaining encrypted payload unchanged. The bridge node cannot decrypt inner layers.

A BGP-X Node v1 with WAN Ethernet + LoRa radio is a domain bridge node. A BGP-X Router v1 in bridge configuration is a domain bridge node. Any OpenWrt router with USB LoRa adapter and BGP-X package is a domain bridge node.

---

## Cross-Domain Routing

### Can a standard internet user reach a service inside a mesh island?

Yes. Install the BGP-X daemon on your device (no mesh hardware or BGP-X router required). The daemon discovers domain bridge nodes that connect clearnet to the mesh island via the unified DHT, constructs a cross-domain path including the bridge node, and the bridge node handles all radio transmission on your behalf. You never send or receive any radio signals. The mesh island service never learns your clearnet IP.

### Can a mesh island user (no ISP) reach clearnet services?

Yes, via a gateway node (domain bridge node with ISP connection). The gateway bridges the mesh island to the clearnet internet. Mesh users route clearnet traffic through the gateway's ISP connection. Individual mesh island members have zero direct ISP exposure.

### Can two mesh islands communicate?

Yes, two ways:

1. **Via clearnet overlay**: mesh island A → gateway A → clearnet relays → gateway B → mesh island B. Both islands need at least one internet-connected gateway node.

2. **Via direct island-to-island bridge**: a node with two radio interfaces (WiFi mesh for island A + LoRa for island B) bridges the islands directly. No clearnet involvement.

### Do cross-domain paths still maintain anonymity?

Yes. The split-knowledge guarantee is maintained across domain boundaries. The clearnet entry node knows the client's clearnet IP but not the mesh island destination. The domain bridge node knows which clearnet relay forwarded to it and which mesh relay received from it — but not the client's IP or the service identity. Mesh relays know only their immediate neighbor addresses. No single node in any domain can link source to destination.

### Is there a hop count limit?

No. BGP-X imposes no protocol-level maximum on total path hops, domain segment count, or domain traversal count. The common header has no hop counter. No mechanism rejects a packet after N hops.

Practical defaults: 4 hops for single-domain clearnet; 7 hops for two-domain cross-domain; 10 hops for three domains. A 20-hop path across 4 domains is as valid as a 4-hop single-domain path. The daemon logs a soft informational message above 15 hops (high latency expected) but does not enforce any limit.

### Can I use more than 4 hops?

Yes. BGP-X has no global enforcement maximum. The default is 4 hops. You can configure any value. The daemon logs a soft warning above 10 hops but does not enforce a limit.

Higher hops means more latency and higher anonymity. Choose based on your threat model.

### What happens when a mesh island has no internet connection?

The island operates in offline mode. Intra-island BGP-X traffic continues normally with full onion encryption and DHT within the island. Cross-domain traffic (clearnet access) is unavailable until a bridge node reconnects. When a bridge node comes back online, it publishes the island's advertisement to the unified DHT, and cross-domain paths become available again within ~60 seconds.

If `store_and_forward = true` is configured, cross-domain stream open requests are queued and delivered when the island reconnects.

### Does BGP-X have a separate mesh DHT and internet DHT?

No. BGP-X operates **one unified DHT** spanning all routing domains. All nodes — clearnet, mesh island, satellite — participate in the same Kademlia key space. There is no separate mesh DHT and no synchronization between separate DHTs. Domain bridge nodes serve as the physical DHT storage infrastructure accessible from the internet for mesh-only nodes. Mesh-only nodes access the unified DHT via their bridge nodes' caches.

There is no "mesh DHT" that syncs with an "internet DHT" — there is one DHT.

### How does a clearnet client discover a mesh island service?

1. BGP-X daemon queries unified DHT for the service record (keyed by ServiceID)
2. Service record indicates it's registered in `mesh:target-island`
3. Daemon queries DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
4. Daemon constructs cross-domain path via discovered bridge nodes
5. Connection established transparently

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

For clearnet sites: partially. BGP-X hides your IP from the destination and the destination from your ISP. The exit node must connect via TLS, which normally includes SNI — the domain name in plaintext.

**Encrypted Client Hello (ECH)** hides the domain when the destination publishes ECH configuration in its DNS HTTPS record and the exit node is ECH-capable. With ECH: network observers between exit and destination see only the destination IP, not the domain name.

Without ECH: the exit node sees the domain in TLS ClientHello SNI.

For `.bgpx` sites: completely. There is no clearnet TLS handshake. The BGP-X onion encryption provides transport security. No domain name is visible to any network observer.

### What cryptographic algorithms does BGP-X use?

| Function | Algorithm |
|---|---|
| Key exchange | X25519 |
| Signatures | Ed25519 |
| Symmetric encryption | ChaCha20-Poly1305 |
| Hash | BLAKE3 |
| KDF | HKDF-SHA256 |

All algorithms are applied identically in all routing domains — clearnet, mesh, satellite internet, LoRa. No domain-specific cipher variations exist. COVER packets use the same session_key as RELAY — externally indistinguishable in all domains.

### What ports does BGP-X use?

Default: **7474/UDP**. Configurable. Pluggable transport available to obfuscate port and protocol fingerprint.

### Does BGP-X support IPv6?

Yes. Implementations SHOULD support IPv6. Dual-stack (IPv4 + IPv6) is RECOMMENDED.

### What happens to traffic that doesn't match any routing policy rule?

Configurable default action. Default: route via BGP-X overlay. Can be changed to `standard` (bypass) or `block` per operator preference.

---

## Privacy and Security

### What can my ISP see if I use BGP-X?

Your ISP sees encrypted UDP traffic to the BGP-X entry node's IP address. They cannot determine your destination, the content of your traffic, or which service you're accessing. They may determine that you're using BGP-X.

### Can a relay node compromise my privacy?

A single relay node cannot link your source IP to your destination — it sees only its immediate predecessor and successor addresses. Multiple colluding relay nodes are constrained by ASN, country, and operator diversity requirements. If both the entry and exit nodes are controlled by the same adversary, correlation is possible — pool-based routing and operator diversity reduce this probability.

### What is Geographic Plausibility Scoring?

BGP-X cannot verify that a node's claimed region/country is accurate (there's no authority to check against). Instead, BGP-X measures the round-trip time (RTT) to each node via KEEPALIVE timing and compares it to the expected range for the node's claimed region.

A node claiming to be in Europe that responds in 400ms from a European client is suspicious (EU-EU should be 8-120ms). This discrepancy generates a reputation penalty (GEO_SUSPICIOUS or GEO_IMPLAUSIBLE events).

**Geographic plausibility scoring is OPTIONAL**:

- If a node declares a jurisdiction: geo plausibility scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply
- Nodes are NOT required to declare jurisdiction
- Declaring jurisdiction is an opt-in privacy/convenience tradeoff

This is not a hard exclusion — it's a reputation signal. Nodes with persistent geographic implausibility accumulate reputation penalties over time.

Mesh nodes have separate thresholds since LoRa hops can be 100ms-5s by design. Satellite nodes are entirely exempt.

### What are BGP-X's hardware security features?

- Ed25519 node private key stored in TPM 2.0 (hardware security module); never exposed to OS
- Session keys protected with mlock() to prevent swap to disk
- eMMC storage encrypted with TPM-sealed AES key
- Secure boot: TPM PCR chain verifies firmware integrity
- Signed firmware updates only; unsigned firmware rejected
- Swap permanently disabled in all BGP-X firmware

---

## Jurisdiction and Geo Plausibility

### Do nodes have to declare their jurisdiction?

No. Jurisdiction declaration is **OPTIONAL**.

- If a node declares a jurisdiction: geographic plausibility RTT-based scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

Declaring jurisdiction is a personal/operational choice. The network functions identically with or without it.

### Why might a node NOT declare jurisdiction?

- **Privacy**: Don't want to reveal any location information
- **Mobile/vehicle nodes**: Location changes
- **Satellite nodes**: IP may be geographically distant from terminal location (geo plausibility exempt anyway)

### Why might a node declare jurisdiction?

- **Reputation**: Strong geo plausibility scores are a positive signal
- **Trust**: Users may prefer nodes in specific jurisdictions
- **Regulatory compliance**: Some operators may be required to disclose location

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

### How does a LAN device access .bgpx sites?

If using BGP-X Browser: install BGP-X Browser on your LAN device. It connects to the BGP-X Router v1 daemon via the SDK socket and handles `.bgpx` URL resolution automatically.

If using a standard browser: the BGP-X Router v1 daemon intercepts all traffic from LAN devices. Standard clearnet traffic routes through the overlay transparently. `.bgpx` addresses are only accessible via BGP-X Browser or SDK-aware applications — standard browsers cannot resolve `.bgpx` addresses without the BGP-X Browser networking layer.

### How do I set up a mesh island?

See `docs/user_guide/mesh_island_setup.md`. In short:
1. Choose a unique island_id string
2. Configure one or two BGP-X Node v1 (or Router v1) units as gateway nodes (with WAN + mesh radio)
3. Configure additional nodes as relay nodes (no WAN needed)
4. All nodes use the same `island_id` in their config
5. The gateway publishes the island to the unified DHT
6. Other BGP-X users worldwide can discover and reach your island's services

### What should I do if I find a malicious node?

Report it through the BGP-X reputation system:

```bash
bgpx-cli nodes blacklist <node_id> --reason "suspected MITM"
```

Enable global blacklist consideration (opt-in) to consider community reports:

```toml
[reputation]
enforce_global_blacklist = true
```

For malicious exit gateways violating their exit policy, include specific behavior (TLS interception, traffic modification) in the report. See `SECURITY.md` for the vulnerability reporting process.

### How do I know if a gateway is trustworthy?

BGP-X gateways publish a signed exit policy that specifies:

- What traffic they will forward
- Whether they log any data (and what)
- Operator identity
- Jurisdiction

The exit policy is signed with the gateway's private key and verifiable by anyone. Clients can filter gateways by exit policy criteria before path selection.

A gateway that claims a no-log policy cannot be forced to produce logs it never collected. However, you should verify that the gateway's jurisdiction and operator reputation support that claim. This is the same trust calculus as evaluating a VPN provider — with the significant difference that BGP-X does not depend on a single gateway's honesty for privacy (the entry node does not know the destination, and the gateway does not know the source).

---

## What is the BGP Replacement Spectrum?

A framework describing how much BGP-X can replace ISP-level BGP routing depending on deployment:

- **Level 0**: pure overlay, BGP unchanged — BGP-X adds privacy only
- **Level 1**: intra-mesh routing — 100% BGP-free for local mesh traffic
- **Level 2**: mesh + gateway — mesh users have zero direct BGP exposure
- **Level 3**: multi-island bridging — islands connected via internet relay pools; individuals never directly exposed
- **Level 4**: dense multi-mode hybrid — variable by position
- **Level 5**: aerial/satellite mesh — near-total BGP independence
- **Level 6**: maximum — BGP-X replaces ISP routing decisions

**Honest limit**: BGP-X replaces ISP dependency and routing decisions, not physical infrastructure (fiber, towers, datacenters).

---

## What does BGP-X not protect against?

BGP-X is explicit about its limitations:

- **Compromised client device**: If malware or spyware is on your device, it can observe your traffic before BGP-X encrypts it.
- **Application-layer identity leakage**: If you log into a service with your real username, the service knows who you are. BGP-X cannot fix that.
- **Global traffic correlation**: A sufficiently powerful adversary who can observe the full public internet can attempt to correlate when traffic enters the overlay and when it exits. This is a known limitation of all low-latency anonymity networks.
- **Exit node observation of clearnet traffic**: The exit gateway knows the destination IP. If you connect to a clearnet service without HTTPS, the exit node can also read the content.

### What are all of BGP-X's limitations?

- **Timing correlation**: all low-latency anonymity networks share this fundamental residual risk; cover traffic raises attacker cost but does not eliminate it
- **ECH requires destination support**: destinations must publish ECH configuration in DNS HTTPS records
- **Geographic plausibility is RTT-signal-based**: not cryptographic proof of location; OPTIONAL feature
- **Application-layer leakage**: BGP-X cannot prevent apps from revealing identity via cookies or logins
- **Compromised device**: BGP-X cannot protect against malware on your own device
- **No legal protection**: BGP-X does not make illegal activities legal
- **LoRa latency**: LoRa paths are high latency; standard web apps designed for <100ms feel slow
- **LoRa duty cycle**: EU ETSI regulations limit LoRa to 1% duty cycle; high-traffic LoRa paths will experience paused stream events; streams resume automatically when duty cycle resets
- **Satellite latency**: GEO satellite paths add 600ms+ RTT. Suitable for non-interactive traffic; challenging for real-time applications
- **No post-quantum cryptography (v1)**: current version uses classical cryptography. Post-quantum upgrade planned for future version.
