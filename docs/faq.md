# BGP-X Frequently Asked Questions

---

## General

### What is BGP-X?

BGP-X is a privacy-preserving, identity-routed overlay routing network. It routes your internet traffic through a chain of encrypted relay nodes such that no single party can see both where your traffic comes from and where it is going.

It is not a replacement for BGP (Border Gateway Protocol) — it uses the existing internet as a transport layer and builds a private routing layer on top of it.

---

### Is BGP-X a VPN?

No. A VPN routes your traffic through a single server operated by a single provider. That provider sees your source IP, your destination, and all unencrypted content. You are trading ISP surveillance for VPN provider surveillance.

BGP-X routes traffic through a chain of multiple independent nodes. No single node sees both where you are and where you are going. There is no single provider to trust — trust is cryptographically distributed across the path.

---

### Is BGP-X a Tor fork?

No. BGP-X shares the concept of multi-hop onion routing with Tor but differs fundamentally in:

- **Discovery**: BGP-X uses a fully decentralized DHT; Tor uses centralized directory authorities
- **Path length**: BGP-X supports configurable N-hop paths; Tor uses a fixed 3-hop circuit
- **Transport**: BGP-X uses UDP with a custom reliability layer; Tor uses TCP
- **Network integration**: BGP-X operates at the IP layer natively; Tor is a SOCKS5 proxy
- **Multiplexing**: BGP-X multiplexes multiple streams per path; Tor creates one circuit per connection

See `docs/comparison.md` for a full side-by-side comparison.

---

### Is BGP-X ready to use?

BGP-X is currently in the **pre-implementation specification phase**. All documentation describes the complete system design. Reference implementation in Rust is the next phase. There is no production software to download or install yet.

---

### Why UDP instead of TCP?

TCP is a stateful, ordered, reliable transport. Those properties are useful for the application layer — but they add overhead and complexity at the relay layer.

BGP-X relays do not need ordered delivery or connection state. They need to:

1. Receive a packet
2. Decrypt one layer
3. Forward to the next hop

UDP is the right primitive for this. BGP-X implements a lightweight reliability layer above UDP for streams that require it (e.g., stream ordering for application data), but the relay path itself is connectionless and stateless.

---

### Does BGP-X replace DNS?

No. BGP-X includes a resolution layer (for mapping service names to BGP-X service identities), but this does not replace DNS for clearnet access. When you access a clearnet destination through a BGP-X gateway, the gateway resolves the domain name via its own DNS resolver before connecting to the destination server.

---

### What does BGP-X not protect against?

BGP-X is explicit about its limitations:

- **Compromised client device**: If malware or spyware is on your device, it can observe your traffic before BGP-X encrypts it.
- **Application-layer identity leakage**: If you log into a service with your real username, the service knows who you are. BGP-X cannot fix that.
- **Global traffic correlation**: A sufficiently powerful adversary who can observe the full public internet can attempt to correlate when traffic enters the overlay and when it exits. This is a known limitation of all low-latency anonymity networks.
- **Exit node observation of clearnet traffic**: The exit gateway knows the destination IP. If you connect to a clearnet service without HTTPS, the exit node can also read the content.

---

## Technical

### How does onion encryption work?

When you send a packet through a 4-hop path (entry → relay 1 → relay 2 → exit), the client wraps the packet in 4 encryption layers:

```
Layer 4 (outermost): encrypted for Entry Node
  Layer 3: encrypted for Relay 1
    Layer 2: encrypted for Relay 2
      Layer 1 (innermost): encrypted for Exit Node
```

The entry node decrypts layer 4, sees "forward to relay 1," and forwards the still-encrypted packet (layers 1–3). Relay 1 decrypts layer 3. Relay 2 decrypts layer 2. The exit node decrypts the final layer and sees the destination.

Each layer is encrypted using a session key derived from a Diffie-Hellman key exchange between the client and that specific node. No node can decrypt any layer other than its own.

---

### What cryptographic algorithms does BGP-X use?

| Function | Algorithm | Reason |
|---|---|---|
| Signatures | Ed25519 | Fast, compact, well-audited |
| Key exchange | X25519 | State-of-the-art ECDH |
| Symmetric encryption | ChaCha20-Poly1305 | Fast without AES-NI; authenticated |
| Hashing | BLAKE3 | High performance; modern design |
| KDF | HKDF-SHA256 | Standard, well-specified |

---

### How are nodes discovered?

BGP-X uses a Kademlia-style DHT (Distributed Hash Table). Nodes publish signed advertisements to the DHT. These advertisements include the node's public key, roles, performance data, and a self-signed timestamp.

Clients query the DHT for nodes matching their path selection criteria. The DHT has no central operator. Any node can participate in the DHT.

---

### What is a NodeID?

A NodeID is derived deterministically from a node's public key:

```
NodeID = BLAKE3(Ed25519_public_key)
```

NodeIDs are 32 bytes (256 bits). They are used as DHT keys for node advertisement lookup and as identifiers in path construction.

---

### What is the difference between an entry node and a relay node?

An entry node is the first hop in your path. It is the only node that sees your real IP address. It does not see your destination.

A relay node is a middle hop. It sees only its immediate predecessor and successor. It knows neither your IP nor your destination.

In practice, a single node daemon can serve both roles depending on how it is configured and how it is selected by clients.

---

### What is an exit node / gateway?

The gateway is the last hop in your path for clearnet traffic. It is the only node that sees the destination IP. It does not know your real IP address — it only knows the previous relay.

The gateway connects to the public internet and sends your traffic to the destination on your behalf. Responses from the destination are received by the gateway and relayed back through the path to you.

---

### Can I run a BGP-X node on a home internet connection?

Relay nodes can run on home connections, but with caveats:

- Many home ISPs use CG-NAT (carrier-grade NAT), which makes your node unreachable from the public internet without port forwarding.
- Home IP addresses are often dynamic, which affects node advertisement stability.
- Bandwidth limitations affect how much relay traffic you can handle.

Exit nodes / gateways should be run on dedicated, stable, public IP addresses — preferably in a data center or colocation facility.

---

### What is the minimum hardware to run a relay node?

The relay node daemon is lightweight. Minimum:

- 1 CPU core
- 512 MB RAM
- 10 Mbps symmetric bandwidth (100+ Mbps recommended)
- Stable public IP address

The dominant resource constraint for relay nodes is network bandwidth, not CPU or RAM.

---

### How does path construction ensure geographic diversity?

The client's path selection algorithm enforces:

- No two nodes in the same Autonomous System (ASN)
- No two nodes in the same country
- Configurable additional diversity constraints (e.g., maximum path latency, minimum geographic spread)

ASN and country information comes from the node's signed advertisement. Nodes that misrepresent their ASN or country are detectable via the reputation system when their routing behavior is inconsistent with their advertisement.

---

## Operations

### Can I use BGP-X with my existing VPN?

Yes. BGP-X and a VPN can be layered:

```
BGP-X overlay → gateway → VPN → internet
```

or

```
BGP-X overlay → gateway → internet
```

If both BGP-X and a VPN are active, the ISP sees only encrypted VPN traffic. The VPN provider sees encrypted BGP-X traffic. The BGP-X entry node sees the VPN provider's IP (not your real IP). This provides additional protection against entry node observation but adds latency.

---

### Will BGP-X work on my router?

BGP-X firmware support is in development for OpenWrt-compatible routers. When available, this allows all devices on your network to route through BGP-X without per-device configuration.

See `/firmware/firmware.md` for supported hardware targets and installation guidance.

---

### How do I know if a gateway is trustworthy?

BGP-X gateways publish a signed exit policy that specifies:

- What traffic they will forward
- Whether they log any data (and what)
- Operator identity
- Jurisdiction

The exit policy is signed with the gateway's private key and verifiable by anyone. Clients can filter gateways by exit policy criteria before path selection.

A gateway that claims a no-log policy cannot be forced to produce logs it never collected. However, you should verify that the gateway's jurisdiction and operator reputation support that claim. This is the same trust calculus as evaluating a VPN provider — with the significant difference that BGP-X does not depend on a single gateway's honesty for privacy (the entry node does not know the destination, and the gateway does not know the source).

---

### What should I do if I find a malicious node?

Report it through the reputation system and to the BGP-X project maintainers. Malicious nodes can be blacklisted in client software via signed revocation records published to the DHT.

If the node is operating an exit gateway, include the specific malicious behavior (e.g., TLS interception, traffic modification) in your report. See `SECURITY.md` for the vulnerability reporting process.
