# BGP and BGP-X: Interoperability, Coexistence, and Domain Routing

**Version**: 0.1.0-draft

This document specifies the complete relationship between BGP (Border Gateway Protocol) and BGP-X, including the layer architecture, gateway behavior, the BGP replacement spectrum, coverage gap bridging, NAT traversal, IPv6 support, traffic analysis boundaries, and the precise limits of BGP independence.

---

## 1. Core Relationship

BGP-X does not participate in BGP. BGP-X does not modify BGP. BGP-X does not require BGP operators to make any changes. BGP-X uses the public internet — which is routed by BGP — as one transport medium among several.

They operate at different layers with no direct coupling.

```
BGP-X Application Layer          ← Applications, SDK
BGP-X Domain Router              ← Cross-domain path construction
BGP-X Overlay Routing            ← Path selection, onion encryption, unified DHT
BGP-X Transport (UDP/IP)         ← BGP-X packets as UDP datagrams on clearnet
BGP-X Transport (Mesh/Satellite) ← BGP-X on radio or satellite transport
         │
         │  Clearnet UDP datagrams are ordinary UDP/7474 to BGP routers
         ▼
BGP Routing Layer                ← ISP routing, AS-level decisions
Physical / Link Layer            ← fiber, radio, physical infrastructure
```

For mesh deployments, the BGP-X Transport layer is replaced by radio (LoRa, WiFi mesh, BLE) with zero BGP involvement. A mesh island's internal traffic is completely invisible to BGP.

This relationship is one-directional in terms of visibility:

- BGP can "see" BGP-X packets (as UDP datagrams with opaque payloads)
- BGP-X can "see" BGP in the sense that it uses BGP-routed paths between nodes
- BGP cannot see inside BGP-X packets
- BGP-X cannot modify BGP routing decisions

---

## 2. What BGP Is (Brief)

The Border Gateway Protocol (BGP) is the routing protocol that connects the internet's Autonomous Systems (AS). Every major network operator (ISP, cloud provider, university) runs BGP to advertise which IP address blocks they own and to exchange routing information with other operators.

BGP determines how packets travel across the internet at the infrastructure level. When your computer sends a packet to a server in another country, BGP-configured routers at each ISP along the path determine the next hop.

BGP has no privacy features. Source and destination IP addresses are visible to every router that handles a packet. BGP is optimized for reachability and stability, not privacy.

---

## 3. What BGP-X Is

BGP-X is a privacy overlay network. It routes traffic through a chain of relay nodes such that no single party can see both where traffic originates and where it is going.

BGP-X is NOT a BGP replacement. BGP-X is NOT a new routing protocol at the infrastructure level. BGP-X is a layer of software that runs on top of BGP-routed infrastructure and uses it as a transparent packet delivery service.

---

## 4. Layer Relationship

```
┌─────────────────────────────────────────────────────────────────┐
│                 BGP-X APPLICATION LAYER                         │
│          (applications, SDK, configuration client)              │
├─────────────────────────────────────────────────────────────────┤
│              BGP-X INTER-PROTOCOL DOMAIN ROUTER                 │
│         (cross-domain path construction, domain bridges)        │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X OVERLAY ROUTING LAYER                     │
│         (onion encryption, unified DHT, pools, reputation)      │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X TRANSPORT LAYER                           │
│     UDP/IP (clearnet) | WiFi mesh | LoRa | BLE | Satellite      │
│     [Satellite = clearnet transport over BGP-routed internet]   │
├─────────────────────────────────────────────────────────────────┤
│                 BGP ROUTING LAYER (clearnet only)               │
│           (ISP routing, AS-level decisions, BGP tables)         │
├─────────────────────────────────────────────────────────────────┤
│                 PHYSICAL / LINK LAYER                           │
│             (fiber, cable, radio, physical infrastructure)      │
└─────────────────────────────────────────────────────────────────┘
```

BGP occupies a layer below BGP-X. BGP-X uses BGP-routed infrastructure as its clearnet transport, the same way it uses WiFi 802.11s as its mesh transport — BGP is one transport option among several, not a peer or a constraint.

For mesh deployments, the transport layer is replaced entirely by radio (LoRa, WiFi mesh, BLE) with zero BGP involvement. A mesh island communicates internally with no BGP participation whatsoever.

---

## 5. What BGP Sees From BGP-X Traffic

To every BGP router along the clearnet path between a BGP-X node and any other clearnet BGP-X node, BGP-X packets appear as:

```
Source IP:      Client/router WAN IP
Destination IP: BGP-X relay node IP
Protocol:       UDP
Port:           7474 (default)
Payload:        Encrypted bytes (opaque)
```

BGP cannot see inside. BGP cannot determine these are BGP-X packets. BGP cannot see the actual destination or whether a path is cross-domain.

**For cross-domain paths**: when traffic traverses a clearnet segment between a domain bridge node and other clearnet relay nodes, BGP sees exactly the same — ordinary UDP between server IPs. BGP cannot determine that the traffic is destined for a mesh island service. BGP cannot see the domain bridge relationship.

From BGP's perspective, BGP-X overlay traffic is indistinguishable from any other UDP traffic on port 7474.

---

## 6. What BGP-X Sees From BGP

BGP-X treats the public internet as a best-effort packet delivery service. BGP-X:

- Knows the IP addresses of remote BGP-X nodes (from their DHT advertisements)
- Sends UDP packets to those IP addresses
- Relies on BGP-routed infrastructure to deliver those packets
- Has NO visibility into BGP routing tables, AS paths, or BGP decisions
- Does NOT participate in BGP peering
- Does NOT announce routes
- Does NOT require any ISP configuration or cooperation

BGP-X uses ASN information only from node advertisements (self-reported by node operators) for path diversity enforcement. This is NOT a BGP lookup — it's the node's self-declared ASN, indirectly verified via geographic plausibility scoring.

---

## 7. Complete Non-Interaction Table

These elements are fully independent between BGP and BGP-X:

| BGP Domain | BGP-X Domain | Relationship |
|---|---|---|
| AS path selection | Overlay/cross-domain path selection | Independent |
| Prefix announcement | Node/island advertisement in unified DHT | Independent |
| BGP peering (TCP/179) | Session handshakes (UDP/7474) | Independent |
| Route filters | Exit policy | Independent |
| RPKI Route Origin Auth | Advertisement Ed25519 signatures | Independent |
| BGP communities | DHT pool membership | Independent |
| IGP (OSPF/IS-IS) | BGP-X DHT routing | Independent |
| BGP hijack detection | Node advertisement signature verification | Independent |
| Anycast routing | BGP-X ServiceID addressing | Independent |
| MPLS | N/A (operates at UDP level) | Independent |

---

## 8. Dual-Stack Traffic Separation

On a dual-stack router:

```
LAN Packet arrives
         │
         ▼
Routing Policy Engine
(domain-aware: may specify domain_segments for cross-domain path)
         │
    ┌────┴──────────────────────┐
    │                           │
    ▼                           ▼
BGP-X overlay path         Standard internet path
(enters bgpx0 TUN)         (enters WAN directly)
    │                           │
    ▼                           ▼
Onion-encrypted UDP        Plain IP packet
→ WAN interface            → WAN interface
    │                           │
    └────────┬──────────────────┘
             │
             ▼
        ISP router
             │
             ▼
    BGP-routed internet
```

Both paths use the same WAN connection. Both appear as standard internet traffic at the ISP level. BGP-X traffic is encrypted and appears as UDP/7474. ISP cannot distinguish between them.

---

## 9. BGP Events and BGP-X Effects

| BGP Event | BGP-X Effect | BGP-X Response |
|---|---|---|
| BGP route change to relay node IP | IP-level path between nodes changes; overlay sessions unaffected | None required |
| BGP hijack of relay node IP | Packets may be delivered to wrong host; ChaCha20-Poly1305 prevents decryption/modification | KEEPALIVE timeout → path rebuild |
| ISP blocks UDP 7474 | Connections to relay nodes fail | Pluggable transport; alternate ports; path rebuild using non-blocked nodes |
| ISP blocks relay node IP | Connections fail; KEEPALIVE timeout | Path rebuild with alternate nodes |
| BGP route instability | Latency spikes; possible KEEPALIVE timeout | Path quality monitoring → rebuild if degraded |
| Anycast routing (CDN) | Exit gateway connects to nearest CDN edge; transparent | None |
| RPKI route origin invalid | No direct BGP-X effect; potential BGP hijack risk | Recommend RPKI-valid IP for gateway operators |

---

## 10. Points of Indirect Interaction

These BGP-domain events affect BGP-X behavior without direct coupling:

### BGP Route Change

**BGP event**: A BGP route to a relay node's IP changes (different AS path, different latency).

**BGP-X effect**: The UDP packets between BGP-X nodes travel a different physical path. BGP-X sessions are unaffected. Latency may change slightly.

**BGP-X response**: None required. The overlay is agnostic to the underlying IP path.

### BGP Hijack of a Node's IP

**BGP event**: A malicious actor announces the IP prefix containing a BGP-X relay node, causing traffic to be routed to the wrong destination.

**BGP-X effect**: Packets destined for the relay node may be delivered to the hijacker's network instead.

**BGP-X protection**: ChaCha20-Poly1305 authentication tags prevent the hijacker from forging valid BGP-X packets. The hijacker cannot decrypt relay traffic. KEEPALIVE timeouts will detect the unresponsive node; path rebuild triggers.

**BGP-X recommendation**: Gateway operators use RPKI-valid IP space to reduce hijack risk.

### ISP Blocking BGP-X Node IP or Port 7474

**BGP event / ISP action**: ISP blocks UDP traffic to known BGP-X entry node IPs or drops port 7474 traffic.

**BGP-X effect**: Connections to affected relay nodes fail. KEEPALIVE timeouts detected.

**BGP-X response**: Path rebuilds using alternate nodes. Pluggable Transport (PT) obfuscates BGP-X traffic to avoid port-based or IP-based blocking.

**Pluggable Transport Architecture**:

PT operates at the transport layer (below BGP-X overlay, above physical):

```
BGP-X overlay packets → PT obfuscation → UDP (obfuscated) → ISP → BGP routing
```

The ISP sees obfuscated UDP traffic that does not match BGP-X signatures. BGP still routes it to the relay node. The relay node's PT layer deobfuscates before BGP-X processing.

**PT Security Model**: PT is a traffic analysis resistance mechanism, not a cryptographic security guarantee. Sufficiently sophisticated deep packet inspection (DPI) with machine learning classification may still identify patterns. Nation-state level traffic analysis over long time periods may succeed. PT raises the cost of blocking significantly.

**Built-in PT Limitations**: The built-in PT is effective against port-based blocking and simple protocol fingerprinting. It is NOT guaranteed effective against advanced DPI with machine learning or nation-state level traffic analysis.

**Recommendation**: For maximum obfuscation resistance in hostile network environments, use external PT implementations specifically designed for censorship circumvention (e.g., obfs4proxy with properly configured bridges). The built-in PT is a reasonable default for users who want basic traffic obfuscation without additional configuration.

### BGP Anycast

**BGP event**: Multiple servers announce the same IP prefix (anycast). BGP routes to the nearest.

**BGP-X interaction**: BGP-X exit nodes connecting to anycast destinations (DNS, CDNs) are unaffected — they connect to destination IP as normal. BGP's anycast routing is transparent to BGP-X. This is expected behavior.

---

## 11. Why BGP-X Doesn't Need BGP Participation

BGP participants (ASes, ISPs) announce IP prefixes and exchange routing information. This is necessary for infrastructure operators who own IP space and need other operators to route traffic to them.

BGP-X relay nodes receive traffic on their existing IP addresses. They don't need to announce new prefixes because they're not providing infrastructure-level routing — they're running application-layer overlay software.

A BGP-X relay node is no different from any other internet-connected server from BGP's perspective. It has an IP address, and BGP routes traffic to it the same as to any other server.

**No ASN Required**: BGP-X nodes do not need Autonomous System Numbers. An ASN is required for BGP peering, which BGP-X does not do. A BGP-X relay node operates entirely at the application layer on top of its host's existing IP connectivity.

**No Prefix Announcement**: BGP-X nodes do not announce IP prefixes to BGP. They use their existing IP address (static, DHCP, or dynamic) and rely on their ISP's BGP announcements to make that IP reachable.

**No ISP Cooperation Required**: BGP-X works without any ISP configuration changes. A BGP-X node can be deployed on any internet-connected host with a routable IP address.

---

## 12. NAT Traversal

BGP-X nodes that are behind NAT (e.g., home routers, some cloud providers) require special handling.

### 12.1 BGP-X Node Behind NAT (Relay Node)

A relay node behind NAT:

- Cannot accept inbound connections from other nodes
- Cannot advertise a reachable endpoint for the overlay
- SHOULD NOT advertise itself as a relay, entry, or exit node

BGP-X does not implement NAT traversal for relay nodes. Relay nodes MUST have a publicly reachable UDP endpoint.

### 12.2 BGP-X Client Behind NAT

BGP-X clients (not nodes) do not need a publicly reachable address. The client initiates the handshake to the entry node, and the entry node responds to the client's NAT-translated address. Standard UDP NAT traversal applies:

- The NAT translation is maintained by KEEPALIVE packets (sent every 25 seconds, randomized ±5 seconds)
- The entry node sees the client's NAT-translated IP (not the client's internal IP)

### 12.3 NAT Detection for Nodes

The node daemon attempts to determine its public IP address via:

1. Checking `public_addr` in configuration (highest priority)
2. STUN query to a well-known STUN server
3. HTTP request to an IP-reflection service

If the detected public IP differs from all local interface addresses, the node is behind NAT and will log a warning recommending the operator configure port forwarding or use a public IP host.

### 12.4 Cellular CGNAT

Cellular connections often use CGNAT. Gateway nodes on cellular should:

- Use STUN for public IP detection
- Or configure a static public IP via their cellular provider (if available)
- Or rely on relay pools rather than direct entry node operation

---

## 13. IPv4 and IPv6

BGP-X operates over both IPv4 and IPv6. The overlay protocol itself is IP-version agnostic — the transport (IPv4 or IPv6) is determined by the addresses in node advertisements.

### 13.1 Dual-Stack Operation

A BGP-X node SHOULD advertise both IPv4 and IPv6 endpoints if available. The client selects the endpoint address family based on:

- Its own network capabilities (IPv4-only, IPv6-only, or dual-stack)
- The address family of available endpoints for the selected node

### 13.2 IPv6 Path Diversity

For diversity enforcement, IPv4 and IPv6 nodes in the same ASN are treated as being in the same AS (ASN diversity check covers both address families). A path with one IPv4 node in AS12345 and one IPv6 node in AS12345 would fail the ASN diversity constraint.

This prevents trivial diversity bypass by switching address families within the same network.

### 13.3 Node Advertisement Example

```json
{
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
    { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
  ]
}
```

For domain-aware advertisements:

```json
{
  "routing_domains": [
    {
      "domain_type": "clearnet",
      "endpoints": [
        { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
        { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
      ]
    }
  ]
}
```

---

## 14. Traffic Analysis by BGP Participants

BGP participants (ISPs, IXPs, transit providers) with visibility into BGP-X traffic can observe:

- That BGP-X-using clients connect to BGP-X entry nodes (visible as UDP traffic to port 7474)
- The volume and timing of BGP-X traffic from a specific client
- That a BGP-X gateway generates UDP traffic to port 7474 (relay traffic) and TCP/UDP traffic to clearnet destinations

BGP participants CANNOT observe:

- The content of any BGP-X packet (encrypted)
- The routing within the BGP-X overlay (hidden in onion encryption)
- The relationship between a client's traffic and any specific destination
- Whether traffic is destined for a mesh island or clearnet service
- Domain transitions in cross-domain paths

This represents the BGP-X privacy boundary at the network layer. Application-layer protections (HTTPS, TLS) provide additional content privacy at the exit gateway.

---

## 15. Satellite WAN for Bridge Nodes

Bridge nodes with satellite WAN (Starlink, Iridium, etc.) participate in BGP-X exactly as fiber/cellular bridge nodes. The satellite provider uses BGP-routed internet for its backhaul — from BGP-X's perspective, this is just another clearnet transport.

**Satellite internet is clearnet**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat, OneWeb, Kuiper) provide BGP-routed IP connectivity. From BGP-X's protocol perspective, a node with Starlink WAN is a clearnet node — domain type 0x00000001. The physical medium (satellite radio vs. fiber) is invisible to the BGP-X protocol layer.

**Domain type 0x00000005** is RESERVED for a future BGP-X-native satellite network where satellites themselves run BGP-X relay software with inter-satellite links. This is NOT currently active. No commercial satellite service provides this capability.

### Configuration Example

```toml
[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ "protocol": "udp", "address": "auto-detect", "port": 7474 }]
wan_provider = "starlink"
latency_class = "satellite-leo"  # 20-40ms RTT
```

Path construction uses satellite-class latency thresholds for geo plausibility scoring. Interactive traffic paths may avoid high-latency satellite segments (configurable via routing policy).

---

## 16. ECH and DNS

When an exit gateway establishes TLS connections to clearnet destinations:

- ECH DNS queries: gateway uses DoH (encrypted DNS) to fetch HTTPS records
- DoH traffic goes to clearnet (standard BGP-routed) but is encrypted
- ECH public keys are in DNS HTTPS records (public information)
- No BGP-specific concerns

ECH configuration at exit nodes:
- `ech_capable: true` in exit policy
- DoH resolver configured (default: Quad9)
- DNSSEC validation enabled

---

## 17. RPKI Recommendations

Resource Public Key Infrastructure (RPKI) validates BGP route origin announcements. It helps prevent BGP hijacking by ensuring that only authorized ASes can announce specific IP prefixes.

BGP-X and RPKI operate at different layers and have no direct coupling:

- RPKI is a BGP-domain mechanism
- BGP-X relay operators who use RPKI-validated IP space benefit from reduced BGP hijacking risk
- BGP-X does not require RPKI and does not validate RPKI itself
- Gateway operators are **recommended** to use RPKI-valid IP space as a defense-in-depth measure

**Recommendation**: Bridge node and gateway operators SHOULD use IP space covered by valid RPKI Route Origin Authorizations. This is defense-in-depth, not a BGP-X protocol requirement.

BGP-X gateway operators are RECOMMENDED to:

- Use IP address space covered by valid RPKI Route Origin Authorizations (ROAs)
- Ensure their ASN is correctly registered with their RIR
- Operate on networks with BCP38 anti-spoofing filtering enabled

These measures protect the gateway's IP space from BGP hijacking, which could redirect BGP-X traffic through an adversary-controlled path at the IP level. While BGP-X's onion encryption would protect traffic content even in this scenario, routing integrity reduces the attack surface.

---

## 18. Multiple WAN and Failover

Gateway nodes may have multiple WAN connections:

- **Fiber + cellular backup**: Primary fiber, failover to 4G/5G
- **Dual ISP**: Load balancing and redundancy
- **Satellite + terrestrial**: Satellite as backup for remote deployments

BGP-X overlay continues during failover. Sessions survive brief WAN transitions. KEEPALIVE timing should account for satellite-class latency when satellite is active.

---

## 19. Geographic Plausibility and BGP

Geographic plausibility scoring is an OPTIONAL reputation signal in BGP-X. It is:

- NOT required for node operation
- Applied only if a node voluntarily declares a jurisdiction
- Based on RTT measurements, not external databases
- Independent of BGP routing tables or AS path information

BGP-X nodes are NOT required to declare their jurisdiction. Nodes that do declare jurisdiction enable geo plausibility scoring; nodes that do not are unaffected.

Satellite-connected nodes are exempt from geo plausibility scoring because satellite terminal IP addresses may be geographically distant from the ground station that assigned them.

---

## 20. BGP Replacement Spectrum

BGP-X enables a spectrum of BGP-independence depending on deployment mode. This is orthogonal to domain routing depth — they can be combined.

### BGP-Independence Levels

**Level 0**: Pure overlay — BGP unchanged, BGP-X adds privacy only. Users gain privacy but zero BGP independence.

**Level 1**: Intra-mesh island — zero BGP involvement for local mesh traffic. The mesh community can communicate internally without any ISP. No BGP packet is generated for intra-mesh traffic.

**Level 2**: Mesh + gateway — individual mesh users have zero direct BGP exposure; all their internet traffic routes through the gateway's ISP connection. From BGP's perspective: the gateway is one server. Everything behind the gateway (the entire mesh community) is invisible.

**Level 3**: Multi-island bridging — two or more physically separated mesh islands that cannot directly connect via radio are bridged via internet relay pools. Individual users on either island have zero BGP exposure. The internet relay nodes are standard BGP-X relay nodes; BGP sees their traffic as ordinary UDP.

**Level 4**: Dense multi-mode hybrid — all modes operating simultaneously. BGP dependence varies by user position:
- Mesh-only users: zero direct BGP exposure for all traffic
- Internet-connected users: BGP as transport only (overlay encrypts all traffic)
- Gateway users: partial — WAN connection uses BGP, mesh traffic doesn't

**Level 5**: Wide-area aerial mesh — elevated relay nodes (fixed masts, tethered aerostats on high ground) connect geographically separated mesh segments without internet involvement. Satellite links (Starlink) provide clearnet gateway connectivity without requiring terrestrial ISP. BGP dependence: minimal. Satellite uses BGP-routed internet for its backhaul, but from the mesh community's perspective, the satellite link is just another WAN for the gateway node.

**Level 6**: Maximum independence — BGP-X replaces ISP routing decisions. **Honest limit**: BGP-X cannot replace the physical infrastructure that BGP-routed networks operate on (fiber optic cables, cellular towers, spectrum allocation, submarine cables). These physical elements are BGP-independent by nature but also BGP-X independent — they're physics, not software.

### Domain Routing Depth (Orthogonal Dimension)

| Domain Depth | Description |
|---|---|
| 0 | Single domain (clearnet only) |
| 1 | Clearnet + BGP-X overlay |
| 2 | Clearnet + overlay + 1 mesh island |
| N | Any combination of N distinct routing domains |

A Level 2 BGP-independence deployment at Domain Depth 2 provides: mesh users with zero BGP exposure AND clearnet users reaching mesh services without any special hardware. These two properties reinforce each other.

### Combining BGP-Independence with Domain Routing

BGP-independence level and domain routing depth can be combined:

- **Level 2 + Depth 2**: Mesh users have zero BGP exposure; clearnet users can reach mesh services without special hardware
- **Level 3 + Depth 3**: Multi-island bridging via clearnet relays; all cross-island traffic is onion-encrypted and BGP-opaque
- **Level 5 + Depth N**: Aerial/satellite mesh with cross-domain routing; near-total BGP independence for all users

**Satellite WAN in BGP-Independence Spectrum**: A mesh island using Starlink for its gateway WAN is at BGP-independence Level 2 (mesh users have zero direct BGP exposure). The satellite link uses BGP-routed internet for its backhaul, but from the mesh community's perspective, it's just a high-latency WAN connection. The physical medium (satellite radio vs. fiber) is invisible to BGP-X routing.

---

## 21. Coverage Gap Bridging via Domain Routing

Two mesh islands that cannot directly connect via radio can be bridged via the clearnet overlay, transparently and privately:

```
Mesh Island A (Lima, Peru)           Clearnet Gap           Mesh Island B (Bogotá, Colombia)
[Pool: mesh-lima]                                           [Pool: mesh-bogota]
         │                                                          │
         └──────────────────────────────────────────────────────────┘
                            bridged via
                   [clearnet BGP-X relay pool]
                   (encrypted, opaque to BGP)

User on Island A → mesh hops → Bridge A → clearnet relays → Bridge B → mesh hops → User on Island B
```

The clearnet relay pool uses BGP as transport for those hops. BGP sees only encrypted UDP between servers. BGP cannot see the end-to-end path structure or that it is bridging two mesh communities.

**Unified DHT**: The unified DHT cross-domain synchronization (performed by bridge nodes) makes Island B's nodes discoverable from Island A's DHT without any manual configuration. There is one unified DHT spanning all routing domains. Bridge nodes with clearnet endpoints serve as the physical DHT storage infrastructure accessible from the internet. Mesh-only nodes access the unified DHT via their bridge nodes' caches.

---

## 22. Web3 and BGP-X

Web3 (blockchain protocols, DeFi, NFT platforms, cross-chain systems like OpenAccord) is an **application layer** that runs over the internet using standard protocols (HTTPS, WebSocket, libp2p). It is NOT a separate BGP-X routing domain.

From BGP-X's perspective:
- Connecting to an Ethereum RPC endpoint is clearnet (UDP/IP over BGP-routed internet)
- Connecting to a Solana validator is clearnet
- Connecting to an IPFS node via libp2p is clearnet

Web3 traffic benefits from BGP-X privacy (IP addresses hidden from RPC providers, validators, and blockchain nodes) but does not require any new BGP-X routing domain. BGP-X treats blockchain traffic identically to any other clearnet application traffic.

**Web3 on mesh islands**: A mesh island user can access Web3 services via a domain bridge node. The bridge node provides clearnet access to blockchain RPCs. The user's mesh address is hidden; only the bridge's WAN IP is visible to the RPC endpoint.

---

## 23. Honest Limits

BGP-X can make ISPs irrelevant from a surveillance and control perspective. ISP routing decisions, ISP surveillance capability, ISP-level blocking and censorship, dependency on any single ISP or jurisdiction — all can be bypassed.

BGP-X cannot replace physical infrastructure. Fiber optic cables, cellular towers, satellite hardware, radio spectrum — these exist below both BGP and BGP-X. BGP-X can make what flows through that infrastructure private; it cannot replace the infrastructure itself.

At full deployment (Level 6 BGP-independence + Domain Depth N): ISP routing decisions and surveillance capability are bypassed. Jurisdictional diversity at the relay level exceeds any single government's enforcement reach. The physical network still exists. BGP-X cannot make photons travel faster or make radio spectrum appear where there is none.

**What BGP-X CAN'T Make Private**: Physical infrastructure, radio spectrum, and the existence of traffic flows. An adversary with physical access to fiber, satellite uplinks, or radio receivers can observe that traffic exists and measure its volume. BGP-X prevents the adversary from determining who is communicating with whom and what they are saying — not from detecting that encrypted traffic is flowing.
