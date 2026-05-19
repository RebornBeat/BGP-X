# Getting Started with BGP-X

**Version**: 0.1.0-draft

This document is your entry point to understanding, deploying, and using BGP-X. It is written for four audiences:

- **Network operators** — people who want to deploy BGP-X on a router for their network.
- **Node operators** — people who want to run BGP-X relay, entry, or exit nodes.
- **Developers** — people who want to build applications using the BGP-X SDK.
- **End users** — people whose traffic is protected by a BGP-X router.

---

## 1. Prerequisites

Before reading further, you should understand the following concepts at a basic level:

- What an IP address is and how it differs from a domain name.
- What a public/private keypair is and how asymmetric cryptography works at a conceptual level.
- What a VPN is and why it has privacy limitations.
- What Tor is and how onion routing works conceptually.

If any of these are unclear, read `docs/faq.md` first — it covers these foundations.

---

## 2. What BGP-X Provides

BGP-X provides a **network-layer privacy overlay**. Concretely:

When your traffic routes through BGP-X:

- Your ISP sees only encrypted traffic to the first BGP-X relay — not your destination.
- The entry node knows your IP address but not your destination.
- The exit node knows the destination but not your IP address.
- No single party in the chain can see both ends simultaneously.
- All traffic between hops is encrypted with per-hop, per-session keys.

BGP-X does not make you invisible. It distributes and limits what any single party can observe.

---

## 3. Deployment Modes

BGP-X supports six deployment modes. Choose the one that fits your situation:

| Mode | Description | ISP Needed | Hardware Target |
|---|---|---|---|
| **Dual-Stack Router** | BGP + BGP-X coexist; routing policy selects per flow | Yes | BGP-X Router v1 |
| **BGP-X Only Router** | All LAN traffic through overlay; no bypass | Yes (as transport) | BGP-X Router v1 |
| **Standalone Device** | BGP-X daemon on one device only | Yes | Any Linux device |
| **Mesh Node** | No ISP; mesh radio transport only | No | BGP-X Node v1 |
| **Gateway Node** | Bridges mesh to clearnet internet | Yes (at gateway) | BGP-X Node v1 or Router v1 |
| **Range Extension Node** | Coverage extension with full BGP-X routing | No | BGP-X Node v1 (Range Extension mode) |

**For most households**: Dual-Stack Router or BGP-X Only Router.

**For communities without reliable internet**: Mesh Node + Gateway Node.

**For developers**: Standalone Device or SDK with Embedded Mode.

**Note on Range Extension**: The earlier "Broadcast Amplifier" concept has been retired. The BGP-X Node v1 in Range Extension mode provides equivalent coverage while maintaining full BGP-X encryption at every hop.

---

## 4. Architecture at a Glance

BGP-X is router infrastructure — it runs on your router and protects all connected devices:

```
[Phone]────┐
[Laptop]───┼──► [BGP-X Router v1] ──► [BGP-X Overlay Network] ──► [Internet]
[TV]───────┘          │
                      │  Routing Policy Engine decides per flow:
                      ├──► BGP-X overlay (privacy-protected)
                      └──► Standard routing (direct, unprotected)
```

All devices connect to the router as normal. The BGP-X daemon handles everything transparently.

### Network Roles

**Entry Node**: The first hop in your path. Accepts your connection. Knows your real IP address but cannot see your destination — it only knows the next relay to forward to.

**Relay Node**: Middle hops. Knows only the previous hop and the next hop. Has no information about origin or destination. Can be chained arbitrarily.

**Exit Node / Gateway**: The last hop. Knows the destination (the website or service you are connecting to) but cannot see your real IP address — it only knows the last relay. Translates overlay traffic to standard internet traffic.

**Discovery Nodes**: Maintain and serve node advertisement records via the distributed hash table (DHT). Do not handle user traffic.

**Domain Bridge Node**: A node with endpoints in multiple routing domains (e.g., clearnet and mesh). Enables traffic to cross between clearnet, overlay, and mesh islands. Bridge nodes forward traffic between domains without re-encrypting inner onion layers.

---

## 5. Who Uses What

| You Are | What You Need | Configuration |
|---|---|---|
| Device on LAN | Nothing — just connect to router | Zero configuration |
| Building a BGP-X native app | BGP-X SDK | SDK integration |
| Managing the router | bgpx-cli or GUI | Control socket access |
| Running a relay node | bgpx-node | Node configuration |
| Running an exit gateway | bgpx-node + exit policy | Exit configuration + DoH/ECH |

---

## 6. Dual-Stack vs. BGP-X Only

**Dual-Stack** (recommended for most):
- Some traffic goes through BGP-X (private).
- Some traffic goes direct (unprotected, lower latency).
- Routing policy decides which traffic is which.
- Good for: households where some devices need low latency (gaming, smart TV).

**BGP-X Only** (maximum privacy):
- All LAN traffic forced through overlay.
- No bypass possible from LAN devices.
- Slightly higher latency for all traffic.
- Good for: activist networks, high-security environments, shared community networks.

---

## 7. Three Equal Entry Points

BGP-X treats three network classes as equal first-class citizens. No entry point is privileged, none is secondary.

**Clearnet (standard internet)**: You have a standard ISP connection. BGP-X daemon routes your traffic through the onion overlay. No special hardware needed.

**BGP-X overlay**: The onion-encrypted routing layer itself. Paths may stay within the overlay for additional hops before exiting.

**Mesh islands**: Community radio networks (WiFi 802.11s, LoRa, Bluetooth). No ISP required within the island. Cross-domain access to/from clearnet via domain bridge nodes. Users in mesh islands can reach clearnet through gateway nodes.

---

## 8. N-Hop Unlimited

BGP-X imposes **no protocol maximum on path length**. Default is 4 hops for single-domain clearnet. You can specify paths with any number of hops across any combination of routing domains.

```toml
# High-security path: 15 hops across 3 domains
domain_segments = [
    { type = "segment", domain = "clearnet", hops = 5 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 4 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "private-exits", hops = 4, exit = true }
]
```

There is no protocol enforcement of a maximum. Configure paths as needed for your threat model and latency tolerance.

---

## 9. Cross-Domain Routing

BGP-X enables routing between clearnet, overlay, and mesh islands in any combination.

**Clearnet user reaching mesh island service**:
- No mesh hardware needed.
- BGP-X daemon discovers bridge nodes automatically via the unified DHT.
- Bridge node handles radio transmission on your behalf.

**Mesh island reaching clearnet**:
- Route via gateway node (domain bridge with internet connection).
- No direct ISP exposure for island members.

**See `/docs/cross_domain_routing.md` for detailed examples.**

---

## 10. For LAN Devices — No Configuration Needed

If your router runs BGP-X, every device on your network is protected without any changes:

- Connect to the router's WiFi or Ethernet normally.
- Your traffic is automatically routed through BGP-X (for traffic matching the routing policy).
- No app changes, no VPN client, no browser extensions.

---

## 11. For Developers — SDK Connection Model

The BGP-X SDK does NOT implement its own routing. It connects to the daemon:

```
Your App → SDK → Unix Socket → BGP-X Router Daemon → Overlay Network
```

The daemon handles all path construction, encryption, and DHT. Your app just calls `connect_stream()` and gets back a socket.

If the daemon is on a router and your app is on a LAN device:
- **SSH socket forwarding**: `ssh -L /tmp/bgpx.sock:/var/run/bgpx/sdk.sock user@router`
- **TCP endpoint**: router exposes `192.168.1.1:7475` on LAN (requires authentication).
- Or use **SDK Embedded Mode** (full daemon in-process, for mobile).

---

## 12. Identity in BGP-X

Every node and service has a stable cryptographic identity:

```
Identity = Ed25519 public key
NodeID   = BLAKE3(public key)
```

Clients use **ephemeral session identities** — a fresh keypair is generated per connection by default. This means your activity across sessions cannot be linked at the protocol level.

Services (servers, APIs) use **stable identities** that can persist while the underlying infrastructure changes — analogous to a domain name that can point to different servers over time.

---

## 13. DHT Pools

Pools are named, signed collections of BGP-X nodes grouped by trust level. Instead of one flat pool of all relays, you can:

- Use the **default pool** for general traffic.
- Use a **curated pool** (signed by a trusted curator) for higher trust.
- Use a **private pool** (your own nodes) for exit infrastructure you control.
- Chain segments from different pools for multi-tier trust paths.

Example: `Entry from default pool → Relay from trusted pool → Exit from my private pool`.

This is the **double-exit architecture** — two independent exit pools in sequence:

```
Client → [default pool entry + relay] → [private pool exit 1] → [private pool exit 2] → Destination
Exit 1 sees: traffic going to Exit 2 (not destination)
Exit 2 sees: traffic from Exit 1 (not client)
```

---

## 14. Encrypted Client Hello (ECH)

When you connect to a clearnet destination through a BGP-X exit node, the exit node must connect to the destination via TLS. Normally, TLS ClientHello includes the Server Name Indication (SNI) — the domain name in plaintext.

ECH (Encrypted Client Hello) hides the domain name from SNI when:
- The destination publishes an ECH configuration in its DNS HTTPS record.
- The exit node is ECH-capable (`ech_capable = true` in exit policy).

Result: even the exit node does not see the domain name in plaintext. The network sees only the destination IP.

---

## 15. Mesh Modes

BGP-X supports operating without any ISP connection:

**Mesh Node (Mode 4)**: Runs on a router with no WAN connection. Communicates with other BGP-X nodes via WiFi 802.11s mesh or LoRa radio. Can be deployed in rural communities, for disaster recovery, or anywhere ISP access is unavailable.

**Gateway Node (Mode 5)**: Has both mesh radio and a WAN connection. Acts as the bridge — mesh users route clearnet traffic through the gateway. The gateway's ISP sees the traffic; individual mesh users have no direct ISP exposure.

**Range Extension Node (Mode 6)**: BGP-X Node v1 in Range Extension mode. Prioritizes forwarding over session origination. Extends LoRa mesh coverage with full BGP-X encryption.

---

## 16. Pluggable Transport

Your ISP can observe that you're connecting to BGP-X entry nodes (encrypted UDP on port 7474). Pluggable transport obfuscates this traffic pattern to look like something else.

BGP-X's built-in PT uses obfs4-style obfuscation (random padding + stream cipher). External PT subprocess interface is also supported.

Enable with: `[pluggable_transport] enabled = true` in daemon config.

---

## 17. HTTP/2 for .bgpx Services

BGP-X native services (`.bgpx` addresses) use **HTTP/2 over BGP-X streams**.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer.
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path.
- HTTP/3's QUIC would add redundant reliability and congestion control.

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers — this is standard HTTP/3 over TLS over the exit's clearnet connection.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 18. Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal.

- If a node **declares** a jurisdiction in its advertisement: geo plausibility scoring applies.
- If a node **does NOT declare** jurisdiction: geo plausibility scoring does NOT apply.
- Jurisdiction declaration is opt-in.

Satellite-connected nodes (latency_class = satellite-*) are **exempt** from geo plausibility scoring.

---

## 19. What BGP-X Does Not Do

Be clear about this before deploying:

- **BGP-X does not protect you if your application leaks your identity.** If you log into a service with your real username, the service knows who you are regardless of BGP-X.
- **BGP-X does not encrypt content beyond the overlay.** If you connect to a clearnet HTTP (not HTTPS) destination, the exit node can read that content. Always use HTTPS for clearnet connections — the exit node sees destination IP and can see plaintext HTTP content.
- **BGP-X does not protect you if your device is compromised.** If malware is on your device, BGP-X cannot protect your traffic from that malware.
- **BGP-X does not hide that you use BGP-X from your ISP.** Your ISP can observe that you are connected to a BGP-X entry node. Traffic obfuscation (pluggable transport) mitigates this.
- **BGP-X does not prevent traffic blocking.** Your ISP can block port 7474. Pluggable transport mitigates this.
- **BGP-X does not make illegal activities legal.** Using BGP-X does not grant legal protection for any conduct.

---

## 20. Security Recommendations

When using BGP-X, follow these practices:

1. Always use HTTPS for connections to clearnet services (the exit node sees the destination; HTTPS ensures it cannot read the content).
2. Use ephemeral session identities (the default) for maximum privacy.
3. Do not reuse application-layer credentials across BGP-X and non-BGP-X sessions.
4. Run the latest version of the BGP-X client or node daemon.
5. Verify node signatures — never disable signature verification.
6. If operating a gateway, publish and adhere to a clear no-log policy.
7. Enable ECH-requiring exit constraints for sensitive destinations.
8. Use private pool for exit if you control server infrastructure.
9. Use pluggable transport if you're in an environment that
